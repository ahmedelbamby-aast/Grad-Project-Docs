# Inference Pipeline Parallelization Plan

**Status:** IMPLEMENTATION COMPLETE — **production benchmark certification pending.**
**Owner:** runtime/inference
**Created:** 2026-05-30
**Scope:** offline Triton inference path (`process_video_upload` →
`_run_triton_frame_level_inference`); most items also benefit the live path.
**Goal:** saturate the RTX 5090 (currently ~1% GPU util, CPU-bound) and raise
offline throughput from single-digit fps toward 100+ fps, with every phase
measured by the telemetry layer (`apps.telemetry`) and shipped behind a flag with
automatic fallback.

Related diagrams: README → *Triton Model Inference End to End lifecycle
(Sequential Order)* (current) and *Triton Model Inference End to End (Parallel)*
(target).

### Implementation completion sign-off (2026-05-31)

**Signed:** Codex implementation pass completed the repository-side plan on
2026-05-31. The remaining work is production evidence collection, not missing
repo implementation. No throughput, p95-latency, or GPU-utilization improvement
is certified until a fresh native Linux RTX 5090 run completes and updates
`docs/production_inference_benchmark.md`.

Completion scope:
- Code paths for P1a/P1b/P2/P3/P4/P5/P6/P7a/P7b are present, flag-gated, and
  covered by focused local tests.
- P7c Triton ensemble/BLS is explicitly classified as **future advanced model-repo
  work**, not a required phase for completing this plan. The optimized `crop_frame`
  mode already ships without retraining and without a Triton ensemble.
- Production wrapper gaps are closed: `prod_run_benchmark.sh` accepts
  `--pipeline-mode`, `prod-runtime-ingest-video.ps1` accepts `crop_frame`, and
  `prod_check_subjective_progress.sh` provides DB-backed progress checks.
- 2026-05-31 production-default activation pass added
  `prod_enable_parallel_flow.sh`, `prod_cancel_video_jobs.sh`,
  `prod_parallel_flow_probe.sh`, and `prod_run_parallel_flow_benchmark.sh`.
  The chained benchmark default is `crop_frame` on
  `/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4`.
- Production run started 2026-05-31 20:28 EEST:
  replay key `parallel-crop-frame-20260531T202819`, job
  `5801ef31-050f-4e20-a58e-d98122c5e920`. First probe confirmed optimized
  env/runtime flags active, Triton HTTP+gRPC ready, and model-call telemetry
  arriving. The run was cancelled at `50/4541` frames because the worker reached
  ~100 GB RSS with `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=16`; the optimized
  crop-frame default is now `1` so batches are bounded by one frame's person
  crops instead of many frames of crops/responses.
- Follow-up production run started 2026-05-31 20:36 EEST:
  replay key `parallel-crop-frame-20260531T203638`, job
  `17518cbe-c320-4880-8265-62df0add1ae3`. The memory-bounded flow is active
  with `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=1`; first observation showed worker
  RSS around 20-26 GB, but GPU samples remained near 0% and progress stalled at
  the next crop batch. The run was cancelled at `25/4541`; optimized crop-frame
  default now uses `OFFLINE_DETECT_EVERY_N_FRAMES=5` so the pipeline still emits
  per-frame outputs. The accepted accuracy contract permits reused person boxes
  on non-detection frames, but disables behaviour/gaze and embedding reuse so
  current-frame crop signals are recomputed for every frame.
- Accuracy-first profile update (2026-05-31): `prod_enable_parallel_flow.sh`
  defaults to `--profile per-frame-signals`. It keeps `TRITON_OFFLINE_FRAME_STRIDE=1`,
  `OFFLINE_DETECT_EVERY_N_FRAMES=5`, and
  `OFFLINE_REUSE_LAST_BOXES_TTL_FRAMES=10`; it sets
  `OFFLINE_BEHAVIOUR_REUSE=0`, `OFFLINE_BEHAVIOUR_REUSE_TTL_FRAMES=0`, and
  `OFFLINE_EMBEDDING_REUSE_BY_TRACK=0`, and bounds the decode/preprocess queue
  with `TRITON_OFFLINE_DECODE_QUEUE_SIZE=4`. The crop-frame loop now skips
  full-frame tensor build/serialization on non-detection frames; those frames
  reuse person boxes and run only the current-frame crop signals. It also avoids
  the generic `FrameCropper` PNG encode path and builds crop tensors directly
  from bounded `xyxy` slices. After observing >100 GB RSS in a dense crop fanout
  frame, `TRITON_NUMPY_OUTPUTS=1` keeps gRPC YOLO outputs as NumPy arrays until
  decode instead of materializing large Python lists. The next crop-fanout fix
  adds `TRITON_TRUE_BATCH_REQUESTS=1`: same-model crop tensors are stacked into
  one Triton request up to model caps from `TRITON_MODEL_BATCH_SIZE_OVERRIDES`,
  then split back into per-crop responses. Follow-up no-copy refinement keeps
  Triton NumPy outputs and split per-crop outputs as views instead of allocating
  contiguous copies for every crop. When RSS still held freed NumPy/gRPC buffers
  after dense batches, `OFFLINE_TRIM_PROCESS_MEMORY=1` was added to run Python GC
  plus Linux `malloc_trim(0)` after each offline frame batch. `--profile
  max-throughput` is retained for old behaviour-reuse benchmarks.
