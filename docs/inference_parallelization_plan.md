# Inference Pipeline Parallelization Plan

**Last updated:** 2026-06-05

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
| Post-stages (tracking/pose/ReID/persist/render) | ⚠️ partially offloadable | optimized prod env sets `OFFLINE_OFFLOAD_POST_STAGES=1`; render still runs inline; persistence and embedding are not streamed during inference yet |
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

Cycle 9b B.4 larger batch window is **NOT ACCEPTED**. Production benchmark
`cycle9b-b4-maxframes4-20260602T175820Z` / job
`416efe8c-772c-442f-8e55-cf44c54fe261` raised
`TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES` from `2` to `4` while keeping the
accepted 320 Top-K route. The candidate completed `4541/4541` frames and
improved Step 2 frame wall `540.399 s → 512.445 s` (`-5.17 %`) plus
DB-completed FPS `4.439 → 4.471` (`+0.71 %`), with sampled worker RSS
`1120.328 MiB`. It failed the acceptance gate because behavior RTT mean
regressed `84.865 ms → 99.251 ms`, StudentTrack count dropped `53 → 47`, and
baseline-agreement F1@IoU0.5 failed for `attention_tracking` (`24.531 %`),
`hand_raising` (`26.648 %`), and `sitting_standing` (`17.217 %`). Production
was restored to `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=2`; do not repeat
`max_frames=4` unless the tracking/model-agreement failure is addressed first.

B.1 compact postprocessing and B.3 Step 2 remain unaccepted; standalone B.2.a
Top-K without exact-slice was not separately benchmarked because the accepted
baseline has moved to B.2.c.

Cycle 9b B.1 compact postprocessing investigation is **OPEN**. The investigation
document `docs/cycle_9b_compact_postproc_investigation.md` updates the original
B.1 premise for the accepted Top-K baseline: B.1 is no longer removing the old
`~17.1 MB/frame` dense behavior output payload, because B.2.c already reduced
that to `~0.33 MB/frame`. The remaining lever is server-side compact
postprocessing to remove almost all remaining output bytes and client-side
Python decode/NMS work. Pinned production Triton currently has no `python`
backend in its active backend directory, so Python BLS cannot be selected until
a controlled backend rebuild or runtime switch is benchmarked. No B.1
implementation is accepted. Phase A production probe
`backend/logs/cycle9b_b1_decode_cost_topk_20260602T185559Z.json` sampled `340`
real accepted-Top-K crops and measured Python decode/NMS at `3.125 ms` per
17-crop batch (`0.183823 ms/crop`) plus `0.114 ms` mean `as_numpy` parsing.
Across the accepted run's `3597` behavior calls, that is about `11.24 s` or
`~2.08 %` of the accepted `540.399 s` Step 2 wall. This is probe-only evidence,
not a decision. The same probe measured gRPC/Triton wait at `59.651 ms` of
`62.082 ms` RTT-with-parse, so the hypothesis for the next benchmark is that
wait/server execution matters more than Python decode. B.1 is not accepted,
rejected, skipped, deprioritized, or closed; only a full production Linux
candidate benchmark on `combined.mp4` with the required decision explanation
table can make that decision. The reproducible wrapper is
`tools/prod/prod_run_b1_decode_cost_full_benchmark.sh`. The first wrapper run
completed on production as `cycle9b-b1-fullbench-20260602T192344Z` / job
`00e0e1da-44b6-4198-ad39-39fd853e4e18` at SHA `7556e84`, but it intentionally
kept the accepted Top-K route and therefore records
`NO_DECISION_BENCHMARK_RECORDED`: Step 2 wall `540.399 s → 540.748 s`, DB FPS
`4.439 → 4.346`, behavior RTT `84.865 ms → 85.201 ms`, rows within `0.02 %`,
tracks unchanged, and model agreement `>=99.724 %`. Fresh decode evidence from
the completed job measured `19,146` crops, `45.160 ms` RTT, `42.704 ms` infer
wait, and `2.040 ms/batch` decode/NMS.
The required second repeat
`cycle9b-b1-fullbench-repeat2-20260602T195517Z` / job
`df7f832f-de81-4f92-89c7-fd213bdba7fa` also completed with wrapper status
`NO_DECISION_BENCHMARK_RECORDED`: Step 2 wall `546.702 s`, DB FPS `4.411`,
behavior RTT `84.360 ms`, rows within `0.01 %`, tracks unchanged, and
model agreement `>=99.707 %`. Its decode evidence measured `47.154 ms` RTT,
`44.162 ms` infer wait, and `4.243 ms/batch` decode/NMS over `19,146` crops.
The production watcher `tools/prod/prod_watch_benchmark_metrics.sh` now shows
these metrics in bounded wrapping tables so long evidence paths and payloads
stay inside the terminal boxes.

