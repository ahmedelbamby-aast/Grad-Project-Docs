# Crop-Frame Optimization Execution Log

**Active baseline:** job `77650001-3c4b-4b0a-94aa-b4eb899b90df` (replay key `roi320-running-crop-frame-20260601T012133`).
Baseline metrics: 4541 / 4541 frames, **1.308 FPS overall**, step-2 wall 2 175 s, avg GPU util **3.95 %**, peak 34 %.

Decomposition source: [`docs/rtt_root_cause_investigation_77650001.md`](rtt_root_cause_investigation_77650001.md).

Each entry below is one optimization cycle. The "Decision" line is the only line that determines whether the change stays in production.

---

## Cycle 1 — Restore per-call telemetry persistence

**Hypothesis.** `apps.telemetry.writer.flush_session` never inserts into `telemetry_model_calls` — only TelemetrySession / Video / Frame / Student rows are written. The JSON file does contain `model_calls` (≈32 899 rows for the baseline job), but the DB has 0. ContextVar propagation and `_try_record_telemetry` are both working (verified with a probe on prod). Without DB rows, no subsequent optimization is measurable from SQL — only from external Triton stats and bench logs.

**Bottleneck removed.** Visibility-only; not a perf cycle.

**Implementation.**
- `backend/apps/telemetry/writer.py`: added `_bulk_insert_model_calls` (batch 1000) and invoked it inside the existing `transaction.atomic()` block of `flush_session`.

**Risk.** 30k–300k bulk-inserted rows per offline job add ~6 MB / job to telemetry storage. Bulk_create in batches of 1000 keeps memory bounded. JSON fallback is unchanged.

**Rollback.** Revert the single Edit in `writer.py` — table is additive-only, no schema change.

**Decision.** **KEPT** (visibility prerequisite for all subsequent cycles).

---

## Cycle 2 — Drop `BATCH_QUEUE_MAX_CONCURRENCY` from 4 → 2

**Hypothesis.** Inflight sweep on the prod RTX 5090 (probe `tools/prod/probe_rtt_multiframe.py`) showed inflight=2 is the saturation knee: 12.82 FPS / 151 ms p50. Inflight=4 regresses to 11.26 FPS / 322 ms because 4 × 4 = 16 simultaneous gRPC calls queue behind a single GPU. Inflight=8 collapses to 7.12 FPS / 1057 ms.

**Bottleneck removed.** Over-concurrency tax that inflates per-frame latency without raising throughput.

**Expected impact.** Per-frame latency at the GPU stack drops 322 → 151 ms (-53 %). Whole-job FPS expected to roughly double if downstream Python orchestration relaxes proportionally; conservative target ≥ 2.5 FPS.

**Implementation.**
- `backend/config/settings/base.py`: default `TRITON_OFFLINE_BATCH_QUEUE_MAX_CONCURRENCY` 4 → **2**.
- `tools/prod/prod_enable_parallel_flow.sh`: prod env writes `TRITON_OFFLINE_BATCH_QUEUE_MAX_CONCURRENCY=2`.

**Risk.** Very low. The knob was historically picked without GPU-saturation data; the new value matches the measured knee exactly.

**Rollback.** Revert both edits; the previous value was 4.

**Decision.** **STAGED for prod-verify.**

---

## Cycle 3 — Raise `BATCH_QUEUE_MAX_FRAMES` from 1 → 2

**Hypothesis.** With `MAX_FRAMES=1` every batch is one frame; the worker does the entire crop-preprocess + true-batch concat + asyncio entry/exit for frame N before any frame N+1 work can begin. Bumping to 2 lets the producer thread prepare frame N+1's crops while frame N is inflight on Triton, reclaiming ~15–25 ms / frame of serial CPU work that currently has no overlap with GPU work.

**Bottleneck removed.** Frame-synchronous Python orchestration that gaps the GPU between dispatches.

**Expected impact.** Additional +30–60 % on top of Cycle 2.

**Implementation.**
- `backend/config/settings/base.py`: default `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES` 4 → **2** (was 4 default in code, 1 in prod env).
- `tools/prod/prod_enable_parallel_flow.sh`: prod env writes `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=2`.

**Risk.** Medium. The previous `MAX_FRAMES` regression history (100 GiB RSS spike) is mitigated by:
- `OFFLINE_TRIM_PROCESS_MEMORY=1` is on,
- True-batch packing keeps only 2 frames' worth of crops in memory (~40 MB FP32),
- `TRITON_NUMPY_OUTPUTS=1` avoids the Python-list materialization that originally caused the spike.