- Active production run started 2026-05-31 21:39 EEST:
  replay key `parallel-per-frame-signals-crop-frame-20260531T213945`, job
  `c1a9117e-7f59-4c53-9731-b528ab5e6cbd`. The run is using commit `a1770e5f`
  and the `per-frame-signals` profile. First probe: `25/4541` frames,
  `window_fps=0.400`, `frame_signal_contract=per_frame_signals_with_person_box_reuse`,
  GPU utilization sample `62%`, and `contract_status=pending_persistence`
  because DB `Frame`/`Detection` rows are written in Step 3 after frame inference.
- Replacement production run started 2026-05-31 21:47 EEST after
  `TRITON_NUMPY_OUTPUTS=1`: replay key
  `parallel-per-frame-signals-crop-frame-20260531T214740`, job
  `e8071c5a-8fb7-4c23-882d-621f8469c097`. First probe: `25/4541`,
  `window_fps=0.400`, `TRITON_NUMPY_OUTPUTS=True`, and active worker RSS near
  12 GiB instead of the previous >100 GiB spike. The run was cancelled at
  `100/4541` when RSS climbed to ~49 GiB and the crop fanout remained CPU-bound;
  follow-up implementation batches same-model crop requests and scopes gRPC
  client reuse to each dispatch window.
- Trim-enabled production run started 2026-05-31 22:33 EEST:
  replay key `parallel-per-frame-signals-trim-crop-frame-20260531T223309`, job
  `a6a06fd6-97e8-4a82-b1ef-2b9fe132d667`. It preserved the per-frame-signal
  contract and one-active-job guard, but was cancelled at `75/4541` frames after
  worker RSS climbed to ~37 GiB and throughput stayed near `0.4 fps`. Follow-up
  implementation keeps one async/gRPC event loop for the whole offline job
  instead of creating/closing gRPC channels every frame, and replaces crop-frame's
  history-wide tracking recomputation with incremental tracking plus tracked
  person-box cache refresh for sparse-detection reuse.
- Follow-up CPU hot-path optimization adds vectorized YOLO output decode and
  `TRITON_YOLO_MAX_DECODE_CANDIDATES=100` before NMS. This preserves the
  highest-confidence candidate set while preventing low-confidence raw anchors
  from making crop-frame NMS dominate one CPU core on dense classroom frames.
- Operational safeguards from the failed subjective run are codified:
  `prod_start_triton.sh` raises `nofile`, reads `TRITON_NOFILE_LIMIT` /
  `TRITON_LOG_MAX_MIB` from `backend/.env`, and truncates oversized Triton logs
  before launch.
- CI now has a dedicated focused gate:
  `.github/workflows/inference-parallelization.yml`.

### Implementation status & validation boundary (2026-05-31)

Per the constitution's evidence/reproducibility pillars, nothing is marked "done"
that has not been validated, and hardware/model-dependent work is **not** claimed
complete from a non-GPU environment.