Cycle 12 persistent async dispatcher is therefore the next staged cycle. Its
investigation document is
`docs/cycle_12_persistent_dispatcher_investigation.md`. It targets the
single-process orchestration boundary in `backend/apps/video_analysis/tasks.py`:
the accepted Top-K baseline already reduced output bytes, and the repeated B.1
benchmarks show decode/NMS is small while infer wait plus server/orchestration
dominates. Phase A is measurement-only: add
`TRITON_ASYNC_DISPATCH_PROFILING`, record `async_runner.run(...)` boundary cost
inside `inference_audit.json`, extend the production metric collector/watcher,
and benchmark `combined.mp4` on the Linux RTX 5090 before implementing the
persistent producer/consumer dispatcher. The instrumentation and reproducible
wrapper `tools/prod/prod_run_async_dispatch_profile_benchmark.sh` are staged in
code. Clean production replay
`cycle12-async-dispatch-profile-clean-20260602T213441Z` / job
`dfa1f138-7086-418a-ba17-9999cd12b9ac` completed and is recorded in
`docs/cycle_12_persistent_dispatcher_results.md`: async-dispatch blocking wall
was `349.643 s`, and `behavior_all` owned `338.779 s`. No optimization
candidate was deployed, so no Cycle 12 acceptance/rejection decision exists.
The next implementation must overlap behavior wait/server execution; removing
only the loop-crossing bridge is probably below the `>=10 %` Step 2 gate.
The 2026-06-03 metric decision therefore selects Cycle 12.B bounded
behavior-wait overlap, documented in
`docs/cycle_12_overlap_dispatcher_investigation.md`, over a bridge-only
dispatcher. The candidate is guarded by
`TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=1` and must be benchmarked with
`tools/prod/prod_run_behavior_overlap_benchmark.sh` before any optimization
decision is valid.

Cycle 12.B production benchmark
`cycle12-behavior-overlap-20260602T223350Z` / job
`46ba8b2a-3c61-4d89-b7b6-63ec72159428` is now recorded as **NEEDS FURTHER
ITERATION**. It improved Step 2 wall `540.399 s → 395.495 s` (`-26.81 %`) and
DB FPS `4.439 → 5.195` (`+17.03 %`) with model-agreement F1 `>=99.716 %`, but
failed the RTT gate because behavior RTT mean regressed `84.865 ms →
115.420 ms` and p95 regressed `128.056 ms → 224.661 ms`. Next task: Cycle
12.C single-inflight behavior overlap.

Cycle 12.C production benchmark
`cycle12-single-inflight-overlap-20260602T225821Z` / job
`069a217f-fa43-48cc-bf18-c946d53bb3ee` is **ACCEPTED**. It improved Step 2
wall `540.399 s → 459.461 s` (`-14.98 %`), DB FPS `4.439 → 4.854`
(`+9.35 %`), and GPU avg `9.344 % → 10.332 %`, while preserving behavior RTT
mean (`84.865 ms → 83.936 ms`) and model-agreement F1 `>=99.716 %`.
`TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=1` is now part of the accepted optimized
production profile.

Cycle 13 Phase A and Cycle 13.A profiling are now measured in
`docs/cycle_13_persistence_render_investigation.md` and
`docs/cycle_13_embedding_profile_results.md`. The Cycle 12.C evidence showed
Step 3 persistence at `39.820 s`, render at `25.692 s`, and the
post-run-complete embedding/finalization tail at `189.323 s`; PostgreSQL
`FrameEmbedding.created_at` bounded the embedding write span at `187.139 s`.
The measurement-only profiling replay
`cycle13-embedding-profile-20260603T003853Z` / job
`aa2fe7a9-b3fb-49d7-92a3-eca41c894dcd` then completed on production and
measured embedding total wall `188.620 s`: track lookup `66.223 s`, Redis
flush `59.304 s`, DB flush `38.467 s`, and existing-embedding checks
`14.527 s`. Because track lookup was the largest measured bucket and code
inspection tied it to repeated per-detection related-manager queries despite
prefetched bounding boxes, Cycle 13.B implemented prefetch-aware embedding
track lookup. Production replay `cycle13-track-lookup-20260603T011324Z` / job
`c9f75d55-6043-4f27-bf9e-b2826d299459` is **ACCEPTED**: DB elapsed
`935.516 s -> 872.317 s`, DB FPS `4.854005 -> 5.205675`, embedding profile
wall `188.620 s -> 121.681 s`, and track lookup `66.223 s -> 0.447 s`, with
exact DB/model parity. The accepted optimized profile now includes
`EMBEDDING_PREFETCH_TRACK_LOOKUP=1`. The next measured post-stage bottlenecks
were Redis flush `59.874 s` and DB flush `38.773 s`; because Cycle 7
overestimated Redis savings, Cycle 13.C / 16.A first measured command count and
command wall before changing Redis semantics. Production replay
`cycle13c-redis-command-profile-20260603T020723Z`, job
`aa246a4e-e0f9-471a-9ce3-74f343bbd1fb`, is recorded in
`docs/cycle_13c_redis_db_side_effect_measurement_results.md`. It proved Redis
server command wall is tiny (`530.485 ms`) compared with the profiled Redis
flush wall (`92.397 s`) and identified client-side helper/payload/pipeline
overhead as the next Redis target. Cycle 16.B then completed in production:
replay `cycle16b-redis-coalescing-20260603T025823Z`, job
`b2dfa987-afc5-4b96-ab12-6799b149ac25`, deployed SHA `c458c443`, status
`completed`. Redis flush wall improved `59.874 s -> 35.970 s`, embedding
profile wall improved `121.681 s -> 97.505 s`, DB FPS improved
`5.205675 -> 5.347791`, DB/model parity was exact, and Redis pipeline executes
fell from the Cycle 13.C measured `72578` shape to `146`. Cycle 16.B is
therefore accepted in
`docs/cycle_16b_redis_side_effect_coalescing_results.md`.

