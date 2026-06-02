# Cycle 12.C Single-In-Flight Behavior Overlap Investigation

**Last updated:** 2026-06-03

**Status:** STAGED - production benchmark required before any decision.

Cycle 12.B proved that overlapping crop-frame behavior wait with later batch
work can materially reduce Step 2 wall time, but it also proved that allowing
two behavior jobs to overlap in Triton regresses behavior RTT. Cycle 12.C keeps
the useful overlap point and removes the measured contention source.

## Source-of-truth references

| Kind | Reference | Why it matters |
|---|---|---|
| File | `backend/apps/video_analysis/tasks.py` | Owns `_process_batch_items` and the crop-frame behavior overlap ordering. |
| File | `backend/config/settings/base.py` | Owns `TRITON_CROP_FRAME_BEHAVIOR_OVERLAP`. |
| Evidence | `backend/logs/cycle12-behavior-overlap-20260602T223350Z/async_dispatch_profile_metrics.json` | Cycle 12.B metrics showing wall-time gain plus RTT regression. |
| Evidence | `backend/logs/cycle12-behavior-overlap-20260602T223350Z/model_agreement_320_topk_vs_async_dispatch_profile.json` | Correctness agreement proof for the prior candidate. |
| Doc | `docs/cycle_12_overlap_dispatcher_results.md` | Decision table and reason Cycle 12.B was not accepted. |
| Doc | `docs/production_inference_benchmark.md` §25 | Benchmark-history authority for the prior candidate. |
| Job | `46ba8b2a-3c61-4d89-b7b6-63ec72159428` | Cycle 12.B completed production job. |

## Problem Statement

Cycle 12.B improved Step 2 wall `540.399 s → 395.495 s` (`-26.81 %`) and
DB-completed FPS `4.439 → 5.195` (`+17.03 %`), but behavior RTT mean regressed
`84.865 ms → 115.420 ms` and p95 regressed `128.056 ms → 224.661 ms`.

That RTT regression violates the Cycle 12 acceptance gate.

## Root Cause

The current overlap ordering starts the current batch's `behavior_all` request
before finalizing the previously pending behavior request. That creates a short
window where two crop-frame behavior jobs can be in flight at the same time.

Evidence:

| Metric | Baseline | Cycle 12.B |
|---|---:|---:|
| Behavior RTT mean | `84.865 ms` | `115.420 ms` |
| Behavior RTT p95 | `128.056 ms` | `224.661 ms` |
| GPU peak util | `53 %` | `77 %` |
| Step 2 wall | `540.399 s` | `395.495 s` |

Inference from the code path and metrics: the overlap hides useful wall time,
but the current submit-before-finalize order increases per-request queuing.

## Candidate

Change the overlap ordering:

1. process current batch person detection, tracking, and crop-payload build;
2. finalize the previous pending behavior job;
3. submit the current behavior job;
4. keep the current job pending for the next batch;
5. drain the final pending job at end of stream.

This preserves overlap between previous behavior execution and current
person/crop preparation, but prevents two behavior jobs from being in flight.

The staged code keeps the same rollback flag,
`TRITON_CROP_FRAME_BEHAVIOR_OVERLAP`, and changes only the enabled-path
ordering in `backend/apps/video_analysis/tasks.py`.

## Expected Gain

The candidate may give back a small part of Cycle 12.B's wall-time improvement,
but it should reduce the RTT regression. Acceptance gates:

| Metric | Required result |
|---|---|
| Step 2 frame wall | better than accepted Top-K baseline by `>=10 %` |
| DB-completed FPS | measurable improvement vs accepted Top-K baseline |
| Behavior RTT mean/p95 | no material regression vs accepted Top-K baseline |
| GPU utilization | no material regression vs accepted Top-K baseline |
| Correctness/model agreement | F1@IoU0.5 `>=99.5 %` per model; tracks unchanged unless documented |

## Risk Assessment

| Risk | Level | Mitigation |
|---|---|---|
| Wall gain shrinks too much | Medium | Production benchmark decides; do not accept if Step 2 gate fails. |
| Ordering regression | Medium | Preserve finalization order and existing callback order tests. |
| RTT still regresses | Medium | Decision remains not accepted; next lever must move server-side. |
| Correctness drift | High | Run DB parity and model-agreement gates. |

## Rollback Strategy

Set:

```env
TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=0
```

Then restart Celery workers. No Triton model/config changes are part of this
candidate.

## Validation Plan

Before production:

1. focused unit tests for crop-frame overlap equivalence and callback order;
2. shell syntax checks for production wrappers;
3. documentation date and reading-order verification;
4. focused CI pytest set on the production Linux environment.

Production:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_run_behavior_overlap_benchmark.sh \
  --tag cycle12-single-inflight-overlap-$(date -u +%Y%m%dT%H%M%SZ)
```

No decision is valid until the production benchmark, model agreement, DB parity,
RTT, GPU, memory, and rollback rows are recorded in the connected Markdown
files.
