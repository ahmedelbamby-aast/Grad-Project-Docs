# Inference Pipeline Parallelization Plan

**Last updated:** 2026-06-02

**Status:** IMPLEMENTATION COMPLETE — **ROI-320 production benchmark partial; final GPU-utilization acceptance pending.**
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
  Production probe `parallel-per-frame-signals-nms100-crop-frame-20260531T230706`
  reached `425/4541` frames with a 120-second window of `1.042 fps`, overall
  `1.000 fps`, and worker RSS near ~1.6 GiB.
- ROI-320 production benchmark completed on 2026-06-01:
  replay key `roi320-running-crop-frame-20260601T012133`, job
  `77650001-3c4b-4b0a-94aa-b4eb899b90df`, video
  `/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4`.
  The run rebuilt posture/gaze TensorRT engines for `320x320` crop behavior
  inputs and completed `4541/4541` frames with `4541` frame rows, `72751`
  detections, `72751` boxes, `57` tracks, and no stale reconciler error.
  Throughput improved from the investigation baseline `~0.975 fps` to
  `1.309 fps`; telemetry wall for main upload/inference was `3000.32 s`;
  behavior RTT improved to mean `102.393 ms` posture, `137.843 ms`
  horizontal gaze, `119.100 ms` vertical gaze, and `155.837 ms` depth gaze.
  However average GPU utilization was only `3.95%` with peak `34%`, lower than
  the certified bottleneck baseline (`13.5%` average, `38%` peak). Therefore the
  ROI phase is a measured partial success and is **not final acceptance**.
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
| 8 ROI-native crop behavior input | **Implemented and production-benchmarked; partial acceptance only** | `TRITON_CROP_BEHAVIOR_INPUT_SIZE=320` reduced tensor volume and improved throughput/RTT, but GPU utilization regressed to `3.95%` average, so final acceptance remains open |

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
- [ ] **P8 ROI-native crop behavior input (`TRITON_CROP_BEHAVIOR_INPUT_SIZE=320`) — code, deploy, and production benchmark complete; not accepted because GPU utilization regressed despite FPS/RTT/traffic wins**
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

### 2026-06-01 Controlled Optimization Execution — ROI behavior candidate

Status: **production benchmark completed; partial acceptance only**. This is not
a completed optimization because the production `combined.mp4` benchmark improved
throughput/RTT/traffic but did not improve GPU utilization.

- Code path: `TRITON_CROP_BEHAVIOR_INPUT_SIZE` lets `crop_frame` preprocess
  posture/gaze crops at a smaller square input size instead of hardcoding
  `640x640`.
- Engine/config path: `backend/scripts/build_tensorrt_engines.py
  --behavior-imgsz <size>` exports/builds posture/gaze engines with matching
  TensorRT profiles and writes Triton configs with matching input/output dims.
- Prod helper: `tools/prod/prod_enable_roi_crop_behavior.sh --input-size 320`
  rebuilds posture/gaze engines, writes configs, and updates `backend/.env`.
- Benchmark helper: `tools/prod/prod_run_parallel_flow_benchmark.sh
  --roi-behavior-input-size 320` chains the ROI build/enable step before restart
  and benchmark.
- Lifecycle fix: long-running progress updates now explicitly refresh
  `updated_at`, and final `completed` status clears stale reconciliation errors.

Production candidate result:

| Metric | Baseline | ROI-320 production result | Delta |
|---|---:|---:|---:|
| Full job throughput | `~0.975 fps` | `1.309 fps` | `+34.3%` |
| Main upload/inference wall | `~70 min` | `3000.32 s` (`50.01 min`) | `-28.6%` |
| Mean frame latency | `728.229 ms` | `394.122 ms` | `-45.9%` |
| Behavior input traffic | `380.1 MB/frame` | `~95.0 MB/frame` | `-75.0%` |
| Dense YOLO output traffic | `81.8 MB/frame` | `~20.5 MB/frame` | `-75.0%` |
| Avg GPU utilization | `13.5%` | `3.95%` | **regressed** |
| Peak GPU utilization | `38%` | `34%` | **regressed** |

Evidence: `docs/production_inference_benchmark.md` §10, replay key
`roi320-running-crop-frame-20260601T012133`, job
`77650001-3c4b-4b0a-94aa-b4eb899b90df`.