Cycle 14.A completed as MEASUREMENT COMPLETE / HYPOTHESIS_ONLY in
`docs/cycle_14a_pose_tail_decomposition_results.md`. Production replay
`cycle14a-pose-tail-profile-20260603T135129Z`, job
`862a13db-a2ae-408f-a737-ee9aeca45f5c`, completed `4541/4541` frames on
`combined.mp4`. The run preserved DB/model parity and proved the tail remains
real: Step 2 through-pose upload was `690.164 s`, Step 2 frame wall was
`465.856 s`, and the candidate tail was `224.308 s`. The measured drain after
the frame loop was `200.778 s`; it was dominated by RTMPose runtime/provider
work (`190.176 s` runtime wall, `186.090 s` provider batch wall, `154.171 s`
provider async batch wall, and `17.327 s` provider payload build). Therefore the
next cycle is not compact behavior postprocessing, Redis Streams/scripts,
streaming persistence, or worker-count scaling. It is split into two benchmarked
sub-cycles because the evidence does not prove which RTMPose scheduling
strategy wins: Cycle 14.B1 uses `POSE_TAIL_OPTIMIZATION_MODE=overlap` for
single-inflight RTMPose overlap, and Cycle 14.B2 uses
`POSE_TAIL_OPTIMIZATION_MODE=cross_frame_batch` plus
`POSE_CROSS_FRAME_BATCH_SIZE=16` for ordered cross-frame RTMPose batching. Each
scenario must prove FPS, Step 2 wall, RTT, GPU, memory, DB parity, and model
agreement in a completed production `combined.mp4` benchmark, then the two
scenario results must be compared directly before any acceptance decision.

Cycle 14.B completed in `docs/cycle_14b_rtmpose_scenario_results.md`. B1
overlap is not accepted. B2 first run is rejected because it dropped pose
records. B2 fixed rerun is accepted: replay
`cycle14b-cross-frame-batch16-r2-20260603T150000Z`, job
`6b42a557-b954-4954-a2f8-de54634229eb`, improved DB FPS
`5.347791 -> 5.680314`, reduced pose tail `221.777 s -> 171.751 s`, reduced
RTMPose calls `5047 -> 1199`, preserved exact DB/model parity, and increased
pose records `19157 -> 19180`. The optimized production profile now includes
`POSE_TAIL_OPTIMIZATION_MODE=cross_frame_batch` and
`POSE_CROSS_FRAME_BATCH_SIZE=16`.

Cycle 14.C is staged in
`docs/cycle_14c_pose_batch_size_matrix_investigation.md` as the next sorted
cycle because the accepted cross-frame RTMPose batch cap is still a measured
tuning lever. It is split into Cycle 14.C1 (`POSE_CROSS_FRAME_BATCH_SIZE=8`)
and Cycle 14.C2 (`POSE_CROSS_FRAME_BATCH_SIZE=32`). Both scenarios must run
full production benchmarks against accepted batch `16`, then be compared
directly before any acceptance decision. The previous compact server-side
behavior postprocessing idea moves to Cycle 14.D.

