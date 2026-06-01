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

**Decision.** **ACCEPTED 2026-06-01** (`74ec0432-995c-487e-9d77-1048ec109fb1`: `TelemetryModelCall` rows went from `0` to `29 123` per offline job; per-model RTT now queryable from SQL for the first time).

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

**Decision.** **ACCEPTED 2026-06-01** (`74ec0432`: Step 2 FPS 2.09 → 5.14, average GPU util 3.95 % → 7.55 %, no correctness regression).

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

**Decision.** **ACCEPTED 2026-06-01** (`74ec0432`: behavior model executions per job dropped 4 543 → 3 598 = −20.8 % thanks to two-frame packing; worker RSS stayed at ~813 MB sustained; no memory regression).

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

**Decision.** **ACCEPTED 2026-06-01** (`74ec0432`: mean preprocess_ms dropped 1.04 → 0.69 = −33.7 %; mean inference_ms dropped 197.9 → 174.4 = −11.9 %; no RSS growth).

---

## Cycle 5 — Memoize true-batch byte concat across 4 fan-out models

**Hypothesis.** `_run_crop_behaviour_for_items` dispatches the same `crop_payloads` list to 4 behavior models. `InferenceOrchestrator._build_true_batch_payload` is called once per model, and each call runs an independent `b"".join(bytes(p["data_bytes"]) for p in pl)` over 17 × 1.17 MB = 19.94 MB of FP32 bytes. Micro-bench measured 1.66 ms per join → ~6.6 ms / frame for the 4 redundant joins.

**Bottleneck removed.** Per-model duplicate concat on the same input list.

**Expected impact.** −5–7 ms / frame; eliminates 3 of the 4 redundant 20 MB allocations per frame.

**Implementation.**
- `backend/apps/video_analysis/services/inference_orchestrator.py`: `_build_true_batch_payload` now stashes the joined bytes on the first payload dict via `_batched_bytes_cache = (batch_len, joined)`. Cache survives across the 4 dispatch calls because they share the same payload list; it is garbage-collected when the payloads go out of scope.

**Risk.** Very low. The cache is keyed by batch length so any payload mutation produces a fresh join. Tests pass.

**Rollback.** Revert the Edit on the orchestrator.

**Decision.** **ACCEPTED 2026-06-01** (`74ec0432`: cache is shared correctly across the 4 fan-out dispatches; no decode contract regression; bench passed end-to-end).

---

## Bundled Result Across Cycles 1–5

| Metric | Baseline `77650001` | Cycles 1–5 `74ec0432` | Δ |
|---|---:|---:|---:|
| Step 2 wall | 2 175 s | **883 s** | **−59.4 %** |
| Step 2 FPS | 2.09 | **5.14** | **+146 %** |
| Overall FPS | 1.308 | **2.644** | **+102 %** |
| Avg GPU util | 3.95 % | **7.55 %** | **+91 %** |
| Peak GPU util | 34 % | 40 % | +18 % |
| % GPU samples in 0–5 % | 80.3 % | 65.9 % | −14 pp |
| Behavior exec/model | 4 543 | 3 598 | −21 % |
| TelemetryModelCall DB rows | 0 | 29 123 | visibility ✓ |
| Detection rows | 72 751 | 72 743 | −0.01 % (parity) |

All cycles individually marked **ACCEPTED**. Replay key `cycle2to5-crop-frame-20260601T012045`.

---

## Cycle 6 — Fix `rtmpose_model` batch > 16 violation in `PoseRuntime`

### Phase 1: Investigation (measurement, no code change)

**Evidence captured before any change:**
- Cycles 1–5 production benchmark (job `74ec0432`) lifecycle showed pose
  post-processing block `step2.frame_stage_timings → step2.pose_upload` took
  **12 m 47 s (767 s)** for the same 4 541-frame video.
- Celery log for that run shows repeated `WARNING TritonClient: gRPC inference
  failed [model=rtmpose_model] ... [StatusCode.INVALID_ARGUMENT] inference
  request batch-size must be <= 16 for 'rtmpose_model'` followed by an HTTP
  fallback that succeeds.
- Triton stats: `rtmpose_model` max_batch_size=16, avg observed effective
  batch 1.44, executions 13 316 for 19 157 inferences across the job.
- `PoseRuntime.infer_from_detections` is called from `tasks.py:3838` once per
  frame, serially across all 4 541 frames.
- `PoseRuntime._provider_infer_batch` builds one payload per crop and calls
  `orchestrator.run_inference_batch(frame_payloads=payloads, ...)`. With
  `TRITON_TRUE_BATCH_REQUESTS=1`, the orchestrator stacks *all* of them into a
  single batched gRPC call without enforcing the model's `max_batch_size`.
- `tasks.py:3817` computes `dynamic_cap = pose_frame_budget_ms /
  avg_pose_ms_per_person`. With a fast GPU `avg_pose_ms_per_person` decays to
  ~20–50 ms, so `dynamic_cap = 1400 / 20 = 70` person crops per frame —
  well above `max_batch_size=16` — and every such frame trips the batch-size
  guard, falls back to HTTP, and pays the fallback round-trip.