Measured outcome: ROI-native input size is worth keeping as a reversible traffic
and latency reduction, but it does not solve RTX 5090 starvation. The next
accepted optimization must remove Python/Triton request-boundary cost and request
fragmentation; a plain smaller input size is insufficient for final acceptance.

### 2026-06-01 Cycles 1–5 — Knob tuning + telemetry fix + concat memoization (ACCEPTED)

Status: **production benchmark ACCEPTED**. Overall FPS doubled, average GPU
utilization rose 91 %, and per-call RTT telemetry is finally queryable from SQL.

Bundled into one commit (single benchmark cycle because each change is small
and independently low-risk; rollback is a single revert):

1. **Telemetry writer fix** (`apps.telemetry.writer.flush_session`): added the
   missing `_bulk_insert_model_calls` so the 29 000–300 000 model-call records
   per offline job now land in `telemetry_model_calls` instead of being
   silently dropped (the JSON file already had them).
2. **`TRITON_OFFLINE_BATCH_QUEUE_MAX_CONCURRENCY` 4 → 2** — matches the
   inflight-sweep saturation knee measured on the prod RTX 5090.
3. **`TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES` 1 → 2** — lets `true_batch`
   pack two frames' crops into a single gRPC request per behavior model.
4. **`OFFLINE_TRIM_EVERY_N_BATCHES=8`** — amortizes per-batch
   `gc.collect()` + `malloc_trim(0)` cost (5–30 ms) so prod no longer pays a
   GC stall on every flush.
5. **`_build_true_batch_payload` memoization** — collapses the 4× redundant
   `b"".join` (one per fan-out behavior model) into one when the same
   `crop_payloads` list is dispatched across multiple models.

Result on `combined.mp4` (4 541 frames):

| Metric | Baseline `77650001` | Cycles 1–5 `74ec0432` | Δ |
|---|---:|---:|---:|
| Step 2 wall | 2 175 s | **883 s** | **−59.4 %** |
| **Step 2 FPS (frame inference)** | **2.09** | **5.14** | **+146 %** |
| **Overall FPS** | **1.308** | **2.644** | **+102 %** |
| **Avg GPU utilization** | **3.95 %** | **7.55 %** | **+91 %** |
| Peak GPU utilization | 34 % | 40 % | +17.6 % |
| Behavior executions / model | 4 543 | 3 598 | −20.8 % (two-frame packing) |
| Detection rows preserved | — | 72 743 / 72 751 = −0.01 % | parity |

Evidence: `docs/production_inference_benchmark.md` §11, replay key
`cycle2to5-crop-frame-20260601T012045`, job `74ec0432-995c-487e-9d77-1048ec109fb1`.
Full breakdown in `docs/crop_frame_optimization_execution.md`.

The dominant remaining cost has now shifted from frame-inference Python
orchestration to post-inference embedding (CPU-bound `sha256_stub`) and
sequential pose dispatch — both are independent of the parallelization plan's
inference hot path. Further inference-side gains would require Triton
ensemble/BLS (Phase 7d below), which is now the only remaining justified
candidate from the bottleneck investigation.

### 2026-06-01 Cycle 6 — Pose dispatch chunking fix (ACCEPTED)

Status: **production benchmark ACCEPTED**. Fixed the root cause of repeated
`inference request batch-size must be <= 16 for 'rtmpose_model'` warnings
plus per-frame HTTP fallback round-trips.

Change: `backend/apps/pipeline/services/pose_runtime.py:_provider_infer_batch`
now chunks its crops to `TRITON_MODEL_BATCH_SIZE_OVERRIDES["pose_estimation"]`
(default 16, bounded by `TRITON_MAX_INFLIGHT_REQUESTS`) before handing each
chunk to the orchestrator. Without this, the orchestrator's true-batch
packer (introduced in cycles 1–5) stacked all per-frame pose crops into a
single gRPC request — fine when `dynamic_cap` stayed ≤ 16 but the cap grew
past 16 once `avg_pose_ms_per_person` decayed below ~88 ms on the prod GPU.

Result on `combined.mp4` (4 541 frames):