Cycle 14.C completed in
`docs/cycle_14c_pose_batch_size_matrix_results.md`. Batch `8` replay
`cycle14c-pose-batch-matrix-20260603T154945Z-batch8` is not accepted because
DB FPS regressed `-2.85 %`, Step 2 through-pose wall regressed `+3.42 %`, and
provider chunks doubled. Batch `32` replay
`cycle14c-pose-batch-matrix-20260603T154945Z-batch32` is not accepted because
DB FPS regressed `-0.98 %`, Step 2 through-pose wall regressed `+1.46 %`, and
RTMPose p95 regressed `+114.18 %` despite higher GPU utilization. Batch `16`
remains the accepted production cap.

Cycle 14.C3 is staged in
`docs/cycle_14c3_batch32_parallel_chunks_investigation.md` as a narrow
batch-32 repair candidate, not a new accepted default. It keeps
`POSE_CROSS_FRAME_BATCH_SIZE=32` and tests
`POSE_PROVIDER_CHUNK_PARALLELISM=2` so the two internal 16-crop RTMPose chunks
can be dispatched concurrently. The wrapper
`tools/prod/prod_run_cycle14c3_batch32_parallel_chunks_benchmark.sh` must
compare the candidate against both the accepted batch-16 baseline and the prior
rejected batch-32 run before any decision is valid.

Cycle 14.C3 completed in
`docs/cycle_14c3_batch32_parallel_chunks_results.md` and is **NOT ACCEPTED**.
Production replay `cycle14c3-batch32-parallel2-20260603T170117Z` / job
`d71802b2-4c20-4f31-9c06-9854ebbc4eed` preserved exact correctness but made
the batch-32 performance problem worse: DB FPS regressed `-3.79 %` versus
accepted batch `16`, RTMPose p95 regressed `+87.16 %` versus prior batch `32`,
provider async wall regressed `+15.91 %` versus prior batch `32`, and GPU
average fell below both comparators. Batch `16` remains accepted, and the next
sorted cycle is Cycle 14.D.

Cycle 14.D Phase A is complete in
`docs/cycle_14d_server_side_compact_postproc_results.md`. The measurement
wrapper ran on production without changing runtime state. It found no Python
backend in the pinned Triton runtime, so Python BLS is blocked until a
controlled runtime switch/rebuild benchmark exists. It also measured
behavior decode/output cost at only `3.357 ms/batch` (`0.197497 ms/crop`) after
Top-K, so TensorRT/plugin compacting and fused output were not selected for
code. This is not an implementation rejection; it is a Phase A no-code
selection gate. The next sorted cycle is Cycle 15.

Cycle 15 is now started as Phase A in
`docs/cycle_15_cuda_shared_memory_vs_sharding_investigation.md`. It splits the
next architecture decision into 15.A CUDA shared memory and 15.B video
sharding before code. No implementation should start until measurements prove
whether copy/serialization or insufficient parallel work in flight is the
dominant lever.

Cycle 15 Phase A is measured in
`docs/cycle_15_cuda_shared_memory_vs_sharding_results.md`. CUDA shared memory
is not selected for immediate implementation because behavior serialize,
deserialize, and `as_numpy` total about `5.177 ms` per 17-crop probe batch
while infer wait remains `49.950 ms` and Step 2 through-pose is
`633.939294 s`. Video sharding is promoted to a design-proof sub-cycle only:
no sharding code should be written until deterministic stitching, duplicate
suppression, DB idempotency, terminal-state coordination, and rollback are
specified.

Cycle 15.B is now started in
`docs/cycle_15b_video_sharding_design_proof_investigation.md`. It is not an
implementation authorization. The first sub-scenario is 15.B1 two-shard
offline inference because Phase A estimated only `0.704691 %` duplicate
overlap frames. Four-shard and larger variants remain blocked until two-shard
stitching and DB idempotency are proven.

Cycle 15.B now has read-only design-proof helpers:
`tools/prod/prod_plan_video_shards.py` and
`tools/prod/prod_run_cycle15b_shard_design_probe.sh`. The sub-scenarios are
split in `docs/cycle_15b1_two_shard_design_proof_investigation.md` and
`docs/cycle_15b2_four_shard_design_proof_investigation.md`. These helpers only
prove shard interval math, pre-roll context, and authoritative frame ownership;
they do not authorize sharded inference implementation or production
acceptance.

Cycle 15.B design proof is recorded in
`docs/cycle_15b_shard_design_probe_results.md`. Production dry-run evidence at
`/home/bamby/grad_project/backend/logs/cycle15b-shard-design-20260603T191500Z`
proves exact authoritative coverage for both 15.B1 and 15.B2. 15.B1 is the
only candidate eligible for the next runtime implementation plan because it
has the lower boundary count and `0.704691 %` context overhead. No FPS,
latency, throughput, or GPU-utilization improvement can be claimed until a
future sharded runtime benchmark runs.

