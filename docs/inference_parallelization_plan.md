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

### Phase 7 — Trade information *between* models and layers (biggest structural win)
*Impact: very high · Effort: high · Risk: medium–high (model/graph changes)*

Today every model re-receives the **whole 640×640 frame** over the wire and runs
independently — no information is shared between models or pipeline layers. The
high-value redesign is to make stages feed each other:

1. **ROI sharing (detector → behaviour models).** Run `person_detector` first,
   then feed **person crops** (e.g. 192×256) to posture/gaze/pose instead of the
   full frame. Smaller inputs = far less compute, less activation VRAM, and many
   persons batch into one call. `rtmpose` is already ROI-based; converting the
   gaze/posture heads to ROI classifiers is the model-side change that unlocks this.
2. **Triton ensemble / BLS — share tensors on the GPU.** Build a Triton
   **ensemble** (or Business-Logic-Scripting) graph: `preprocess → person_detector
   → crop → {posture, gaze×3, rtmpose}`. The frame is uploaded **once** and lives
   on the GPU; downstream models read the **same device tensors** and the detector's
   ROIs without host round-trips. This collapses 5–6 network calls per frame into
   **one** and eliminates repeated host→device copies — the cleanest "models trading
   information" architecture, and a large bandwidth + VRAM-copy saving.
3. **GPU-side preprocessing.** Move letterbox/normalize into a DALI/ensemble step so
   the CPU never builds tensors (also kills the JSON/`.tolist()` cost at the source).
4. **Temporal reuse across layers (cache, don't recompute).**
   - Tracking → embeddings: compute a ReID embedding only for **new/changed
     track_ids** (cache by `track_id` in Redis — partially done) instead of every
     person every frame.
   - Detection cadence + box reuse (already implemented) extends to **behaviour
     reuse**: for a static track, reuse last posture/gaze label for a TTL window.
   - Pass tracking IDs into the renderer/telemetry so per-student timelines are
     assembled without re-scanning detections.

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

- [ ] P1a binary tensors (`TRITON_BINARY_TENSORS`) + unit test
- [x] **P1b concurrent model `asyncio.gather` (`TRITON_CONCURRENT_MODELS`) + unit test — done (flag default off, awaiting prod validation)**
- [ ] P2 producer/consumer pipeline overlap (`TRITON_PIPELINE_OVERLAP`)
- [ ] P3 gRPC transport (`TRITON_PROTOCOL_PREFERENCE=grpc`)
- [ ] P4 batched DB writer (FK-safe) + throttled progress
- [ ] P4 offload pose/embeddings/render to dedicated queues
- [ ] P5 Triton `config.pbtxt` dynamic-batching tuning
- [ ] P6 VRAM discipline (instance_group/batch caps, telemetry peak_gpu_memory gate)
- [ ] P7 ROI sharing + Triton ensemble (shared on-GPU tensors) + temporal behaviour reuse
- [ ] Per-phase benchmark + telemetry comparison recorded in
      `docs/production_inference_benchmark.md`
