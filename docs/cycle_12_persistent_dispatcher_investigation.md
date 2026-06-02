# Cycle 12 Persistent Async Dispatcher Investigation

**Last updated:** 2026-06-02

**Status:** PHASE A MEASUREMENT COMPLETE; implementation candidate pending.
**Decision authority:** no acceptance, rejection, skip, closure, or priority
change is valid without a completed production Linux RTX 5090 benchmark on
`combined.mp4`.

## Problem Statement

The accepted Cycle 9b exact-slice + Top-K route still has a Step 2 frame wall of
`540.399 s` on `combined.mp4`, and two B.1 repeat benchmarks stayed inside small
run-to-run variance rather than proving a new optimization. The latest evidence
shows that client-side decode/NMS is small while gRPC/Triton wait plus server
execution remains dominant.

This cycle targets single-process Python orchestration around async Triton
dispatch boundaries. It does not add models, change behavior input size, widen
the batch window, or alter persisted detection semantics.

## Source-of-truth references

| Kind | Reference | Why it matters |
|---|---|---|
| File | `backend/apps/video_analysis/tasks.py` | Contains `_run_triton_frame_level_inference`, `_AsyncLoopRunner`, `_dispatch_task_inputs`, `_infer_task_batch`, and `_process_batch_items`. |
| File | `backend/apps/video_analysis/services/inference_orchestrator.py` | Contains `InferenceOrchestrator.run_inference_batch`, true-batch payload construction, and bounded async dispatch. |
| File | `tools/prod/prod_watch_benchmark_metrics.sh` | Live production watcher used to monitor jobs and evidence artifacts. |
| File | `tools/prod/prod_collect_benchmark_metrics.py` | Production evidence collector for FPS, GPU, RTT, DB parity, and audit metrics. |
| File | `tools/prod/prod_run_async_dispatch_profile_benchmark.sh` | Reproducible Cycle 12 Phase A wrapper for the `combined.mp4` profiling benchmark. |
| Workflow | `.github/workflows/inference-parallelization.yml` | CI syntax-checks the new production helper and runs the touched unit suites. |
| Doc | `docs/production_inference_benchmark.md` | Benchmark authority table, including Cycle 9b B.1 repeat evidence. |
| Doc | `docs/cycle_9b_compact_postproc_results.md` | Records the B.1 repeat evidence that motivates this cycle. |
| Doc | `docs/cycle_11_input_size_results.md` | Records why input `320 -> 256` is not accepted despite speed improvements. |
| Doc | `docs/cycle_9b_batch_window_results.md` | Records why `max_frames=4` is not accepted despite Step 2 improvement. |
| Job | `be4ba9ee-4786-48e9-8334-28feb237a1fb` | Accepted Cycle 9b exact-slice + Top-K baseline. |
| Job | `df7f832f-de81-4f92-89c7-fd213bdba7fa` | Second B.1 repeat proving no compact-backend decision was made. |

## Current Evidence

| Evidence | Measurement | Interpretation |
|---|---:|---|
| Accepted Top-K Step 2 wall | `540.399 s` | Current accepted inference-wall baseline. |
| B.1 repeat 2 Step 2 wall | `546.702 s` | Repeat-only run stayed near baseline. |
| B.1 repeat 2 behavior RTT mean | `84.360 ms` | Behavior RTT stayed near accepted baseline. |
| B.1 repeat 2 decode/NMS | `4.243 ms/batch` | Decode/NMS is too small to carry a `>=10 %` Step 2 win by itself. |
| B.1 repeat 2 infer wait | `44.162 ms` | Wait/server execution still dominates the decode probe. |
| Cycle 11.A `256` Step 2 wall | `391.673 s` | Performance hypothesis worked, but correctness failed. |
| Cycle 9b B.4 Step 2 wall | `512.445 s` | Larger batch window moved Step 2, but correctness/tracking failed. |

The next lever should therefore attack orchestration wait without changing the
model output contract.

## Root Cause Hypothesis

The current code already uses a job-scoped async event loop, but the synchronous
frame loop still crosses into that loop through blocking `async_runner.run(...)`
calls around each dispatch group. In crop-frame mode, each batch must:

1. dispatch full-frame `person_detection`,
2. decode and track person boxes,
3. build per-person crop tensors,
4. dispatch `behavior_ensemble_gaze_slice_topk`,
5. decode behavior outputs and emit frame callbacks.