A pre-shard baseline benchmark for that future runtime comparison completed as
`cycle15b-pre-shard-baseline-20260603T193531Z` / job
`74561b05-105f-4ca8-aeaf-f510f4f802de` on the accepted single-job profile.
It is recorded in `docs/production_inference_benchmark.md` §39 and
`docs/cycle_15b_shard_design_probe_results.md`. This benchmark is
`NO_DECISION_BASELINE_ONLY`: runtime sharding is still blocked, but 15.B1 must
beat `5.620` DB FPS, `467.450 s` Step 2 frame wall, `641.154 s`
through-pose wall, and `83.530 ms` behavior RTT mean while preserving
DB/model parity.

Cycle 15.B1 runtime Phase A is now started in
`docs/cycle_15b1_two_shard_runtime_investigation.md`. The read-only production
readiness audit at
`/home/bamby/grad_project/backend/logs/cycle15b1-runtime-readiness-20260603T200611Z`
returned `blocked_no_runtime_candidate` with `8` critical blockers. The
production env is safe because `OFFLINE_VIDEO_SHARDING_ENABLED` is absent/off,
but a valid runtime benchmark is blocked until env/profile management,
authoritative frame-window support, parent/shard lineage, a runtime wrapper, a
parent merge helper, and detection/bbox/embedding merge idempotency exist.

Cycle 15.B1 safe-default slice is staged after that audit:
`OFFLINE_VIDEO_SHARDING_ENABLED`, `OFFLINE_VIDEO_SHARD_COUNT`, and
`OFFLINE_VIDEO_SHARD_CONTEXT_FRAMES` are now declared disabled by default;
`tools/prod/prod_set_cycle15b1_sharding_defaults.sh` sets only those keys; and
`process_video_upload` fails closed with
`cycle15b1_sharding_runtime_not_implemented` if sharding is requested before the
runtime exists. Local validation passed focused guard tests and the readiness
audit now reports `blocked_no_runtime_candidate` with `5` critical blockers.
Production deployed SHA `74631e6`, set the safe env defaults, restarted Celery
workers, and wrote readiness evidence to
`/home/bamby/grad_project/backend/logs/cycle15b1-runtime-readiness-safe-default-20260603T202308Z`.
The production audit is still `blocked_no_runtime_candidate` with `5` critical
blockers. This is safety/config work only: it does not implement, accept,
reject, skip, or close runtime sharding.

Cycle 15.B1 runtime candidate is now implemented, production-benchmarked, and
**NOT ACCEPTED**. The old runtime blockers are resolved in code:
`backend/apps/video_analysis/services/offline_sharding.py` defines the
offline-only shard planner/filter/merge contract,
`process_video_upload` accepts only valid child-shard metadata and runs the
`decode_frame_window` path, context-only frames are filtered before Step 3,
child jobs stop before render/embedding, parent orchestration lives in
`cycle15b1_sharded_ingest`, the merge helper is
`tools/prod/prod_merge_cycle15b1_shards.py`, and the production wrapper is
`tools/prod/prod_run_cycle15b1_two_shard_runtime_benchmark.sh`. Migration
`0014_cycle15b1_shard_provenance.py` adds detection/bbox/embedding provenance.
Local validation passed Python compile, shell syntax, `manage.py check`,
`makemigrations --check --dry-run`, targeted Cycle 15.B1 tests (`4 passed`),
and the local readiness helper now reports `ready_for_runtime_benchmark=True`
with `0` critical blockers. Production replay
`cycle15b1-two-shard-runtime-repeat-20260603T211319Z` / parent job
`e602a0ca-6efc-4cb0-8d30-9466fe76287b` proved the performance case against
`cycle15b-pre-shard-baseline-20260603T193531Z`: DB FPS `5.620 → 7.867`,
Step 2 critical-path frame wall `467.450 s → 233.038 s`, through-pose wall
`641.154 s → 324.763 s`, GPU average `11.846 % → 17.495 %`, and GPU peak
`57 % → 89 %`. It is still not accepted because correctness failed:
StudentTracks changed `53 → 52`, behavior RTT mean regressed
`83.530 ms → 89.718 ms`, and baseline-agreement F1@IoU0.5 was only
`59.473 %` (`attention_tracking`), `60.700 %` (`hand_raising`),
`61.387 %` (`person_detection`), and `53.455 %` (`sitting_standing`).
Evidence is under
`/home/bamby/grad_project/backend/logs/cycle15b1-two-shard-runtime-repeat-20260603T211319Z/`.
Keep `OFFLINE_VIDEO_SHARDING_ENABLED=0`; future sharding work must first fix
boundary-state identity/track stitching before any 15.B2 four-shard benchmark.