| Metric | Baseline `77650001` | Cycle 1–5 `74ec0432` | **Cycle 6 `a1a448b9`** |
|---|---:|---:|---:|
| Pose-upload wall | (not separately measured) | 12 m 13 s | **3 m 42 s (−69.7 %)** |
| Total `run.complete` | 50 m 1 s | 28 m 36 s | **19 m 26 s (−32.1 % vs C1-5)** |
| **Overall FPS (DB completed)** | 1.309 | 2.077 | **2.78 (+33.8 % vs C1-5, +112 % vs baseline)** |
| **rtmpose batch-warnings** | not measured | many | **0** |
| Detection-row parity | — | 72 743 | 72 752 (+9 vs C1-5, +1 vs baseline) |

Evidence: `docs/production_inference_benchmark.md` §12, replay key
`cycle6-posechunk-crop-frame-20260601T022240`, job
`a1a448b9-474f-4dea-942b-3288bcae6900`.

The next bottleneck is now the embedding stage (~6–8 min CPU-bound
`sha256_stub` per offline job, after `run.complete`). It is independent of
the Triton parallelization plan. Inference-side, the remaining justified
candidate is still Phase 7d (Triton ensemble/BLS) — gain estimated at +20–40 %
over cycle 6, but at high engineering cost.

### 2026-06-01 Cycle 9 — Behavior ensemble (NOT ACCEPTED)

Status: **production benchmark completed, NOT accepted.** Step 2 wall
852.8 s → 858.1 s (+0.6 %, failed the ≥ 10 % reduction gate). DB-completed
FPS 3.46 → 4.09 (+18.1 %), behavior app calls −75 %, behavior RTT
143–168 ms → 107.9 ms ensemble, correctness counters unchanged.

Root cause: the four sub-models still execute identically and still return
dense YOLO outputs server-side. The savings landed in app-side request
fragmentation (which was already absorbed by `TRITON_CONCURRENT_MODELS=1`
since Cycle 1–5), not in the GPU critical path or in dense output
transfer.

Evidence: `docs/cycle_9_results.md`, replay key
`cycle9-behavior-ensemble-crop-frame-20260601T180847`, job
`c1651663-e08a-4e29-9ee3-fd0f09884b98`.

Cycle 9b Step-1 child critical-path measurement is now recorded in
`docs/cycle_9b_child_critical_path_results.md`. Direct production gRPC
decomposition identified `gaze_horizontal_model` as the dominant child
(`16.058 ms/exec` server delta versus `12.133 ms` posture, `11.759 ms`
vertical, and `11.909 ms` depth). It is also the widest dense-output model
(`[84,2100]`). The next Cycle 9b implementation should target
`gaze_horizontal_model` first, with B.2.b output fusion / narrow-head work as
the lowest-risk first candidate.

Cycle 9b B.2.b TensorRT output-slice variant is **NOT ACCEPTED**. Production
built and loaded `gaze_horizontal_gaze2_model` and `behavior_ensemble_gaze2` at
SHA `49932a22bfb429a74075e6952788af63eb007810`, but the required parity probe
failed twice. The final rebuilt-engine proof
`backend/logs/gaze_horizontal_gaze2_parity_20260601T231503_postrebuild.json`
reported `max_abs_diff=9.5` against a `1e-6` tolerance. No full benchmark was run
because the pre-benchmark parity gate failed. Production was rolled back to
`GAZE_HORIZONTAL_HEAD_VARIANT=coco80`, `MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble`,
and `LPM_ENABLED=0`.

Cycle 9b exact server-side slice is **ACCEPTED**. This candidate keeps the
legacy `gaze_horizontal_model` plan unchanged, then runs
`gaze_horizontal_slice_model` on the legacy dense output inside Triton so the app
receives `[6,2100]` instead of `[84,2100]`. The matching
`behavior_ensemble_gaze_slice` and `gaze_horizontal_slice_adapter` are guarded by
`GAZE_HORIZONTAL_HEAD_VARIANT=slice`. Production benchmark
`cycle9b-exactslice-crop-frame-20260601T233211` / job
`7933c1e5-a970-47a3-81c5-0c9bd01bd332` completed at deployed SHA
`ca69f02a8ceb214d7ef55cd2ae4b7ec75549c257`; post-benchmark parity
`backend/logs/gaze_horizontal_slice_parity_20260601T235623_postbench.json`
reported `max_abs_diff=0.0`. Step 2 wall improved `858.1 s → 573.927 s`
(`-33.1 %`), behavior RTT mean improved `107.9 ms → 91.470 ms`, DB-completed
FPS improved `4.09 → 4.307`, and correctness stayed within noise. This accepts
the exact-slice B.2.b subcandidate.