| Phase | State | Notes |
|---|---|---|
| 1b concurrent models | **Implemented** (flag `TRITON_CONCURRENT_MODELS`; optimized prod default `1`) | equivalence unit-tested; dispatch now runs on a job-scoped async/gRPC loop to avoid per-frame channel churn; speedup/VRAM to be confirmed on prod GPU |
| 4 DB progress throttle | **Implemented** (`OFFLINE_PROGRESS_UPDATE_EVERY_N`; optimized prod default `25`) | first/last frame still persisted |
| 5 dynamic batching | **Already present** in `triton_repository_cuda12/*/config.pbtxt` (`dynamic_batching` + `instance_group`) | tune `max_batch_size`/`preferred_batch_size` on prod |
| 5a true client-side batch requests | **Implemented** (`TRITON_TRUE_BATCH_REQUESTS`; optimized prod default `1`) | same-model crop tensors stack into one Triton request, capped by `TRITON_MODEL_BATCH_SIZE_OVERRIDES`, then split back to per-crop responses |
| 5b CPU decode/NMS bound | **Implemented** (`TRITON_YOLO_MAX_DECODE_CANDIDATES`; optimized prod default `100`) | vectorized YOLO coordinate decode and top-confidence candidate cap before NMS to protect dense crop fanout |
| 1a binary tensors | **Implemented** (flag `TRITON_BINARY_TENSORS`; optimized prod default `1`) | encoder unit-tested; client auto-falls back to JSON on a binary error |
| 2 pipeline overlap | **Implemented** (flag `TRITON_OFFLINE_PIPELINE_OVERLAP`; optimized prod default `1`) | producer thread does decode+preprocess; main thread dispatches; equivalence-tested |
| 3 gRPC transport | **Implemented and bug-fixed (optimized prod default `grpc`, HTTP fallback retained; prod validation pending)** | request timeout is sent as int microseconds and client deadline as seconds |
| 4 batched detection writer / offload | **Implemented (optimized prod default on; partial offload)** | `OFFLINE_DB_BATCH_WRITES` bulk-creates frame detections/boxes per frame with per-row fallback; `OFFLINE_OFFLOAD_POST_STAGES` forces existing follow-up stages off inline execution. Render remains inline until a render-worker contract is added |
| 6 VRAM discipline | **Implemented (config + prod launcher safeguards; prod validation pending)** | per-model in-flight split (1b), tracked `config.pbtxt` single-instance guard, Triton `nofile`/log guard, telemetry `peak_gpu_memory_mb` remains prod gate |
| 7 crop-frame mode first-class | **Implemented (prod CLI/benchmark default now `crop_frame`; DB/API compatibility default unchanged)** | `crop_frame` is first-class in API/CLI/frontend and reuses the Triton frame batch/concurrent dispatch after `person_detection` crops; no retraining. Triton ensemble remains optional |

The remaining work is validation sequencing on the production GPU, not additional
application implementation.

---

## 1. Current state (what is and isn't parallel)

| Stage | Concurrent today? | Evidence |
|---|---|---|
| Frame **decode** | ✅ yes | `FrameReadPipeline` background thread |
| **Frames/crops within one model** | ✅ yes | `InferenceOrchestrator.run_inference_batch` uses true batched Triton requests when `TRITON_TRUE_BATCH_REQUESTS=1`, otherwise bounded async per-item dispatch |
| **Across the 5–6 models** | ✅ enabled in optimized prod env | `TRITON_CONCURRENT_MODELS=1` dispatches task groups through shared `asyncio.gather`; HTTP fallback remains available |
| **Across frame-batches** | ✅ opt-in overlap | `TRITON_OFFLINE_PIPELINE_OVERLAP=1` runs decode/preprocess for batch K+1 while batch K dispatches/postprocesses; final persistence remains ordered |
| Transport | ✅ gRPC in optimized prod env / HTTP fallback | `TRITON_PROTOCOL_PREFERENCE=grpc` + `TRITON_GRPC_ENABLED=1`; HTTP fallback retained |
| Tensor encoding | ✅ binary opt-in | `TRITON_BINARY_TENSORS=1` sends raw float bytes; JSON materialization remains fallback |
| Post-stages (tracking/pose/ReID/persist/render) | ⚠️ partially offloadable | optimized prod env sets `OFFLINE_OFFLOAD_POST_STAGES=1`; render still runs inline |
| DB writes | ✅ bulk/progress-throttled in optimized prod env | optimized prod env sets `OFFLINE_PROGRESS_UPDATE_EVERY_N=25`, `OFFLINE_DB_BATCH_WRITES=1`, `OFFLINE_DB_BATCH_SIZE=1000` |

**Diagnosis:** the GPU is idle because each frame is dominated by **CPU JSON
encoding of ~6 MB of float text per model** and by **serially awaiting 5–6
models**. Fixing those two things is the bulk of the win.

---

## 2. Phases

Each phase is independently shippable, flag-gated, reversible, and A/B-measurable.

### Phase 1 — Binary tensors + concurrent model dispatch  ⟵ START HERE
*Impact: very high · Effort: low–medium · Risk: low*

- **1a. Binary tensors — ✅ IMPLEMENTED (2026-05-31; optimized prod default ON).** Replace `.tolist()`/JSON with raw `float32` bytes
  (`ndarray.tobytes()`) using Triton's binary tensor extension (HTTP
  `application/octet-stream`) or gRPC tensors.
  - Touch: `TritonClient.infer` request builder; the `frame_data` construction in
    `_ingest_decoded_frame` (stop materializing a Python list).
  - Flag: `TRITON_BINARY_TENSORS` (optimized prod default on, fallback to JSON on encode error).
  - Expected: 3–8× lower per-call CPU; removes the dominant bottleneck.