Cycle 15.B1.C is now started as a `PROBE_ONLY` boundary identity-stitching
subcycle. The read-only helper
`tools/prod/prod_analyze_cycle15b1_stitching.py` was run on production against
baseline job `74561b05-105f-4ca8-aeaf-f510f4f802de` and sharded parent
`e602a0ca-6efc-4cb0-8d30-9466fe76287b`; evidence lives under
`/home/bamby/grad_project/backend/logs/cycle15b1c-stitching-probe-20260603T220206Z/`.
The probe proves geometry is preserved while shard-1 identity labels are not:
shard-1 geometry F1 is `100.000 %` for attention, hand, person, and
sitting/standing, but track-sensitive F1 is `4.043 %`, `2.974 %`, `21.308 %`,
and `4.124 %` respectively. The next implementation must choose a governed
identity-stitching candidate (context sweep, ReID-assisted canonicalizer, or
tracker-state handoff) and then run the full production `combined.mp4`
benchmark before any decision.

Cycle 15.B1.C deep investigation is recorded under
`/home/bamby/grad_project/backend/logs/cycle15b1c-deep-stitching-20260603T221605Z/`
using helper commit `3baa4cdc`. The oracle relabeling upper bound improves
shard-1 track F1 but does not restore parity by itself: attention
`4.043 % -> 75.458 %`, hand `2.974 % -> 72.531 %`, person
`21.308 % -> 56.733 %`, and sitting/standing `4.124 % -> 84.021 %`.
Source-parent majority match is only `3.030 %` - `28.125 %` on shard 1, so
the current IoU-only 32-frame boundary map is not an acceptance path. Do not
move to 15.B2 or a non-sharding cycle yet: the next executable subcycle is
15.B1.C1, a full production context-window benchmark at `256` frames via
`tools/prod/prod_run_cycle15b1_two_shard_runtime_benchmark.sh --context-frames 256`.
If that benchmark fails model agreement, the next code subcycle is a guarded
15.B1.C2 canonicalizer with ambiguity gates rather than a simple whole-track
relabel.

Cycle 15.B1.C1 context `256` was then production-benchmarked as
`cycle15b1c1-context256-20260603T222123Z`, parent job
`401498f1-d5e4-4b95-8a46-ad3fcbbc2c25`. It completed `4541/4541` frames and
proved the throughput path: DB FPS `5.620 -> 7.905`, Step 2 frame wall
`467.450 s -> 242.392 s`, and GPU avg `11.846 % -> 17.636 %`. It is **NOT
ACCEPTED** because correctness still fails: `StudentTracks` stayed `52` vs the
baseline `53`, model-agreement F1@IoU0.5 was attention `58.997 %`, hand
`61.109 %`, person `61.767 %`, and sitting/standing `53.730 %`, and behavior
RTT regressed `83.530 ms -> 89.717 ms`. The next current-cycle implementation
is therefore 15.B1.C2, a guarded majority-vote track canonicalizer behind
`OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=majority_vote`. Do not start 15.B2 until
15.B1.C2 has either passed or failed a full production benchmark.

Cycle 15.B1.C2 majority-vote was production-benchmarked as
`cycle15b1c2-majority-vote-20260603T223932Z`, parent job
`78388c2c-d7f5-42b7-afa4-321216d23b11`, at deployed SHA `ed3d766`. It
completed `4541/4541` frames and retained the sharding performance gain
(DB FPS `5.620 -> 7.833`, Step 2 frame wall `467.450 s -> 243.909 s`, GPU avg
`11.846 % -> 17.891 %`), but is **NOT ACCEPTED**. Shard 1 mapped only `10/36`
tracks to existing parent IDs, `26/36` fell back to offset IDs, `StudentTracks`
rose `53 -> 64`, model-agreement F1 remained `53.730 %` - `61.109 %`, and
behavior RTT regressed `83.530 ms -> 90.372 ms`. Cycle 15.B2 remains blocked.
Cycle 17 Redis Streams later completed and was accepted as observability-only;
further sharding requires a new identity-state design proof before more
production runtime benchmarks.

Broader Redis strategies are appended after the current Cycle 13/14/15 sequence
unless production measurement promotes a specific Redis candidate. Cycle 13.C
did promote Cycle 16.B side-effect coalescing because Redis command count,
pipeline execute count, payload bytes, memory, errors, and server command wall
were measured on production. Redis Streams, Redis scripts, and sharding-state
cache remain later-cycle ideas in
`docs/redis_broader_optimization_opportunities.md`.