Cycle 9b exact slice plus Top-K anchor packing is **ACCEPTED WITH CAVEAT**. It
keeps `GAZE_HORIZONTAL_HEAD_VARIANT=slice`, adds FP32 TensorRT Top-K adapters
after all four behavior children, and routes `behavior_all` through
`behavior_ensemble_gaze_slice_topk` when `TRITON_BEHAVIOR_TOP_K_ENABLED=1` and
`TRITON_BEHAVIOR_TOP_K_VALUE=100`. Production benchmark
`cycle9b-topk-crop-frame-20260602T041900` / job
`be4ba9ee-4786-48e9-8334-28feb237a1fb` completed at deployed SHA
`9f879affeb4478e63a09276b10a2d64844bcbc44`; decoded parity
`backend/logs/behavior_topk_parity_20260602T011830Z_fp32.json` reported
`failed_count=0`, `max_score_diff=0.0`, and `max_box_diff=0.0`. Versus the
accepted exact-slice baseline, Step 2 frame wall improved
`573.927 s → 540.399 s` (`-5.84 %`), DB-completed FPS improved
`4.307 → 4.429`, behavior RTT mean improved `91.470 ms → 84.865 ms`, and
behavior output traffic fell from `~6.85 MB/frame` to `~0.33 MB/frame`.
Correctness stayed within tolerance (`attention_tracking` `11776 → 11781`,
`person_detection` unchanged at `19162`, tracks unchanged at `53`). Caveat:
average GPU utilization did not improve (`9.595 % → 9.3 %`) even though peak GPU
utilization improved (`45 % → 53 %`). Production remains on the Top-K route.
B.1 compact postprocessing, B.3 Step 2, and B.4 remain unaccepted; standalone
B.2.a Top-K without exact-slice was not separately benchmarked because the
accepted baseline has moved to B.2.c.

Cycle 11.A behavior input `320 → 256` is **BENCHMARK REQUIRED / UNDECIDED**.
Production built the 256 behavior engines plus matching slice/Top-K adapters
and captured candidate outputs. The synthetic pre-benchmark parity probe failed
and is recorded as a correctness warning, but no solution or optimization is
accepted, skipped, or neglected until the real production benchmark records the
throughput and correctness evidence. Evidence:
`backend/logs/parity_capture_320_20260602T123459.npz`,
`backend/logs/parity_capture_256_20260602T154826.npz`, and
`backend/logs/parity_input_size_256_20260602T154842.json`. Failures included
`posture_model` class agreement `0.695`, `gaze_vertical_model` agreement
`0.955`, and large centroid drift on posture / vertical / depth. Runtime guard
commit `4bcc79a5a4ea7c4d452b6fcd3ae3a6ff064a3bb5` keeps the useful validator
and explicit model-loading fixes. Additional benchmark tooling now exists:
`tools/prod/prod_run_behavior_input_size_matrix.sh` and
`tools/prod/prod_collect_benchmark_metrics.py`. Production was rolled back to
the accepted `TRITON_CROP_BEHAVIOR_INPUT_SIZE=320` exact-slice + Top-K profile
while the real `combined.mp4` matrix remains pending.

Five concrete continuation options recorded in `docs/cycle_9_results.md` as
Cycle 9b candidates; each STAGED until prod evidence selects which to
implement. The general lesson — **stop optimizing gRPC call count alone**
— is now an explicit discipline rule for future cycle hypotheses.

### 2026-06-01 Cycle 10 — Logical Path Matrix (STAGED)

Status: **NEEDS FURTHER ITERATION — NOT ACCEPTED.** Code, math, the C.2.1
crop-frame hook, C.2.2 `telemetry_lpm_events` storage path, unit tests, and CI
gate are deployed. Production benchmark
`cycle10-lpm-crop-frame-20260601T201239` / job
`17075418-4386-4b5f-85d4-ea23bec71f66` completed, but failed the gates in
`docs/logical_path_matrix_spec.md` §10 because LPM recorded no contradictions
(`C1=0`, `eliminated=0`) and `attention_tracking` boxes regressed
`11776 → 2680`. Production was rolled back to `LPM_ENABLED=0`.