- **1b. Concurrent models — ✅ IMPLEMENTED (2026-05-31, flag default OFF).**
  `_process_batch_items` now builds per-task inputs then dispatches every model in
  a **single event loop** via `asyncio.gather`, mapping results back per task
  (person_detection keeps its detect-cadence index map). Falls back to the
  sequential loop on any error.
  - 2026-05-31 hardening: the offline Triton path now owns one job-scoped async
    loop for all gRPC calls, so the crop-frame hot loop reuses one transport
    channel instead of creating/closing gRPC channels every frame.
  - Flag: `TRITON_CONCURRENT_MODELS` (default **off** until validated on the prod GPU).
  - **VRAM guard:** the global `TRITON_MAX_INFLIGHT_REQUESTS` budget is split across
    active models (`per_model_conc = max_inflight // n_models`), so client-side
    in-flight requests — and thus peak GPU activation memory — do **not** scale with
    model count.
  - Verified: `tests/unit/video_analysis/test_concurrent_model_dispatch.py` asserts
    the concurrent path yields identical detections to sequential.
  - Expected: per-batch inference latency → `max(model RTT)` instead of `sum` ≈ 3–5×.
  - **Enable on prod:** `echo 'TRITON_CONCURRENT_MODELS=1' >> backend/.env` then restart workers + benchmark.

**Exit criteria:** telemetry `mean_rtt_per_model_ms` unchanged per model, but
`p95_frame_latency_ms` drops ~3–5×; GPU util rises materially in `prod_run_benchmark.sh`.

### Phase 2 — Pipeline overlap (double/triple buffering)  — ✅ IMPLEMENTED (2026-05-31, flag default OFF)
*Impact: high · Effort: medium · Risk: medium*

- `_run_triton_frame_level_inference` now splits the hot loop into
  `_prepare_decoded_frame` (pure CPU/IO: decode-feed + letterbox + tensor build,
  **no shared state**) run on a background producer thread, and
  `_consume_prepared_frame` (enqueue + dispatch + postprocess + callbacks, **owns
  all mutation**) on the main thread, joined by a bounded `queue.Queue`. Batch *K*
  inference overlaps batch *K+1* preprocess. Decode is already threaded
  (`FrameReadPipeline`), so with overlap on there are three stages: decode →
  preprocess → dispatch.
- Flag: `TRITON_OFFLINE_PIPELINE_OVERLAP` (optimized prod default on). Falls back to the inline
  prepare→consume path (identical code, no thread) when disabled.
- Verified: `test_concurrent_model_dispatch.py` asserts overlap == sequential and
  overlap+concurrent == sequential.
- Expected: hides preprocess + postprocess behind inference → 1.3–2×.
- **Enable on prod:** `echo 'TRITON_OFFLINE_PIPELINE_OVERLAP=1' >> backend/.env`.

### Phase 3 — gRPC transport — ✅ IMPLEMENTED (2026-05-31, default OFF)
*Impact: medium–high · Effort: medium · Risk: medium*

- Added a gRPC client (`tritonclient.grpc.aio`) selected by
  `TRITON_PROTOCOL_PREFERENCE=grpc` / `TRITON_GRPC_ENABLED` (already in `.env`/base).
  Binary protobuf + HTTP/2 multiplexing.
- Touch: `TritonClient` (transport abstraction). HTTP remains the automatic fallback
  unless HTTP is explicitly disabled.
- 2026-05-31 hardening: the gRPC request timeout bug observed on production
  (`'float' object cannot be interpreted as an integer`) is fixed by passing the
  Triton request timeout as integer microseconds and the client RPC deadline as
  seconds. Unit coverage: `backend/tests/unit/pipeline/test_triton_grpc_transport.py`.
- Expected: 2–3× lower per-call latency for these small tensors; compounds with 1a.

### Phase 4 — Batched DB writes + offloaded post-stages — ✅ IMPLEMENTED (default OFF; render offload deferred)
*Impact: medium · Effort: medium · Risk: medium*

- **Batched writer** that respects the `Detection → BoundingBox → StudentTrack`
  FK order (the correct version of the removed `db_buffer` scaffold): accumulate
  rows, `bulk_create` in per-frame batches, and **throttle** `processed_frames`/progress
  updates to every N frames instead of every frame.
- **Offload** existing follow-up stages to Celery by disabling benchmark-style inline
  execution with `OFFLINE_OFFLOAD_POST_STAGES`; render still runs inline because no
  governed render-worker contract exists yet.
- Flags: `OFFLINE_DB_BATCH_WRITES`, `OFFLINE_OFFLOAD_POST_STAGES`.
- Expected: removes inter-batch stalls; cuts DB round-trips from ~18k to hundreds.

### Phase 5 — GPU-side saturation
*Impact: medium · Effort: low–medium · Risk: low*

