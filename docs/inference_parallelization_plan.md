# Inference Pipeline Parallelization Plan

**Status:** ACTIVE — **highest-priority performance plan for this project.**
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

### Implementation status & validation boundary (2026-05-31)

Per the constitution's evidence/reproducibility pillars, nothing is marked "done"
that has not been validated, and hardware/model-dependent work is **not** claimed
complete from a non-GPU environment.

| Phase | State | Notes |
|---|---|---|
| 1b concurrent models | **Implemented** (flag `TRITON_CONCURRENT_MODELS`, default off) | equivalence unit-tested; speedup/VRAM to be confirmed on prod GPU |
| 4 DB progress throttle | **Implemented** (`OFFLINE_PROGRESS_UPDATE_EVERY_N`, default 1 = unchanged) | safe, opt-in |
| 5 dynamic batching | **Already present** in `triton_repository_cuda12/*/config.pbtxt` (`dynamic_batching` + `instance_group`) | tune `max_batch_size`/`preferred_batch_size` on prod |
| 1a binary tensors | **Implemented** (flag `TRITON_BINARY_TENSORS`, default off) | encoder unit-tested; client auto-falls back to JSON on a binary error. Enable + validate on the GPU host (live Triton) before trusting throughput |
| 2 pipeline overlap | **Implemented** (flag `TRITON_OFFLINE_PIPELINE_OVERLAP`, default off) | producer thread does decode+preprocess; main thread dispatches; equivalence-tested |
| 3 gRPC transport | **Designed** | new `tritonclient.grpc` path; requires live-Triton validation |
| 4 batched detection writer / offload | **Designed** | FK-safe bulk writer + dedicated-queue offload |
| 6 VRAM discipline | **Partly enforced** | per-model in-flight split (1b) + telemetry `peak_gpu_memory_mb`; config caps to confirm |
| 7 crop-frame mode first-class | **Designed (code, NO retraining)** | crop pipeline + Ultralytics `save_crop` already exist; make both `full_frame`/`crop_frame` first-class + all optimisations work under both. Triton ensemble is an optional extra |

The remaining phases are sequenced below with concrete designs so they can be
implemented and validated on the production GPU one at a time.

---

## 1. Current state (what is and isn't parallel)

| Stage | Concurrent today? | Evidence |
|---|---|---|
| Frame **decode** | ✅ yes | `FrameReadPipeline` background thread |
| **Frames within one model** | ✅ yes | `InferenceOrchestrator.run_inference_batch` → `asyncio.gather` + `max_concurrency` semaphore over `httpx.AsyncClient` |
| **Across the 5–6 models** | ❌ sequential | `for task_key in task_keys:` in `_process_batch_items` awaits each model in turn → per-batch latency = **sum** of model RTTs |
| **Across frame-batches** | ❌ sequential | `_flush_pending` fully processes one batch (infer + postprocess + DB + WS + telemetry) before the next |
| Transport | ⚠️ HTTP/1.1 (`httpx`) | not gRPC |
| Tensor encoding | ❌ JSON list | `input_data = input_tensor.reshape(-1).tolist()` — a **1.2M-float Python list, JSON-serialized, per model, per frame** |
| Post-stages (tracking/pose/ReID/persist/render) | ❌ inline, sequential | all inside `process_video_upload`, blocking the next batch |
| DB writes | ❌ per-frame | `VideoAnalysisJob.update` every frame + per-detection `Detection`/`BoundingBox` creates |

**Diagnosis:** the GPU is idle because each frame is dominated by **CPU JSON
encoding of ~6 MB of float text per model** and by **serially awaiting 5–6
models**. Fixing those two things is the bulk of the win.

---

## 2. Phases

Each phase is independently shippable, flag-gated, reversible, and A/B-measurable.

### Phase 1 — Binary tensors + concurrent model dispatch  ⟵ START HERE
*Impact: very high · Effort: low–medium · Risk: low*

- **1a. Binary tensors.** Replace `.tolist()`/JSON with raw `float32` bytes
  (`ndarray.tobytes()`) using Triton's binary tensor extension (HTTP
  `application/octet-stream`) or gRPC tensors.
  - Touch: `TritonClient.infer` request builder; the `frame_data` construction in
    `_ingest_decoded_frame` (stop materializing a Python list).
  - Flag: `TRITON_BINARY_TENSORS` (default on, fallback to JSON on encode error).
  - Expected: 3–8× lower per-call CPU; removes the dominant bottleneck.
- **1b. Concurrent models — ✅ IMPLEMENTED (2026-05-31, flag default OFF).**
  `_process_batch_items` now builds per-task inputs then dispatches every model in
  a **single event loop** via `asyncio.gather`, mapping results back per task
  (person_detection keeps its detect-cadence index map). Falls back to the
  sequential loop on any error.
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
- Flag: `TRITON_OFFLINE_PIPELINE_OVERLAP` (default off). Falls back to the inline
  prepare→consume path (identical code, no thread) when disabled.