The hard guardrail remains: if RSS exceeds 8 GiB on prod we revert to `MAX_FRAMES=1`.

**Rollback.** Revert both edits.

**Decision.** **STAGED for prod-verify with active RSS watch.**

---

## Cycle 4 — Amortize `malloc_trim` across batches

**Hypothesis.** `OFFLINE_TRIM_PROCESS_MEMORY=1` runs `gc.collect()` + `libc.malloc_trim(0)` after every batch — at `MAX_FRAMES=1` that is every frame. Combined cost is 5–30 ms / frame on prod and is wasted because freed buffers from frame N–1 are usually re-allocated in identical sizes for frame N (no fragmentation to reclaim).

**Bottleneck removed.** Per-frame GC stall in the hot path.

**Expected impact.** −5–25 ms / frame on average.

**Implementation.**
- `backend/apps/video_analysis/tasks.py`: added `OFFLINE_TRIM_EVERY_N_BATCHES` knob with a per-pid counter; trim only fires every Nth batch.
- `backend/config/settings/base.py`: defined `OFFLINE_TRIM_EVERY_N_BATCHES` default 1 (backwards-compatible).
- `tools/prod/prod_enable_parallel_flow.sh`: prod env sets `OFFLINE_TRIM_EVERY_N_BATCHES=8` (so at `MAX_FRAMES=2`, trim every 16 frames).

**Risk.** Low. If RSS climbs, lower the period — the existing guardrail still fires every 16 frames in prod.

**Rollback.** Set the env back to 1.

**Decision.** **STAGED for prod-verify.**

---

## Cycle 5 — Memoize true-batch byte concat across 4 fan-out models

**Hypothesis.** `_run_crop_behaviour_for_items` dispatches the same `crop_payloads` list to 4 behavior models. `InferenceOrchestrator._build_true_batch_payload` is called once per model, and each call runs an independent `b"".join(bytes(p["data_bytes"]) for p in pl)` over 17 × 1.17 MB = 19.94 MB of FP32 bytes. Micro-bench measured 1.66 ms per join → ~6.6 ms / frame for the 4 redundant joins.

**Bottleneck removed.** Per-model duplicate concat on the same input list.

**Expected impact.** −5–7 ms / frame; eliminates 3 of the 4 redundant 20 MB allocations per frame.

**Implementation.**
- `backend/apps/video_analysis/services/inference_orchestrator.py`: `_build_true_batch_payload` now stashes the joined bytes on the first payload dict via `_batched_bytes_cache = (batch_len, joined)`. Cache survives across the 4 dispatch calls because they share the same payload list; it is garbage-collected when the payloads go out of scope.

**Risk.** Very low. The cache is keyed by batch length so any payload mutation produces a fresh join. Tests pass.

**Rollback.** Revert the Edit on the orchestrator.

**Decision.** **STAGED for prod-verify.**

---

## Pending Cycles (not implemented in this session)

Listed in order. Only proceed if the staged cycles above do not lift FPS to the target.

| # | Optimization | Expected lift | Cost / risk |
|---:|---|---|---|
| 6 | Persistent asyncio producer/consumer frame loop — drop `async_runner.run(...)` per-frame round-trip; one persistent dispatcher coroutine processes frames as they leave decode | +10–20 % | medium refactor of `_process_batch_items` and `_AsyncLoopRunner` |
| 7 | Triton ensemble / BLS that fuses 4 behavior models into one gRPC call (shared input tensor, multi-output) | +20–40 % | high; requires `ensemble_scheduling` config + parity test |
| 8 | TRT NMS plugin server-side — return compact detections instead of dense `[84,2100]` / `[14,2100]` grids | small wall, large bandwidth | engine rebuild + decoder refactor |
| 9 | Re-export behavior engines at 256×256 (smaller than current 320) | +20–40 % step-2 | medium — accuracy parity required |
| 10 | Triton CUDA Shared Memory for input tensors | tiny | medium-high — SHM lifecycle |

---

## Production-validation requirement (per mission rules)

Every cycle marked **STAGED** above must, before being marked **ACCEPTED**:

1. Be committed and pushed to prod.
2. Be benchmarked via `tools/prod/prod_run_parallel_flow_benchmark.sh` on `combined.mp4` (4541 frames).
3. Have results recorded in [`docs/production_inference_benchmark.md`](production_inference_benchmark.md) as a Before/After table.
4. Show no regression in detection rows, tracking continuity, persistence, rendering, or embeddings.
5. Have `AGENTS.md` updated with the new replay key + job id + accept/reject status.

Until all five are true, the optimization is **STAGED**, not **ACCEPTED**.