Cycle 17 Phase B is accepted as observability-only in
`docs/cycle_17_redis_streams_progress_sampling_investigation.md`. Production
replay `cycle17-redis-streams-20260604T025328Z` completed with exact DB/model
parity, bounded Redis Stream evidence, and rollback verified. DB FPS was neutral
(`5.620 -> 5.611`), so Cycle 17 must not be described as an inference-wall or
throughput optimization.

Four-agent coordination for the current cycle queue is recorded in
`docs/four_agent_cycle_coordination_board.md`. Agent 18 has released Cycle 17,
Agent 19 owns Cycle 18 design work, and Agent 20 owns the remaining Cycle 20
readiness plus Cycle 21 concurrency/benchmark governance lanes. Shared roadmap
and benchmark files are orchestrator-owned, and only one production benchmark
may hold the lock at a time.

Current turn records are `docs/agent_18_cycle_17_turn.md`,
`docs/agent_19_cycle_18_turn.md`, and
`docs/agent_20_remaining_lanes_turn.md`; future agents must update `AGENTS.md`
when they take or release a turn.

Cycle 17's decision is now recorded in `docs/production_inference_benchmark.md`
§ 40.

Cycle 18 is now documented as Phase A contract-only in
`docs/cycle_18_redis_boundary_state_cache_investigation.md`. Runtime Redis
boundary-state caching remains blocked until a new identity-state design proof
addresses the Cycle 15 sharding correctness failures.

### 2026-06-05 Open Latency-Cycle Sort

The next optimization work is sorted by measured upside, dependency order, and
readiness. This replaces the older "Cycle 20 last" wording below for active
cycle selection; the older paragraphs remain as historical context.

| Sort | Cycle | Current state | Why this order | Expected gain if gates pass |
|---:|---|---|---|---|
| 1 | **Cycle 20 post-stage timeline, then streaming persistence/embedding overlap** | `PHASE B MEASUREMENT-ONLY IMPLEMENTATION STARTED / NO DECISION` | Cycle 18.D is complete and **NOT ACCEPTED**; sharding remains blocked by packet-schema and identity correctness. Cycle 20 is now the next non-sharding latency lane, but the first slice only records serial lifecycle timestamps and keeps `OFFLINE_STREAM_POST_STAGES=0`. | Upper bound from the accepted pre-shard baseline is roughly the embedding created span `98.578 s`; hiding all of it would move total wall from `808.038 s` to about `709.460 s` and DB FPS from `5.620` to about `6.40` (`~+13.9 %`). |
| 2 | **Cycle 21 Celery worker/thread/concurrency matrix** | `GOVERNANCE ONLY` | Extra workers are credible only after sharding or streamed post-stages create independent work. Running it before then mostly measures contention or idle workers. | Unknown until a matrix runs; no expected gain may be claimed before baseline/candidate worker topology, duplicate-worker checks, DB/Redis/GPU budgets, and rollback proof exist. |
| 3 | **Cycle 11.B / Cycle 9b B.3 child-kernel tuning at 320** | `PLANNED / LOW CEILING` | It is lower risk than lifecycle work, but the measured dominant-child gap caps the benefit. It cannot close the SLA alone. | Bounded at about `~4 %` Step 2 wall, roughly `18 s` on a `459 s` Step 2 frame wall. |
| 4 | **Cycle 9b B.1 / Cycle 14.D compact postprocessing** | `OPEN / NO IMPLEMENTATION SELECTED` | Phase A measured decode/output work as small after Top-K, and the pinned Triton runtime lacks the Python backend. | Decode/NMS-only savings are about `2 %` of Step 2; a larger gain requires a backend/runtime change that also reduces server wait or execution. |
| 5 | **Cycle 19 Redis server-side scripts** | `CONDITIONAL` | Cycle 16.B already coalesced the measured Redis side-effect bottleneck; scripts require a new measured Redis read/compute/write hotspot. | No current gain estimate; start only if new production profiling exposes a Redis script-shaped hotspot. |
| 6 | **Cycle 10 LPM redesign** | `STAGED AFTER REJECTION` | Important for fusion quality, but not a latency-first cycle. It should not preempt higher-throughput blockers unless the user explicitly changes priority to behavior correctness. | No latency gain expected; acceptance would be correctness/contradiction-signal quality, not throughput. |
| Blocked | **Cycle 15.B1 identity-fixed sharding rerun, then 15.B2 only after B1 passes** | `B1 NOT ACCEPTED / B2 BLOCKED` | Cycle 18.D fixed packet validity and then tested a real OSNet-AIN `triton_reid` descriptor, but the latest replay still failed merge readiness (`1/2`), StudentTrack parity (`53 -> 57`), model agreement, and label-invariant identity. | Same measured two-shard envelope exists, but no sharding gain is accepted. A new identity-state producer, human-labeled ground-truth evaluation, or association redesign must pass a two-shard benchmark before 15.B1/15.B2 can resume. |