Scope (hard-restricted): the LPM consumes per-person probabilities from
`gaze_horizontal_model`, `gaze_vertical_model`, `gaze_depth_model` only.
RTMPose head-yaw is a read-only input to C4 (physical-coupling
constraint). `person_detector` and `posture_model` (sitting/standing) are
not touched. Constraints C1–C4 enforce within-axis exclusivity
(margin-based), temporal hysteresis, impossible-state alarms, and head-pose
coupling.

Phase 1: pure-Python module at
`backend/apps/pipeline/services/logical_path_matrix.py`, flag-gated by
`LPM_ENABLED=0` default, now wired into
`backend/apps/video_analysis/tasks.py:_run_crop_behaviour_for_items` for
post-decode crop-frame gaze boxes. C.2.2 telemetry is implemented via
`TelemetrySession.record_lpm_event(...)`, `telemetry_lpm_events`, and the
JSON/PostgreSQL writer. The post-run safety fix was deployed and production
benchmarked as
`cycle10-lpm-violationonly-crop-frame-20260601T221110` / job
`21666815-f4bd-4f5f-b90e-b9101b4d899d` at SHA
`31edac44c66233baadd3a26ddd57b51b1a043d66`. It restored most of the original
attention-box loss (`2680 → 11122`) but still failed Cycle 9 parity
(`11776`) and still recorded no contradiction signal (`C1=0`,
`eliminated=0`). Production was rolled back to `LPM_ENABLED=0` again. Phase 2
(future cycle): migrate the same pure constraint solver to a Triton BLS Python
backend, which dovetails with Cycle 9b Option 1 (server-side compact
postprocessing).

### 2026-06-01 Cycle 8 — Embedding stage attack (ACCEPTED)

Status: **production benchmark ACCEPTED.** Embedding stage wall 450.7 s →
~174 s (−61.5 %); total DB-completed elapsed 26.37 min → 21.87 min
(−17.1 %); overall FPS 2.87 → 3.46 (+20.5 % vs C7, +164 % vs baseline).

Change (bundled, low-risk, one stage budget): `OFFLINE_EMBEDDING_REUSE_BY_TRACK=1`
+ process-local `track_vector_cache` + lazy `cv2.VideoCapture.read` (only
when a detection in the frame has no cached vector; forward-skip with
`.grab()`) + new `persist_embeddings_bulk` helper that replaces 72 k
single-row `.create()` calls with `bulk_create` batches of 500.

Evidence: `docs/production_inference_benchmark.md` §14, replay key
`cycle8-embed-bulk-crop-frame-20260601T125627`, job
`d2de80a0-31b7-4a47-b9f1-d2e2156ea3a8`.

At the time of Cycle 8, the 7.5-min SLA gap was 14.4 min and Step 2 was the
dominant remaining block. Later Cycle 9b work has moved the current accepted
baseline to exact slice + Top-K (`4.429 FPS`, `1022.952 s` total); use
`docs/runtime_sla_video_plus_5min.md` for the current SLA gap and stage budget.

### 2026-06-01 Cycle 7 — Redis client caching (ACCEPTED with caveat)

Status: **production benchmark ACCEPTED.** Modest +3.2 % overall FPS gain
(2.78 → 2.87 DB-completed basis; +119 % vs baseline). The hypothesis under-
projected the result by ~20× — `redis-py` 5.x lazy pool initialization made
the per-call `Redis.from_url + ping` cost ~0.08 ms instead of the ~1.5 ms I
assumed. Kept anyway because the code is cleaner, the gain is measurable
and parity is preserved. The next-cycle target shifts to the *real*
embedding sub-cost (cv2 frame seek + model.embed + per-row creates).

Evidence: `docs/production_inference_benchmark.md` §13, replay key
`cycle7-rediscache-crop-frame-20260601T120927`, job
`515fe118-6009-4776-916d-6473fbf31ed7`.

### 2026-06-01 Cycle 9 — Behavior Triton ensemble (NEEDS FURTHER ITERATION)

Status: **production benchmark completed, not accepted.** The app-level
request fragmentation target was achieved, but the Step 2 acceptance gate was
not.

