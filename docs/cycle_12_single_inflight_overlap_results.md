# Cycle 12.C Single-In-Flight Behavior Overlap Results

**Last updated:** 2026-06-03

**Status:** ACCEPTED.

Cycle 12.C keeps the useful crop-frame behavior overlap from Cycle 12.B while
preventing two behavior jobs from being in flight at once. It was accepted by a
completed production Linux RTX 5090 benchmark on the canonical `combined.mp4`
video.

## Source-of-truth references

| Kind | Reference | Why it matters |
|---|---|---|
| File | `backend/apps/video_analysis/tasks.py` | Implements single-in-flight behavior overlap under `TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=1`. |
| File | `tools/prod/prod_run_behavior_overlap_benchmark.sh` | Reproducible production benchmark wrapper. |
| Evidence | `backend/logs/cycle12-single-inflight-overlap-20260602T225821Z/async_dispatch_profile_metrics.json` | Candidate metrics collected from production. |
| Evidence | `backend/logs/cycle12-single-inflight-overlap-20260602T225821Z/model_agreement_320_topk_vs_async_dispatch_profile.json` | Per-model agreement vs. accepted Top-K baseline. |
| Doc | `docs/production_inference_benchmark.md` §26 | Benchmark-history authority. |
| Job | `069a217f-fa43-48cc-bf18-c946d53bb3ee` | Accepted production job, completed `4541/4541` frames. |

## Decision Table

| Metric | Accepted 320 Top-K baseline | Cycle 12.C candidate | Delta |
|---|---:|---:|---:|
| Replay key | `cycle9b-topk-crop-frame-20260602T041900` | `cycle12-single-inflight-overlap-20260602T225821Z` |  |
| Job ID | `be4ba9ee-4786-48e9-8334-28feb237a1fb` | `069a217f-fa43-48cc-bf18-c946d53bb3ee` |  |
| Status | `completed` | `completed` |  |
| Processed frames | `4541/4541` | `4541/4541` | `0.00 %` |
| DB-completed elapsed | `1022.952 s` | `935.516 s` | `-8.55 %` |
| DB-completed FPS | `4.439` | `4.854` | `+9.35 %` |
| Step 2 frame wall | `540.399 s` | `459.461 s` | `-14.98 %` |
| Step 2 through pose upload | `767.589 s` | `680.619 s` | `-11.33 %` |
| Behavior RTT mean | `84.865 ms` | `83.936 ms` | `-1.09 %` |
| Behavior RTT p95 | `128.056 ms` | `130.200 ms` | `+1.67 %` |
| GPU avg util | `9.344 %` | `10.332 %` | `+10.57 %` |
| GPU peak util | `53.000 %` | `53.000 %` | `0.00 %` |
| Peak VRAM | `16055 MiB` | `15725 MiB` | `-2.06 %` |
| Detection rows | `72762` | `72744` | `-0.02 %` |
| BBox rows | `72762` | `72744` | `-0.02 %` |
| Embedding rows | `72596` | `72578` | `-0.02 %` |
| Student tracks | `53` | `53` | `0.00 %` |
| `attention_tracking` F1@IoU0.5 | `100.000 %` reference | `99.732 %` | pass |
| `hand_raising` F1@IoU0.5 | `100.000 %` reference | `99.716 %` | pass |
| `person_detection` F1@IoU0.5 | `100.000 %` reference | `100.000 %` | pass |
| `sitting_standing` F1@IoU0.5 | `100.000 %` reference | `99.935 %` | pass |

## Acceptance Decision

| Gate | Result | Evidence |
|---|---|---|
| Production benchmark completed | PASS | Replay `cycle12-single-inflight-overlap-20260602T225821Z`, job `069a217f-fa43-48cc-bf18-c946d53bb3ee`. |
| Step 2 wall improves by at least `10 %` | PASS | `540.399 s → 459.461 s` (`-14.98 %`). |
| DB-completed FPS improves | PASS | `4.439 → 4.854` (`+9.35 %`). |
| GPU utilization improves | PASS | average `9.344 % → 10.332 %`. |
| Correctness/model agreement preserved | PASS | all model F1 values `>=99.716 %`; tracks unchanged at `53`. |
| Behavior RTT has no material regression | PASS | mean improved `-1.09 %`; p95 drift was `+1.67 %`. |

Decision: **ACCEPTED**. Cycle 12.C should replace Cycle 12.B as the production
overlap implementation.

## Why The Metrics Moved

Cycle 12.C preserved the useful part of Cycle 12.B: prior behavior execution
overlaps current person/crop preparation. It removed the measured contention by
submitting the current behavior job only after the previous pending behavior job
is finalized. That is why Step 2 wall still improves while behavior RTT returns
near the accepted baseline.

The tradeoff is that Cycle 12.C gives back part of Cycle 12.B's wall-time win:
Step 2 wall is `459.461 s` instead of Cycle 12.B's `395.495 s`. The RTT gate is
more important than the extra speculative overlap because the system must avoid
building throughput on hidden Triton queueing.

## Rollback

Set:

```env
TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=0
```

Then restart Celery workers. No Triton restart is required because no model
repository or engine changed.