This serial dependency cannot be removed blindly because person-box reuse,
tracking IDs, LPM state, and frame-order callbacks are stateful. The safe first
step is to measure async boundary count and wall time in production, then decide
whether a persistent producer/consumer dispatcher can overlap useful work
without breaking ordering.

## Expected Gain

The named lever is single-process orchestration. The expected gain is
hypothesis-only until production measurement lands:

| Metric | Baseline | Candidate target |
|---|---:|---:|
| Step 2 frame wall | `540.399 s` | `<=486.359 s` for acceptance (`>=10 %` reduction) |
| DB-completed FPS | `4.439` | measurable improvement with correctness parity |
| Behavior RTT mean | `84.865 ms` | no regression |
| GPU average utilization | `9.344 %` | measurable increase or no regression with wall-time win |
| Model agreement F1 | `100 %` reference | `>=99.5 %` per behavior model unless stricter doc gate applies |

## Phase A Measurement Plan

Instrumentation is required before persistent-dispatch code:

1. Add an opt-in `TRITON_ASYNC_DISPATCH_PROFILING` setting. **Staged in code.**
2. Measure each async boundary around `async_runner.run(...)`. **Staged in code.**
3. Persist the summary into `inference_audit.json` as
   `step2.async_dispatch.summary`.
   **Staged in code.**
4. Extend production metric collection and watcher output to show this summary.
   **Staged in code.**
5. Run `combined.mp4` on production with the accepted 320 Top-K route and the
   profiling flag enabled. **Complete.**

Reproducible command used for the clean production benchmark:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_run_async_dispatch_profile_benchmark.sh \
  --tag cycle12-async-dispatch-profile-$(date -u +%Y%m%dT%H%M%SZ)
```

Phase A result: see `docs/cycle_12_persistent_dispatcher_results.md`. Clean
production replay `cycle12-async-dispatch-profile-clean-20260602T213441Z` / job
`dfa1f138-7086-418a-ba17-9999cd12b9ac` completed `4541/4541` frames. It
measured `349.643 s` async-dispatch blocking wall, with `behavior_all`
responsible for `338.779 s`. It did not implement a persistent dispatcher, so no
Cycle 12 acceptance/rejection decision exists.

## Candidate Implementation After Measurement

Only if Phase A proves the candidate can overlap useful wall-time:

1. Introduce a persistent bounded dispatcher coroutine that owns model dispatch
   requests for the offline job.
2. Keep stateful frame ordering in the main thread.
3. Use a strict bounded queue and preserve `TRITON_MAX_INFLIGHT_REQUESTS`.
4. Keep fallback to the current blocking path behind an env flag.
5. Benchmark before any decision.

Phase A changed the constraint: a candidate that only removes the synchronous
loop-crossing bridge is unlikely to pass the `>=10 %` Step 2 gate. The next
candidate must overlap behavior wait/server execution while preserving ordered
frame commits.

## Risk Assessment

| Risk | Level | Mitigation |
|---|---|---|
| Tracking or person-box reuse order changes | High | Phase A is measurement only; later implementation must preserve ordered commit. |
| Hidden correctness drift | High | Use full DB parity and model-agreement gates. |
| More in-flight tensors increase RSS | Medium | Bound dispatcher queue and keep `max_frames=2`. |
| Instrumentation overhead changes benchmark | Low | Profiling records only counts and `perf_counter` timings; compare with accepted baseline and repeat variance. |
| No measurable boundary cost | Medium | Candidate is rejected by production data; move to next sorted cycle. |

## Rollback Strategy

For Phase A instrumentation, set:

```env
TRITON_ASYNC_DISPATCH_PROFILING=0
```

For any future persistent-dispatch implementation, keep a separate runtime flag
so production can return to the current blocking dispatch path with one env
change and worker restart.

## Acceptance Criteria

This cycle can be accepted only if all conditions are true:

1. Production `combined.mp4` benchmark completes on the Linux RTX 5090 server.
2. Candidate Step 2 frame wall improves by at least `10 %` versus `540.399 s`.
3. DB-completed FPS improves.
4. Behavior RTT mean and p95 do not regress materially.
5. Detection rows, bbox rows, embeddings, and StudentTrack count stay within the
   accepted 320 Top-K correctness gate.
6. Model-agreement F1@IoU0.5 stays within the documented gate.
7. Evidence is recorded in `docs/production_inference_benchmark.md`,
   `docs/crop_frame_optimization_execution.md`, `docs/inference_parallelization_plan.md`,
   and `AGENTS.md`.

Until then, status remains **STAGED / MEASUREMENT ACTIVE**, not accepted.
