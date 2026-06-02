# Cycle 12.B Behavior-Wait Overlap Dispatcher Investigation

**Last updated:** 2026-06-03

**Status:** STAGED - production benchmark required before any decision.

Cycle 12 Phase A measured async-dispatch blocking wall, but did not implement an
optimization candidate. This document records the metric-driven decision and the
next candidate scope before code changes.

## Source-of-truth references

| Kind | Reference | Why it matters |
|---|---|---|
| File | `backend/apps/video_analysis/tasks.py` | Contains `_run_triton_frame_level_inference`, `_process_batch_items`, and crop-frame behavior dispatch. |
| File | `backend/config/settings/base.py` | Runtime flags for candidate rollout and rollback. |
| File | `backend/tests/unit/video_analysis/test_concurrent_model_dispatch.py` | Focused equivalence coverage for crop-frame behavior overlap and callback ordering. |
| File | `tools/prod/prod_run_async_dispatch_profile_benchmark.sh` | Phase A benchmark wrapper that produced the decision evidence. |
| File | `tools/prod/prod_run_behavior_overlap_benchmark.sh` | Reproducible production wrapper for the Cycle 12.B candidate. |
| Doc | `docs/cycle_12_persistent_dispatcher_results.md` | Production evidence and next-candidate constraints. |
| Doc | `docs/production_inference_benchmark.md` §24 | Benchmark-history authority for the Phase A numbers. |
| Job | `dfa1f138-7086-418a-ba17-9999cd12b9ac` | Clean Cycle 12 Phase A production job. |

## Metric-Driven Decision

| Candidate option | Decision | Evidence |
|---|---|---|
| Bridge-only async replacement | Do not implement. | Phase A estimated the loop-crossing gap at about `35.3 s`, below the `>=10 %` Step 2 gate. |
| Bounded behavior-wait overlap | Implement behind a rollback flag. | `behavior_all` owns `338.779 s` of `349.643 s` async-dispatch blocking wall. |
| Person-only overlap | Do not prioritize. | `person_detection` owns `10.864 s`, too small for the acceptance gate. |

This is a candidate-selection decision, not an optimization acceptance decision.
Cycle 12 remains incomplete until the candidate is deployed and benchmarked on
production Linux RTX 5090.

## Candidate Scope

The candidate will keep the accepted 320 exact-slice + Top-K behavior route and
add an env-gated crop-frame dispatcher overlap:

1. process batch `N` person detection, tracking, and crop-payload build;
2. start `behavior_all` inference for batch `N` without blocking the main
   thread immediately;
3. process batch `N+1` person detection, tracking, and crop-payload build while
   batch `N` behavior inference is in flight;
4. finalize batch `N` behavior results and callbacks before finalizing batch
   `N+1`;
5. keep `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=2`.

Implementation is guarded by `TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=1`. The
default `0` path keeps the previous blocking behavior finalization path.

## Expected Gain

The target is to overlap some of the `behavior_all` wait with next-batch
primary work. The acceptance target remains:

| Metric | Baseline | Candidate gate |
|---|---:|---:|
| Step 2 frame wall | `540.399 s` | `<=486.359 s` |
| DB-completed FPS | `4.439` | measurable improvement |
| Behavior RTT mean | `84.865 ms` | no material regression |
| Student tracks | `53` | unchanged unless documented variance gate allows |
| Model agreement F1@IoU0.5 | baseline `100 %` | `>=99.5 %` per behavior model |

## Risk Assessment

| Risk | Level | Mitigation |
|---|---|---|
| Frame callback order changes | High | Finalize pending behavior jobs strictly in original batch order. |
| Tracking/cache state changes | High | Keep person detection, cache refresh, and incremental tracking on the main thread. |
| More live tensors increase RSS | Medium | One in-flight behavior job only; keep `max_frames=2`. |
| Triton queue contention regresses RTT | Medium | Benchmark RTT mean/p95 and GPU telemetry against accepted baseline. |
| Correctness drift | High | Run DB parity and model-agreement gates before any decision. |

## Rollback Strategy

Default-off flag:

```env
TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=0
```

Candidate enablement:

```env
TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=1
```

Rollback is setting the flag back to `0` and restarting Celery workers. No
Triton model or engine changes are part of this candidate.

## Validation Plan

Before production:

1. focused unit tests proving overlap output matches the existing sequential
   crop-frame path;
2. production helper dry-run;
3. workflow shell/compile checks;
4. doc-date and reading-order verification.

Production:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_run_behavior_overlap_benchmark.sh \
  --tag cycle12-behavior-overlap-$(date -u +%Y%m%dT%H%M%SZ)
```

No decision is valid until the production benchmark, model agreement, DB parity,
RTT, GPU, and memory rows are recorded in the connected Markdown files.