- Verified: `test_concurrent_model_dispatch.py` asserts overlap == sequential and
  overlap+concurrent == sequential.
- Expected: hides preprocess + postprocess behind inference → 1.3–2×.
- **Enable on prod:** `echo 'TRITON_OFFLINE_PIPELINE_OVERLAP=1' >> backend/.env`.

### Phase 3 — gRPC transport
*Impact: medium–high · Effort: medium · Risk: medium*

- Add a gRPC client (`tritonclient.grpc.aio`) selected by
  `TRITON_PROTOCOL_PREFERENCE=grpc` / `TRITON_GRPC_ENABLED` (already in `.env`/base).
  Binary protobuf + HTTP/2 multiplexing.
- Touch: `TritonClient` (transport abstraction), `_build_triton_orchestrator`.
- Expected: 2–3× lower per-call latency for these small tensors; compounds with 1a.

### Phase 4 — Batched DB writes + offloaded post-stages
*Impact: medium · Effort: medium · Risk: medium*

- **Batched writer** that respects the `Detection → BoundingBox → StudentTrack`
  FK order (the correct version of the removed `db_buffer` scaffold): accumulate
  rows, `bulk_create` in batches, and **throttle** `processed_frames`/progress
  updates to every N frames instead of every frame.
- **Offload** pose / embeddings-ReID / render to the existing dedicated queues
  (`pipeline.offline.rtmpose_model.worker`, `pipeline.offline.behavior.worker`,
  a render worker) so they run on parallel workers instead of inline.
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

---

### Phase 6 — VRAM discipline (keep memory flat as concurrency rises)
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
- **Never run the hybrid/local path in prod.** It loads ~5 GB of local TensorRT
  engines on top of Triton's ~21 GB; `triton_only` avoids that entirely.
- **FP16 today; INT8 later.** Engines are FP16. INT8 (with calibration) would cut
  weight + activation VRAM further and raise throughput — a follow-up, not a blocker.
- **Watch it:** the telemetry session records `peak_gpu_memory_mb`; the benchmark's
  GPU CSV tracks `memory.used`. Gate any concurrency increase on these staying flat.

### Phase 7 — Crop-frame pipeline as a first-class, optimised mode (NO retraining)
*Impact: very high · Effort: medium · Risk: low–medium (uses existing crops, not new models)*

**Key correction (2026-05-31):** the system already supports two pipeline modes —
`VideoAnalysisJob.pipeline_mode` is `legacy_crop` (default) or `full_frame`
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
2. **Temporal reuse across layers (cache, don't recompute).**
   - Tracking → embeddings: compute a ReID embedding only for **new/changed
     track_ids** (cache by `track_id` in Redis — partially done) instead of every
     person every frame.
   - Detection cadence + box reuse (already implemented for person boxes) extends
     to **behaviour reuse**: for a static track, reuse last posture/gaze label for a
     TTL window.
3. **Triton ensemble / BLS (optional, advanced).** Once crop_frame is first-class,
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

- [x] **P1a binary tensors (`TRITON_BINARY_TENSORS`) + encoder unit test — done (default off; client falls back to JSON; validate on GPU host)**
- [x] **P1b concurrent model `asyncio.gather` (`TRITON_CONCURRENT_MODELS`) + unit test — done (flag default off, awaiting prod validation)**
- [x] **P2 producer/consumer pipeline overlap (`TRITON_OFFLINE_PIPELINE_OVERLAP`) + equivalence test — done (default off)**
- [ ] P3 gRPC transport (`TRITON_PROTOCOL_PREFERENCE=grpc`)
- [x] **P4 throttled progress writes (`OFFLINE_PROGRESS_UPDATE_EVERY_N`) — done (default 1 = unchanged)**
- [ ] P4 batched detection writer (FK-safe) + offload pose/embeddings/render to dedicated queues
- [x] **P5 dynamic batching present in `triton_repository_cuda12` configs** — tune `max_batch_size`/`preferred_batch_size` on prod
- [ ] P6 VRAM discipline (instance_group/batch caps, telemetry peak_gpu_memory gate)
- [ ] P7a make `full_frame` + `crop_frame` both first-class & user-selectable; all optimisations (binary/concurrent/overlap/telemetry) work under BOTH modes (NO retraining — uses FrameCropper / Ultralytics save_crop)
- [ ] P7b temporal reuse: embeddings per new track_id + behaviour reuse for static tracks
- [ ] P7c (optional) Triton ensemble/BLS shared-GPU-tensor graph
- [ ] Per-phase benchmark + telemetry comparison recorded in
      `docs/production_inference_benchmark.md`
