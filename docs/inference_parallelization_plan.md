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
- **1b. Concurrent models.** Replace the sequential `for task_key in task_keys`
  with a single `asyncio.gather` across all task keys (models are independent —
  each consumes the same full 640×640 frame).
  - Touch: `_process_batch_items` / `_infer_task_batch` in `tasks.py`.
  - Flag: `TRITON_CONCURRENT_MODELS` (default on), bounded by `max_inflight`.
  - Expected: per-batch inference latency → `max(model RTT)` instead of `sum` ≈ 3–5×.

**Exit criteria:** telemetry `mean_rtt_per_model_ms` unchanged per model, but
`p95_frame_latency_ms` drops ~3–5×; GPU util rises materially in `prod_run_benchmark.sh`.

### Phase 2 — Pipeline overlap (double/triple buffering)
*Impact: high · Effort: medium · Risk: medium*

- Decouple **Stage A** (decode + preprocess, producer threads) → bounded ring
  buffer → **Stage B** (async dispatch, consumer) → **Stage C** (postprocess /
  persist). Batch *K* infers while *K+1* preprocesses and *K−1* persists.
- Touch: restructure `_run_triton_frame_level_inference` around a
  `queue.Queue(maxsize=…)` producer/consumer (or an `asyncio` pipeline).
- Flag: `TRITON_PIPELINE_OVERLAP`.
- Expected: hides preprocess + postprocess behind inference → 1.3–2×.

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

- [ ] P1a binary tensors (`TRITON_BINARY_TENSORS`) + unit test
- [ ] P1b concurrent model `asyncio.gather` (`TRITON_CONCURRENT_MODELS`) + unit test
- [ ] P2 producer/consumer pipeline overlap (`TRITON_PIPELINE_OVERLAP`)
- [ ] P3 gRPC transport (`TRITON_PROTOCOL_PREFERENCE=grpc`)
- [ ] P4 batched DB writer (FK-safe) + throttled progress
- [ ] P4 offload pose/embeddings/render to dedicated queues
- [ ] P5 Triton `config.pbtxt` dynamic-batching tuning
- [ ] Per-phase benchmark + telemetry comparison recorded in
      `docs/production_inference_benchmark.md`