Change: added a `behavior_ensemble` Triton model that accepts one
`[N, 3, 320, 320]` crop batch and fans out server-side to `posture_model`,
`gaze_horizontal_model`, `gaze_vertical_model`, and `gaze_depth_model`.
The app routes crop-frame behavior through `behavior_all` when
`TRITON_BEHAVIOR_ENSEMBLE=1`, then maps the four named outputs back to the
existing `output0` decode path. Rollback remains `TRITON_BEHAVIOR_ENSEMBLE=0`.

Production readiness finding: the pinned Triton build had been compiled with
`TRITON_ENABLE_ENSEMBLE=OFF`, so the first restart rejected the valid ensemble
config with `unexpected platform type 'ensemble'`. The production binary was
rebuilt in place with `TRITON_ENABLE_ENSEMBLE=ON`; backup binary:
`/home/bamby/services/triton_build_r2502/tritonserver/install/bin/tritonserver.pre_cycle9_no_ensemble_20260601T180729`.

Result on `combined.mp4` (4 541 frames):

| Metric | Cycle 8 `d2de80a0` | Cycle 9 `c1651663` | Δ |
|---|---:|---:|---:|
| Step 2 wall | 852.8 s | **858.1 s** | **+0.6 %** |
| DB-completed elapsed | 1 312.3 s | **1 110.7 s** | **−15.4 %** |
| DB-completed FPS | 3.46 | **4.09** | **+18.1 %** |
| App-level model calls | 20 348 | **9 557** | **−53.0 %** |
| Behavior crop calls | 14 391 | **3 597** | **−75.0 %** |
| Behavior mean RTT | 143-168 ms/model | **107.9 ms ensemble** | improved |
| Avg / peak GPU util | 9.65 % / 36 % | **9.36 % / 43 %** | avg flat, peak up |
| Row parity | baseline | **exact parity** | pass |
| Tensor parity | n/a | **max abs diff 0.0** | pass |

Evidence: `docs/production_inference_benchmark.md` §15, replay key
`cycle9-behavior-ensemble-crop-frame-20260601T180847`, job
`c1651663-e08a-4e29-9ee3-fd0f09884b98`, parity log
`backend/logs/behavior_ensemble_parity_cycle9_20260601T180827.json`.

Lesson: a plain Triton ensemble removes Python/gRPC request fragmentation but
does not reduce the server-side four-model behavior critical path or dense
YOLO output volume. The next inference-side iteration should not repeat this
shape unless it also reduces dense output movement or child-model critical
path, for example compact server-side postprocessing/BLS or a sharded
multi-process design selected by the Cycle 13 architecture decision.

### Phase 7d (optional) — Triton ensemble / BLS

**STATUS UPDATE (2026-06-01 after Cycle 9):** the simple four-behavior Triton
ensemble is implemented and production-benchmarked, but **not accepted** because
Step 2 wall stayed flat. Evidence now justifies further server-side BLS/graph
work only if it removes dense output movement or the four-child critical path,
not merely Python request count. See
[`docs/crop_frame_rtx5090_bottleneck_investigation.md`](crop_frame_rtx5090_bottleneck_investigation.md).
Production job `80027072-a9d4-4be7-9099-4354acd1170b` and two bounded profilers
showed behavior/gaze client RTT around `192-212 ms` per true-batch request while
Triton server success time was only `14.87-19.70 ms`. The current path transfers
about `380 MB/frame` of behavior input plus about `82 MB/frame` of dense YOLO
outputs through Python/gRPC for a `19.33` crops/frame sample. Cycle 9 confirmed
that a plain ensemble that still returns raw dense tensors is not sufficient for
the Step 2 gate.

**DEV/REPO:** author an `ensemble` model (`platform: "ensemble"` + `ensemble_scheduling`)
plus a Python **BLS** backend for the crop step
(`preprocess → person_detector → crop → {posture, gaze×3, rtmpose} → NMS`) under the model
repository. No retraining — same engines. The required output contract is compact
detections, not raw YOLO grids.

**CI-CD:** none (graph lives in the model repo, validated on prod).

**PROD:** deploy the ensemble dir to `triton_repository_cuda12`; reload Triton;
route the offline path to call the single ensemble model; benchmark — target one
network call per frame, compact detection outputs, and a large GPU-util jump.

**ROLLBACK:** route back to per-model calls; the ensemble model can stay loaded but unused.