**Most recent cycle decision:** Cycle 18.D OSNet-AIN Triton ReID replay
`cycle18d-osnet-reid-20260605T202019Z` is **NOT ACCEPTED**. The model build and
parity gates passed, DB FPS improved `5.619787 -> 7.584265`, and Step 2 wall
improved `467.449833 s -> 245.762854 s`, but identity gates still failed:
merge-ready packets `1/2`, shard-1 existing-parent mapping `17/36`, offset
fallbacks `19/36`, StudentTracks `53 -> 57`, minimum model-agreement
F1@IoU0.5 `53.788 %`, and minimum shard-1 global-assignment F1 `79.876 %`.

Cycle 20 is now started as a measurement-only implementation slice in
`docs/cycle_20_streaming_persistence_embedding_overlap_investigation.md` for a
future streaming persistence and embedding overlap architecture. Current code
persists `Frame`/`Detection`/`BoundingBox` rows in Step 3 after the in-memory
frame inference aggregation, and `generate_embeddings` starts after
finalization/follow-up handoff. The kickoff adds
`OFFLINE_STREAM_POST_STAGE_TIMELINE` evidence and the production wrapper
`tools/prod/prod_run_cycle20_post_stage_timeline_benchmark.sh`; it does not
enable `OFFLINE_STREAM_POST_STAGES` or any queue/worker split.

Cycle 21 is staged in
`docs/cycle_21_celery_concurrency_scaling_investigation.md` for Celery worker,
thread, queue-concurrency, and GPU-cap tuning. The current metric-based
recommendation is not to raise production worker counts yet: one
`process_video_upload` task owns the single-video frame loop, one
`generate_embeddings` task owns the embedding loop, and Cycle 16.B showed that
the remaining post-stage wall is now embedding DB flush, Redis payload
serialization, and the Step 2 through-pose tail. Extra workers become credible
only after sharding, streaming persistence/embedding, multiple independent
jobs, or a production matrix proves hidden queue parallelism exists.

Cycle 11.A behavior input `320 → 256` is **NOT ACCEPTED by real production
benchmark**. Production built the 256 behavior engines plus matching slice/Top-K
adapters and captured candidate outputs. The synthetic pre-benchmark parity
probe failed and was recorded as a correctness warning, not the final decision.
The real `combined.mp4` benchmark was then run. Synthetic parity evidence:
`backend/logs/parity_capture_320_20260602T123459.npz`,
`backend/logs/parity_capture_256_20260602T154826.npz`, and
`backend/logs/parity_input_size_256_20260602T154842.json`. Failures included
`posture_model` class agreement `0.695`, `gaze_vertical_model` agreement
`0.955`, and large centroid drift on posture / vertical / depth. Runtime guard
commit `4bcc79a5a4ea7c4d452b6fcd3ae3a6ff064a3bb5` keeps the useful validator
and explicit model-loading fixes; follow-up commit `d378cec7` trims non-320
Top-K Triton load lists so stale 320-only compatibility ensembles are not
loaded during future 256 runs. Real benchmark
`cycle11-input256-realbench-20260602T161641Z-input256` / job
`822b0da4-fbf2-4186-a5a6-dd066f2eb571` improved Step 2 frame wall
`540.399 s → 391.673 s` (`-27.52 %`) and behavior RTT mean
`84.865 ms → 51.529 ms` (`-39.28 %`), but regressed persisted signals:
detection rows `72,762 → 101,213` (`+39.10 %`) and `attention_tracking`
boxes `11,781 → 20,558` (`+74.50 %`). Average GPU utilization also fell
`9.344 % → 7.367 %`. Step 2 FPS improved `8.403 → 11.594` (`+37.97 %`),
but model-agreement accuracy proxy (`F1@IoU0.5` against the accepted 320
baseline, not human-labeled accuracy) fell to `31.195 %` for
`attention_tracking`, `38.032 %` for `hand_raising`, and `65.250 %` for
`sitting_standing`; `person_detection` stayed `100.000 %`. Evidence is in
`backend/logs/cycle11-input256-realbench-20260602T161641Z/input_256_metrics.json`
and
`backend/logs/cycle11-input256-realbench-20260602T161641Z/model_agreement_320_vs_256.json`.
Production was rolled back to the accepted `TRITON_CROP_BEHAVIOR_INPUT_SIZE=320`
exact-slice + Top-K profile.

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
dominant remaining block. Later Cycle 12.C work has moved the current accepted
baseline to single-inflight behavior overlap (`4.854 FPS`, `935.516 s` total);
use `docs/runtime_sla_video_plus_5min.md` for the current SLA gap and stage
budget.

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