- Verify/tune **Triton dynamic batching** in each `config.pbtxt`
  (`max_batch_size`, `preferred_batch_size: [8,16,32]`,
  `max_queue_delay_microseconds`) so concurrent requests coalesce into large GPU
  batches (engines already built for profile `[1,8,32]`).
- Optional/advanced: move letterbox+normalize into a **DALI/ensemble**
  preprocessing model or use **CUDA shared memory** to skip host→device copies.

**Implementation status:** dynamic batching and single-instance discipline are
present in the tracked CUDA 12 model configs. Production tuning remains evidence
work because it depends on live RTX 5090 memory/latency behavior.

---

### Phase 6 — VRAM discipline (keep memory flat as concurrency rises) — ✅ IMPLEMENTED (prod validation pending)
*Impact: enables the above safely · Effort: low–medium · Risk: low*

Parallelism must not balloon VRAM. Key facts and levers:

- **Concurrency ≠ more engines.** Triton loads each engine **once**; concurrent
  client requests queue at Triton and are executed by its scheduler. Peak
  activation VRAM is bounded by **Triton's batch size × instance count**, not by
  how many requests the client has in flight.
- **Cap client in-flight globally.** `TRITON_MAX_INFLIGHT_REQUESTS` split across
  models (Phase 1b) keeps total in-flight ≈ one budget regardless of model count.
- **Single instance per model.** Keep `instance_group { count: 1 }` in each
  `config.pbtxt` unless the GPU has clear headroom; each extra instance duplicates
  weights + activation scratch.
- **Bounded dynamic batch.** `max_batch_size` + `preferred_batch_size` set the
  activation ceiling; pick the largest batch that fits, not "unlimited".
- **Reuse the CUDA/pinned pools** already configured in `prod_start_triton.sh`
  (`--cuda-memory-pool-byte-size`, `--pinned-memory-pool-byte-size`) so allocations
  are recycled instead of growing.
- **Raise Triton process file-descriptor ceiling.** `prod_start_triton.sh` reads
  `TRITON_NOFILE_LIMIT` from `backend/.env` (default `65535`) and refuses to
  inherit the low 1024 soft limit that caused `accept(): Too many open files`
  during the subjective run.
- **Bound Triton log growth.** `prod_start_triton.sh` reads `TRITON_LOG_MAX_MIB`
  (default `1024`) and truncates oversized `backend/logs/triton.log` to the last
  5000 lines before launching. `prod_disk_cleanup.sh --delete` remains the manual
  cleanup tool for running systems.
- **Never run the hybrid/local path in prod.** It loads ~5 GB of local TensorRT
  engines on top of Triton's ~21 GB; `triton_only` avoids that entirely.
- **FP16 today; INT8 later.** Engines are FP16. INT8 (with calibration) would cut
  weight + activation VRAM further and raise throughput — a follow-up, not a blocker.
- **Watch it:** the telemetry session records `peak_gpu_memory_mb`; the benchmark's
  GPU CSV tracks `memory.used`. Gate any concurrency increase on these staying flat.

### Phase 7 — Crop-frame pipeline as a first-class, optimised mode (NO retraining)
*Impact: very high · Effort: medium · Risk: low–medium (uses existing crops, not new models)*

**Key correction (2026-05-31):** the system already supports two pipeline modes —
`VideoAnalysisJob.pipeline_mode` is `legacy_crop` (backward-compatible DB/API
default), `full_frame`, or `crop_frame` (production CLI/benchmark default)
([models.py:26](backend/apps/video_analysis/models.py), routed by
`_resolve_upload_pipeline_executor`). Cropping is already implemented
(`apps/pipeline/cropper.py::FrameCropper`) and the Ultralytics path already passes
`save_crop=True` / `save_frames=True` ([tasks.py:2472](backend/apps/video_analysis/tasks.py)).
Ultralytics gives us crops natively — `save_crop=True`, `result.save_crop(save_dir)`,
and manual `result.boxes.xyxy → result.orig_img[y1:y2, x1:x2]` (Boxes exposes
`xyxy/xywh/conf/cls/id`). **So ROI sharing needs NO model retraining** — we run the
existing models on person crops instead of the full frame.

The goal of Phase 7 is therefore: make **both** pipeline modes first-class,
user-selectable, and make **all** optimisations (1a binary, 1b concurrent, 2
overlap, 4 throttle, telemetry) work identically under **both** modes.