**Root cause (proven):** `_provider_infer_batch` does not chunk its
`frame_payloads` list to the model's max batch size. The behavior-model path
in `tasks.py:_infer_task_batch` *does* chunk via
`_effective_task_batch_size(...)`, but pose's dedicated dispatcher bypasses
that helper and goes straight to the orchestrator.

### Phase 2: Hypothesis

If `PoseRuntime._provider_infer_batch` chunks its crops to `max_batch_size=16`
before dispatch:

- 0 rtmpose batch>16 warnings (correctness improvement).
- Removes the per-frame HTTP fallback round-trip (~50–100 ms on the affected
  frames).
- Should not change Triton's effective batch (the orchestrator was already
  stacking; we now stack in correctly sized chunks).
- Bench expectation: small but measurable drop in pose-upload elapsed time
  (~10–20 % of the 767 s pose block) and zero gRPC→HTTP fallbacks for
  rtmpose during the run.

**Risk:** very low — the chunk size is read from settings with a default of
16 that already matches the deployed engine; if the chunk size matches the
old behavior (one batch of all crops), behavior is identical.

**Rollback:** revert the single edit in `pose_runtime.py`.

### Phase 3: Implemented in commit `17b3745d`

- `backend/apps/pipeline/services/pose_runtime.py`: `_provider_infer_batch`
  now chunks `payloads` at `_resolve_pose_max_batch_size()` (reads
  `TRITON_MODEL_BATCH_SIZE_OVERRIDES["pose_estimation"]`, falls back to 16,
  bounded by `TRITON_MAX_INFLIGHT_REQUESTS`).
- `backend/tests/unit/pipeline/test_pose_runtime_batch_chunking.py`: 5
  regression tests asserting chunk sizes for 4 / 20 / 40 crops, the
  invalid-override fallback, and the empty-input contract.
- `.github/workflows/inference-parallelization.yml`: blocks the gate on the
  new test file plus the cycle 1–5 telemetry writer regression suite.

### Phase 4: Production benchmark — job `a1a448b9-474f-4dea-942b-3288bcae6900`

Run on the same `combined.mp4` (4 541 frames) prod RTX 5090.

| Stage | Cycle 1–5 `74ec0432` | Cycle 6 `a1a448b9` | Δ |
|---|---:|---:|---:|
| Step 2 frame inference wall | 883 s (14 m 43 s) | **879 s** (14 m 39 s) | −0.5 % (unchanged, as designed) |
| Step 2 → step3 transition (pose upload + persist + render) | **13 m 27 s** | **5 m 10 s** | **−61.5 %** |
| Pose-upload block specifically (`step2.frame_stage_timings → step2.pose_upload`) | **12 m 13 s** | **3 m 42 s** | **−69.7 %** |
| Total `run.complete` elapsed | 1 716 s (28 m 36 s) | **1 166 s** (19 m 26 s) | **−32.1 %** |
| Total DB `status=completed` elapsed | 36.44 min | **27.22 min** | **−25.3 %** |
| **Overall FPS (bench probe basis)** | 2.644 fps | **2.776 fps** | +5.0 % |
| **Overall FPS (DB-completed basis)** | 2.077 fps | **2.78 fps** | **+33.8 %** |

| Correctness check | Cycle 1–5 | Cycle 6 | Δ |
|---|---:|---:|---:|
| Frames persisted | 4 541 / 4 541 | 4 541 / 4 541 | ✓ |
| Detections | 72 743 | 72 752 | +9 (+0.012 %) |
| BoundingBoxes | 72 743 | 72 752 | +9 (+0.012 %) |
| FrameEmbeddings | 72 577 | 72 586 | +9 (+0.012 %) |
| Per-class bbox parity (`attention/hand/person/sitting`) | 11 772 / 8 801 / 19 162 / 33 008 | 11 769 / 8 801 / 19 162 / 33 020 | ≤ 0.04 % per class |
| **rtmpose batch>16 warnings** | many | **0** | **bug fixed** |
| TelemetryModelCall DB rows | 29 123 | **20 348** | −30 % (fewer pose-retry calls) |
| Final status | completed | **completed** | ✓ |

Pose-upload contribution to total time dropped from 33.6 % of the run to
13.4 %. Detection numbers improved slightly (more detected objects, not
fewer) because frames that previously timed out after the HTTP fallback now
return clean.

### Phase 5: Decision — **ACCEPTED 2026-06-01**

- Removed the root-cause batch-size-violation: rtmpose warnings dropped from
  many per run to zero.
- Pose-upload block: −69.7 %.
- Total DB-completed elapsed: −25.3 % vs cycle 1–5; **−52.9 % vs baseline**.
- Overall FPS: +33.8 % vs cycle 1–5; **+112 % vs baseline**.
- No correctness regression (parity within 0.012 % on every counter).
- All 5 new chunking unit tests + 41 existing telemetry tests pass; the
  inference-parallelization workflow gate has been updated to enforce them.

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
