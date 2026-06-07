# Pipeline Latency Optimization Program — Target ≤ 10 min end-to-end

**Created:** 2026-06-07 · **Branch:** `014-yoloe-scene-srvl`
**Goal:** Cut the full all-models `combined.mp4` (4541 frames) run from **~23–35 min → ≤ 10 min** (≥ 2.5×) with no accuracy regression.
**Authority:** every cycle below is a governed optimization cycle — accepted only by a completed full end-to-end native-Linux RTX 5090 benchmark **at frame stride = 1** (inference on every decoded frame; stride > 1 is profiling-only, no authority — §7.1.1 v2.12.0) with the §7.1.1 precision metrics + §12.6 figure bundle. Every run is recorded in [`docs/BENCHMARK_RESULTS_LEDGER.md`](BENCHMARK_RESULTS_LEDGER.md) with its decision + reason. Each carries a `streaming_compatibility` declaration (§8.6).
**Frozen baseline (B003):** full all-models run = **1912 s / 2.37 FPS** at stride=1; exit target **≤600 s** with correctness parity.

---

## 1. Deep investigation — the system is CPU/orchestration-bound, NOT GPU-bound

Evidence captured during a live all-models run (job `a4b08e1c`, RTX 5090):

| Signal | Measurement | Implication |
|---|---|---|
| **GPU utilization** | **2 %, 2 %, 9 %, 2 %** (36 % spike), power ~90–100 W / 575 W | GPU is **idle/starved** — not the bottleneck |
| Triton `nv_inference_queue_duration_us` | ≫ request/compute duration (queue ~1e20 µs scale) | Requests **wait in queue**, dispatched inefficiently |
| Aggregate postprocess (prior audit) | **739 ms/frame · 66 %** of compute | Dominant cost is **CPU/Python postprocess** |
| Aggregate inference | 367 ms/frame · 33 % (many sequential per-crop calls) | Serial dispatch, no overlap |
| `async_dispatch` | **disabled** (`enabled: false`) | Model calls not overlapped |
| Pose kinematics | 11.5 ms/frame on **CPU** (`device: cpu`) | Serial CPU stage |
| Per-call Triton RTT | person_detector **41 ms**, gaze 23–24, posture 22 | RTT inflated by queue wait, not compute |

**Root cause:** the pipeline decodes masks/poses/gaze/posture and runs NMS + the scene normalizer in **Python on the CPU**, and dispatches Triton requests **one crop at a time, synchronously**. The GPU finishes each tiny request almost instantly then sits idle waiting for the CPU to prepare/serialize the next. Throughput is gated by single-threaded CPU postprocess + dispatch latency, not by model compute.

**Consequence for strategy:** do **not** chase faster/quantized models (GPU is already idle). Win by (a) **feeding the GPU in batches with overlap** and (b) **removing the CPU postprocess bottleneck**. There is ≥ 10× idle GPU headroom to absorb the work.

---

## 2. Target budget (how ≤ 10 min is reached)

4541 frames in 600 s ⇒ **≥ 7.6 FPS sustained** end-to-end (current all-models ≈ 2.2–3.3 FPS). Allocation:

| Stage | Now (≈ms/frame) | Target | Lever |
|---|---|---|---|
| Postprocess (CPU) | 739 | ≤ 200 | GPU/vectorized decode + NMS, drop per-crop Python loops |
| Inference dispatch | 367 | ≤ 120 | batch crops, async/overlapped dispatch, raise dynamic batching |
| Preprocess | 19 | ≤ 15 | vectorized letterbox, pinned buffers |
| Pose kinematics | 11.5 (CPU) | ≤ 5 | GPU/vectorized + overlap off critical path |
| Embeddings (OSNet) | new ~per-person | hidden | already dynamic-batched; keep off critical path |
| **Per-frame wall** | **~300–450 effective** | **≤ 130** | overlap so wall ≈ max(stage), not sum |

---

## 3. Optimization cycles (ordered by impact ÷ risk)

### Cycle 017 — Async / overlapped Triton dispatch (enable the idle GPU) — HIGHEST IMPACT
- Turn on `async_dispatch`; pipeline the per-frame model calls (person→pose→gaze→posture→scene) so dispatch, GPU compute, and CPU postprocess **overlap** instead of serializing.
- Batch all person crops of a frame into one request per model (person_detector/gaze/posture already support `max_batch_size`≥8; OSNet ≤64).
- Raise Triton `dynamic_batching.max_queue_delay` tuning + `preferred_batch_size` to coalesce across frames where offline.
- **Expected:** GPU util 2 %→50–80 %; inference wall 367→≤120 ms/frame. **Risk:** low–med (ordering/attribution — preserve §5.4 batch mapping). **Verify:** GPU-util figure + per-model RTT before/after.

### Cycle 018 — GPU/vectorized postprocess (kill the 66 %) — HIGHEST IMPACT
- Move mask decode (`process_mask`), NMS, and gaze/posture/scene decode off per-crop Python: vectorize with NumPy/torch on GPU; decode whole-frame tensors in one shot.
- Eliminate per-object Python loops in the scene normalizer + SRVL hot path (already partly vectorized — extend).
- **Expected:** postprocess 739→≤200 ms/frame. **Risk:** med (numerical parity — adversarially diff outputs vs baseline). **Verify:** correctness agreement table + postprocess wall figure.

### Cycle 019 — Pose kinematics off the CPU critical path
- Run pose-kinematics on GPU (or vectorized batch) and **off the per-frame critical path** (compute on a worker lane, join at persistence).
- **Expected:** 11.5→≤5 ms/frame, removed from serial path. **Risk:** low. **Verify:** stage-timing figure.

### Cycle 020 — Decode + preprocess pipelining
- Overlap video decode + letterbox + tensor build with inference (producer/consumer, pinned host buffers, bounded queue per §8.6).
- Optional NVDEC/GPU decode if available under CUDA 12.8.
- **Expected:** decode/preprocess hidden under inference. **Risk:** low–med (bounded-queue memory). **Verify:** wall-time figure.

### Cycle 021 — Concurrency / worker scaling (governed, §8.1.1)
- Scale offline worker/thread concurrency to the GPU's real ceiling now that the GPU is fed; declare queue topology + resource ceilings + duplicate-worker checks.
- **Expected:** fill remaining GPU headroom. **Risk:** med (contention — measure GPU/DB/Redis). **Verify:** concurrency-scaling decision table.

### Cycle 022 — Persistence / Redis / DB batching
- Bulk-write detections/poses/embeddings; pipeline Redis writes; batch Postgres inserts to remove per-row latency.
- **Expected:** persistence off critical path. **Risk:** low. **Verify:** persistence-timing figure.

---

## 4. Sequencing & exit

Run **017 → 018** first (they unlock the bulk of the speedup: feed the GPU + cut postprocess). Re-benchmark after each — the §12.5 full RTX-5090 run is the only acceptance authority. Stop when the full `combined.mp4` all-models run is **≤ 600 s** with correctness parity (no detection/pose/identity regression vs the pre-optimization baseline) and the GPU-util figure shows the card is actually working.

## 5. Guardrails
- No accuracy regression: every cycle ships a correctness-agreement table vs the frozen baseline audit.
- Streaming safety: each cycle declares `streaming_compatibility`; offline-only levers (whole-file batching) must not ship to the live profile (§8.6).
- Measured, not assumed: GPU util, per-model RTT, per-stage wall, queue time, persistence time, and correctness must all appear in each cycle's figure bundle (§7.1.1).