1. **Two first-class, user-selectable modes.**
   - `full_frame`: every model sees the whole 640×640 frame (current optimised path).
   - `crop_frame`: `person_detector` runs first; each person is cropped (via
     `FrameCropper` / Ultralytics `boxes.xyxy`), and posture/gaze/pose run on the
     **small crops** (smaller input = far less compute + activation VRAM; many
     persons batch into one call). `rtmpose` is already ROI-based — same pattern.
   - The user picks the mode per job (`--pipeline-mode`); the API/UI expose it.
   - **Contract:** old, new, and future code paths must run under either mode. Wire
     the crop path through the same telemetry, batch queue, concurrent dispatch,
     pipeline overlap, and binary-tensor code so they apply to crops too.
2. **Temporal reuse across layers (cache, don't recompute) — ✅ IMPLEMENTED default OFF.**
   - Tracking → embeddings: `OFFLINE_EMBEDDING_REUSE_BY_TRACK` reuses the latest
     Redis job/track embedding instead of recomputing every detection.
   - Detection cadence + box reuse (already implemented for person boxes) extends
     to **behaviour reuse**: `OFFLINE_BEHAVIOUR_REUSE` reuses last posture/gaze boxes
     for static crop-frame tracks within `OFFLINE_BEHAVIOUR_REUSE_TTL_FRAMES`.
3. **Triton ensemble / BLS (future advanced work, not a completion blocker).** Once crop_frame is first-class,
   a Triton **ensemble** (`preprocess → detector → crop (BLS) → {posture, gaze×3,
   rtmpose}`) can keep the frame on the GPU and collapse 5–6 calls into one. This is
   the only sub-item that needs Triton-graph work; it is an optimisation on top of
   the crop mode, not a prerequisite, and still needs **no retraining**.
4. **GPU-side preprocessing (optional).** Move letterbox/normalize into a DALI step
   so the CPU never builds tensors.

These turn the pipeline from "N independent full-frame inferences" into a single
**shared-tensor graph with cached temporal state** — the durable path to GPU
saturation at low VRAM.

## 3. Expected trajectory

| After | Offline throughput (4 541-frame classroom clip) | GPU util |
|---|---|---|
| Today (sequential models + JSON) | ~0.5–5 fps | ~1% |
| Phase 1 (binary + concurrent models) | ~15–40 fps | rising |
| Phase 2 (overlap) | ~25–60 fps | — |
| Phase 3 (gRPC) | ~40–80 fps | — |
| Phase 4–5 (batched DB + GPU batching) | 100+ fps | >50% |

Numbers are estimates; the telemetry layer + `prod_run_benchmark.sh` provide the
ground truth after each phase.

---

## 4. Validation & rollout

- **Measure** before/after each phase with `tools/prod/prod_run_benchmark.sh`
  (GPU CSV + telemetry JSON) and the `telemetry_sessions` /
  `telemetry_model_calls` tables (per-model RTT, p95 frame latency, drop rate).
- **Compare** runs via `GET /api/v1/video-analysis/jobs/{id}/benchmark-comparison/`.
- **Flag-gate** every change with automatic fallback to the prior behavior; keep
  the sequential path working so a phase can be disabled instantly in `.env`.
- **Correctness:** detection/track parity vs. the sequential baseline on the same
  clip before accepting any phase.

---

## 5. Task checklist

- [x] **P1a binary tensors (`TRITON_BINARY_TENSORS`) + encoder unit test — optimized prod default on; client falls back to JSON**
- [x] **P1b concurrent model `asyncio.gather` (`TRITON_CONCURRENT_MODELS`) + unit test — optimized prod default on; awaiting prod benchmark certification**
- [x] **P2 producer/consumer pipeline overlap (`TRITON_OFFLINE_PIPELINE_OVERLAP`) + equivalence test — optimized prod default on**
- [x] **P3 gRPC transport (`TRITON_PROTOCOL_PREFERENCE=grpc`) — optimized prod default on; HTTP fallback; prod validation pending**
- [x] **P4 throttled progress writes (`OFFLINE_PROGRESS_UPDATE_EVERY_N`) — optimized prod default `25`**
- [x] **P4 batched detection writer (`OFFLINE_DB_BATCH_WRITES`) + existing follow-up offload (`OFFLINE_OFFLOAD_POST_STAGES`) — optimized prod default on; render-worker split deferred**
- [x] **P5 dynamic batching present in `triton_repository_cuda12` configs** — tune `max_batch_size`/`preferred_batch_size` on prod
- [x] **P5a true Triton request batching (`TRITON_TRUE_BATCH_REQUESTS`) — crop fanout now stacks same-model crops into model-capped batch requests before splitting responses**
- [x] **P6 VRAM discipline — tracked Triton configs use `instance_group count: 1`; telemetry peak GPU gate remains prod acceptance**
- [x] **P7a make `full_frame` + `crop_frame` both first-class & user-selectable — prod CLI/benchmark default is now `crop_frame`; DB/API compatibility default remains `legacy_crop`; prod GPU validation pending**
- [x] **P7a crop-frame tracking hot path — incremental tracking replaces full-history `_assign_tracking_ids(all_frames)` rescans; sparse person-box reuse now preserves assigned track IDs**
- [x] **P7b temporal reuse: embeddings per track + crop-frame behaviour reuse for static tracks — implemented, optional, not the default for per-frame-signal runs**
- [x] **Production helper suite added — cancel active jobs, enable optimized defaults, probe flow evidence, verify per-frame signals, and run `combined.mp4` benchmark**
- [x] **P7c decision — ensemble/BLS classified as future advanced Triton model-repo work, not required for this plan's repo-side completion**
- [x] **CI focused gate added — `.github/workflows/inference-parallelization.yml` runs transport, batching, crop-frame, docs, and prod-helper syntax checks**
- [x] **Per-phase benchmark workflow documented; fresh production telemetry comparison remains a prod-only evidence gate in `docs/production_inference_benchmark.md`**

---

## 6. Explicit execution steps — by environment (dev / CI-CD / prod)

Every phase was built with a reversible flag and equivalence/encoder test.
The production helper now enables the optimized defaults in `backend/.env`; rollback
still means flipping the relevant flag off and restarting workers.
Commands assume the dev venv (`.venv/Scripts/python.exe`) and the prod host
(`/home/bamby/grad_project`, `APP_ENV=prod`, `config.settings`).

### Phase 7a — `full_frame` + `crop_frame` both first-class (NO retraining)

**DEV (local):**
1. Read the legacy path end-to-end first: `apps/pipeline/cropper.py::FrameCropper`,
   `apps/tracking/pipeline_mode.py::normalize_pipeline_mode`,
   `_resolve_upload_pipeline_executor` and the `legacy_crop` branch in
   `apps/video_analysis/tasks.py` (around line 2293+). Confirm whether `legacy_crop`
   currently routes to Triton or the local Ultralytics `model.track(...)` path.
2. Add a crop-frame inference function that mirrors `_run_triton_frame_level_inference`
   but, after person detection, crops each person (reuse `FrameCropper` / `boxes.xyxy`)
   and dispatches the **crop tensors** to posture/gaze/pose through the **same**
   `_process_batch_items` / batch queue. Crops use the model's native input size
   (e.g. rtmpose 256×192) instead of 640×640.
3. Make **every** optimisation flag apply under both modes (they read `settings`,
   so they already will once the crop path calls the shared dispatch): verify
   `TRITON_BINARY_TENSORS`, `TRITON_CONCURRENT_MODELS`, `TRITON_OFFLINE_PIPELINE_OVERLAP`,
   `OFFLINE_PROGRESS_UPDATE_EVERY_N`, and telemetry record_frame/record_model_call
   fire on the crop path too.
4. Expose the choice: support `--pipeline-mode {legacy_crop,full_frame,crop_frame}` in
   `runtime_ingest_video`; surface `pipeline_mode` in the job-create serializer/API
   (`apps/video_analysis/views.py`, `serializers`) and the SPA upload form so the
   **user** selects it. Default stays `legacy_crop` (existing default).
5. Tests (`backend/tests/unit/video_analysis/`): equivalence per mode — run a tiny
   mock-orchestrator job in `full_frame` and `crop_frame`; assert each produces the
   expected detections and that flags on==off behaviour holds under both. Run:
   `.venv/Scripts/python.exe -m pytest tests/unit/video_analysis -q`.

**CI-CD:** no workflow change needed — new tests auto-run under `tests/unit/`. The
crop path must NOT require model artifacts in unit tests (mock the orchestrator);
any test that needs real models gets `@pytest.mark.model_artifacts` (excluded from
the gate). `manage.py check` + docs gate already wired.

**PROD:** `git pull`; pick mode per job via `--pipeline-mode`. No new `.env` flag is
required (mode lives on the job). Validate **both** modes with
`tools/prod/prod_run_benchmark.sh` (add a `--pipeline-mode` pass-through) and compare
`telemetry_sessions` p95 latency + `peak_gpu_memory_mb` between modes.

**ROLLBACK:** set the job's `pipeline_mode` back to `legacy_crop`/`full_frame`; no
code revert needed.

### Phase 7b — temporal reuse (embeddings per new track, behaviour reuse)

**DEV:**
1. In `apps/tracking/embeddings.py`, before computing an embedding, check the Redis
   `cache_job_track_embedding` key for that `track_id`; compute only for new/changed
   tracks. Add flag `OFFLINE_EMBEDDING_REUSE_BY_TRACK` (optional; off in the
   accuracy-first production profile).
2. Extend the box-reuse pattern (`_apply_person_detection_cadence_and_reuse`) to
   posture/gaze: cache last label per `track_id` and reuse within
   `OFFLINE_BEHAVIOUR_REUSE_TTL_FRAMES` for static tracks. Flag
   `OFFLINE_BEHAVIOUR_REUSE` (optional; off in the accuracy-first production profile).
3. Tests: assert reuse path returns the cached label and that a moved/new track
   recomputes; assert parity of final per-track results vs. recompute-every-frame.

**CI-CD:** none (unit tests auto-run). DB-backed reuse tests run in the gate (Postgres
present); Redis-dependent assertions use the LocMemCache test default or fakeredis.

**PROD:** Reuse is optional. For the current accuracy-first benchmark, run
`bash tools/prod/prod_enable_parallel_flow.sh --profile per-frame-signals`; this
allows sparse person detection with last-box reuse while disabling behaviour/gaze
and embedding reuse. For pure throughput experiments only, run
`bash tools/prod/prod_enable_parallel_flow.sh --profile max-throughput`, restart
workers, benchmark, and confirm detection/track parity + lower CPU.

### Phase 3 — gRPC transport

**DEV:**
1. Add a gRPC client path in `apps/pipeline/services/triton_client.py` using
   `tritonclient.grpc.aio` (add the dep to `backend/requirements.txt`), selected by
   `TRITON_PROTOCOL_PREFERENCE=grpc` / `TRITON_GRPC_ENABLED` (already in base/.env).
   Reuse `_resolve_io_spec`; binary tensors are native to gRPC.
2. Keep HTTP as the automatic fallback if gRPC connect/infer fails.
3. Tests: unit-test request construction for gRPC (mock stub); `manage.py check`.

**CI-CD:** add `tritonclient[grpc]` to the installed deps so import-time paths are
covered; gRPC is import-guarded so the gate (no Triton) still passes. No service needed.

**PROD:** `git pull`; ensure gRPC port reachable (`curl`/`grpcurl` to 39101);
`echo 'TRITON_PROTOCOL_PREFERENCE=grpc' >> backend/.env`; restart workers; benchmark.
**This is the main RTT-bound win** — confirm `mean_rtt_per_model_ms` drops sharply
and GPU util rises in the GPU CSV.

**ROLLBACK:** `TRITON_PROTOCOL_PREFERENCE=http`; restart workers.

### Phase 4 (remaining) — FK-safe batched writer + offload

**DEV:**
1. Batched writer that respects `Detection → BoundingBox → StudentTrack` ordering
   (`bulk_create` per layer, capturing PKs). Flag `OFFLINE_DB_BATCH_WRITES` (optimized prod default on).
2. Offload pose/embeddings/render to the existing queues
   (`pipeline.offline.rtmpose_model.worker`, `pipeline.offline.behavior.worker`,
   a render worker). Flag `OFFLINE_OFFLOAD_POST_STAGES` (optimized prod default on).
3. Tests (DB-backed): row counts + referential integrity parity vs. per-row writes.

**CI-CD:** none (Postgres already in the gate). Ensure the offload tasks are declared
on mode/stage queues per the constitution (live/offline separation).

**PROD:** `git pull`; enable the two flags; restart workers (the offload queues must
have running workers — they already start via `prod_start_celery_workers.sh`); benchmark.

### Phase 6 — VRAM discipline (Triton config tuning)

**DEV/REPO:** edit `backend/models/triton_repository_cuda12/*/config.pbtxt`: confirm
`instance_group { count: 1 }` unless GPU headroom is proven (person_detector is
currently `count: 2`), and raise `max_batch_size`/`preferred_batch_size` only to a
value the `.plan` engine's profile supports (engines built `[1,8,32]`).

**CI-CD:** none.

**PROD:** copy updated `config.pbtxt` to the prod model repo; **reload Triton**
(`bash -l tools/prod/prod_start_triton.sh`); wait for READY; benchmark and watch
`peak_gpu_memory_mb` (must stay within budget) and `avg_util`.

**ROLLBACK:** restore the previous `config.pbtxt` and reload Triton.

### Phase 7c (optional) — Triton ensemble / BLS

**DEV/REPO:** author an `ensemble` model (`platform: "ensemble"` + `ensemble_scheduling`)
plus a Python **BLS** backend for the crop step
(`preprocess → person_detector → crop → {posture, gaze×3, rtmpose}`) under the model
repository. No retraining — same engines.

**CI-CD:** none (graph lives in the model repo, validated on prod).

**PROD:** deploy the ensemble dir to `triton_repository_cuda12`; reload Triton;
route the offline path to call the single ensemble model; benchmark — target one
network call per frame and a large GPU-util jump.

**ROLLBACK:** route back to per-model calls; the ensemble model can stay loaded but unused.
