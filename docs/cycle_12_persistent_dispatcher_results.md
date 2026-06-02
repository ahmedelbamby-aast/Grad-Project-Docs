# Cycle 12 Persistent Async Dispatcher Results

**Last updated:** 2026-06-02

**Status:** PHASE A MEASUREMENT COMPLETE. NO OPTIMIZATION DECISION.

This document records the production Linux RTX 5090 measurement for Cycle 12.
No persistent dispatcher candidate was implemented in this run, so the result is
`HYPOTHESIS_ONLY`: it may justify or reshape the next implementation candidate,
but it does not accept, reject, skip, close, or complete Cycle 12.

## Source-of-truth references

| Kind | Reference | Why it matters |
|---|---|---|
| File | `backend/apps/video_analysis/tasks.py` | Emits `step2.async_dispatch.summary` from the crop-frame Step 2 loop. |
| File | `tools/prod/prod_run_async_dispatch_profile_benchmark.sh` | Reproducible wrapper used for the production measurement. |
| File | `tools/prod/prod_collect_benchmark_metrics.py` | Captures benchmark metrics and async-dispatch summary. |
| File | `tools/prod/prod_watch_benchmark_metrics.sh` | Live watcher now displays async-dispatch rows. |
| Doc | `docs/cycle_12_persistent_dispatcher_investigation.md` | Phase A hypothesis, risk, rollback, and acceptance gates. |
| Benchmark | `backend/logs/cycle12-async-dispatch-profile-clean-20260602T213441Z/async_dispatch_profile_metrics.json` | Clean production measurement metrics. |
| Benchmark | `backend/logs/cycle12-async-dispatch-profile-clean-20260602T213441Z/model_agreement_320_topk_vs_async_dispatch_profile.json` | Baseline-agreement correctness evidence. |
| Job | `dfa1f138-7086-418a-ba17-9999cd12b9ac` | Clean Cycle 12 profiling job, completed `4541/4541` frames. |
| Commit | `3d2c8e8a` | Deployed SHA for the no-rebuild profiling wrapper. |

## Production Runs

| Run | Replay key | Job ID | Deployed SHA | Notes |
|---|---|---|---|---|
| Initial profile | `cycle12-async-dispatch-profile-20260602T211343Z` | `c7a1c069-48d1-46ae-ae56-00df62bad593` | `3fecf5b9` | Completed, but wrapper passed `--roi-behavior-input-size 320`, causing an unnecessary 320 engine rebuild before the run. Kept as evidence but not the clean comparison row. |
| Clean profile | `cycle12-async-dispatch-profile-clean-20260602T213441Z` | `dfa1f138-7086-418a-ba17-9999cd12b9ac` | `3d2c8e8a` | Completed without the redundant rebuild. Use this row for the Phase A comparison. |

## Before / After Measurement

| Metric | Accepted 320 Top-K baseline | Cycle 12 clean profile | Delta |
|---|---:|---:|---:|
| Step 2 frame wall | `540.399 s` | `552.886 s` | `+2.31 %` |
| Step 2 through pose upload | `767.589 s` | `778.491 s` | `+1.42 %` |
| DB-completed elapsed | `1022.952 s` | `1034.673 s` | `+1.15 %` |
| DB-completed FPS | `4.439` | `4.389` | `-1.13 %` |
| Behavior RTT mean | `84.865 ms` | `84.376 ms` | `-0.58 %` |
| Behavior RTT p95 | `128.056 ms` | `129.656 ms` | `+1.25 %` |
| GPU avg util | `9.344 %` | `9.935 %` | `+6.32 %` |
| GPU peak util | `53.000 %` | `87.000 %` | `+64.15 %` |
| Peak VRAM | `16055 MiB` | `16045 MiB` | `-0.06 %` |
| Detection rows | `72762` | `72744` | `-0.02 %` |
| BBox rows | `72762` | `72744` | `-0.02 %` |
| Embedding rows | `72596` | `72578` | `-0.02 %` |
| Student tracks | `53` | `53` | `0.00 %` |

## Async Dispatch Measurement

| Boundary | Calls | Total ms | Mean ms | Max ms |
|---|---:|---:|---:|---:|
| `ALL` | `4507` | `349643.451` | `77.578` | `171.388` |
| `run_inference_batch:behavior_all` | `3597` | `338778.502` | `94.184` | `171.388` |
| `run_inference_batch:person_detection` | `909` | `10864.072` | `11.952` | `33.261` |
| `orchestrator.close` | `1` | `0.877` | `0.877` | `0.877` |

Interpretation:

| Observation | Evidence | Constraint for next implementation |
|---|---|---|
| Async-dispatch blocking wall is large. | `349.643 s`, `63.2 %` of the `552.886 s` Step 2 wall. | A persistent dispatcher may be worth testing only if it overlaps or reduces this wait without changing ordering. |
| The behavior boundary dominates async wall. | `338.779 s`, `96.9 %` of async-dispatch total. | Target `behavior_all` wait first; person detection overlap alone is too small. |
| Pure Python boundary overhead is probably below the acceptance gate. | Behavior boundary mean `94.184 ms` versus telemetry RTT mean `84.376 ms`, a gap of `9.808 ms/call` or about `35.3 s` over `3597` calls. | A candidate that only removes loop-crossing overhead is unlikely to reach the `>=10 %` Step 2 gate. It must overlap behavior wait/server execution. |
| Correctness stayed within the accepted gate. | Minimum model-agreement F1@IoU0.5 was `99.716 %`; StudentTrack count stayed `53`. | Future candidate can use the same agreement workflow as its correctness gate. |

## Model Agreement

| Model | Agreement F1@IoU0.5 | Precision | Recall | Count delta |
|---|---:|---:|---:|---:|
| `attention_tracking` | `99.732 %` | `99.779 %` | `99.686 %` | `-0.09 %` |
| `hand_raising` | `99.716 %` | `99.773 %` | `99.659 %` | `-0.11 %` |
| `person_detection` | `100.000 %` | `100.000 %` | `100.000 %` | `0.00 %` |
| `sitting_standing` | `99.935 %` | `99.930 %` | `99.939 %` | `0.01 %` |

## Decision Gate

| Gate | Result | Reason |
|---|---|---|
| Production Linux RTX 5090 benchmark completed | PASS | Clean replay completed `4541/4541` frames. |
| Correctness preserved | PASS for measurement | Rows stayed within `0.02 %`; model F1 stayed `>=99.716 %`; tracks stayed `53`. |
| Candidate optimization implemented | FAIL | This run only enabled profiling instrumentation. |
| Step 2 wall improved by `>=10 %` | FAIL for optimization acceptance | Step 2 wall regressed by `2.31 %`; profiling-only run is not a candidate. |
| Cycle 12 accepted/rejected/skipped | NO DECISION | No persistent dispatcher candidate has been benchmarked. |

## Next Recommendation

Do not implement a candidate that merely replaces `async_runner.run(...)` with a
different synchronous bridge. Phase A shows that the measured removable
loop-crossing gap is probably about `35.3 s`, below the acceptance gate.

The next candidate must explicitly overlap useful work:

1. keep `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=2`;
2. preserve ordered frame commits and callback order;
3. submit `person_detection` for the next prepared batch while `behavior_all`
   for the current batch is in flight, or prove another overlap point with a
   bounded queue;
4. expose a rollback flag before production benchmark;
5. benchmark against `cycle9b-topk-crop-frame-20260602T041900` and the clean
   Cycle 12 profile evidence.

Until that candidate is implemented and benchmarked, Cycle 12 remains active
and incomplete.
