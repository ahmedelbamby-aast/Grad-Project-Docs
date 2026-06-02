# Cycle 12.B Behavior-Wait Overlap Dispatcher Results

**Last updated:** 2026-06-03

**Status:** NEEDS FURTHER ITERATION. NOT ACCEPTED.

Cycle 12.B ran the bounded crop-frame behavior-wait overlap candidate on the
production Linux RTX 5090 server. The candidate improved throughput and
preserved model agreement, but it failed the RTT/latency gate because behavior
request RTT regressed materially.

## Source-of-truth references

| Kind | Reference | Why it matters |
|---|---|---|
| File | `backend/apps/video_analysis/tasks.py` | Implements the `TRITON_CROP_FRAME_BEHAVIOR_OVERLAP` crop-frame behavior overlap path. |
| File | `tools/prod/prod_run_behavior_overlap_benchmark.sh` | Reproducible production benchmark wrapper. |
| Evidence | `backend/logs/cycle12-behavior-overlap-20260602T223350Z/async_dispatch_profile_metrics.json` | Candidate metrics collected from the completed production job. |
| Evidence | `backend/logs/cycle12-behavior-overlap-20260602T223350Z/model_agreement_320_topk_vs_async_dispatch_profile.json` | Per-model agreement vs. accepted 320 Top-K baseline. |
| Doc | `docs/production_inference_benchmark.md` §25 | Benchmark-history table and decision authority. |
| Job | `46ba8b2a-3c61-4d89-b7b6-63ec72159428` | Candidate production job, completed `4541/4541` frames. |

## Decision Table

| Metric | Accepted 320 Top-K baseline | Cycle 12.B candidate | Delta |
|---|---:|---:|---:|
| Replay key | `cycle9b-topk-crop-frame-20260602T041900` | `cycle12-behavior-overlap-20260602T223350Z` |  |
| Job ID | `be4ba9ee-4786-48e9-8334-28feb237a1fb` | `46ba8b2a-3c61-4d89-b7b6-63ec72159428` |  |
| Status | `completed` | `completed` |  |
| Processed frames | `4541/4541` | `4541/4541` | `0.00 %` |
| DB-completed elapsed | `1022.952 s` | `874.104 s` | `-14.55 %` |
| DB-completed FPS | `4.439` | `5.195` | `+17.03 %` |
| Step 2 frame wall | `540.399 s` | `395.495 s` | `-26.81 %` |
| Step 2 through pose upload | `767.589 s` | `616.156 s` | `-19.73 %` |
| Behavior RTT mean | `84.865 ms` | `115.420 ms` | `+36.00 %` |
| Behavior RTT p95 | `128.056 ms` | `224.661 ms` | `+75.44 %` |
| GPU avg util | `9.344 %` | `11.274 %` | `+20.65 %` |
| GPU peak util | `53.000 %` | `77.000 %` | `+45.28 %` |
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
| Production benchmark completed | PASS | Replay `cycle12-behavior-overlap-20260602T223350Z`, job `46ba8b2a-3c61-4d89-b7b6-63ec72159428`. |
| Step 2 wall improves by at least `10 %` | PASS | `540.399 s → 395.495 s` (`-26.81 %`). |
| DB-completed FPS improves | PASS | `4.439 → 5.195` (`+17.03 %`). |
| GPU utilization improves | PASS | average `9.344 % → 11.274 %`, peak `53 % → 77 %`. |
| Correctness/model agreement preserved | PASS | All model-agreement F1 values are `>=99.716 %`; tracks remain `53`. |
| Behavior RTT has no material regression | FAIL | mean RTT `+36.00 %`; p95 RTT `+75.44 %`. |

Decision: **NEEDS FURTHER ITERATION**. The candidate is not accepted because it
fails the RTT/latency gate, even though it improves wall time.

## Why The Metrics Moved

The implementation overlaps useful work by allowing batch `N` behavior
inference to remain in flight while batch `N+1` is prepared. That explains the
large Step 2 and total-wall improvement.

The same implementation starts the current batch's behavior request before
finalizing the previously pending behavior request. That creates brief periods
with two crop-frame behavior jobs competing for Triton/GPU service. The evidence
is the behavior RTT regression: mean `115.420 ms` and p95 `224.661 ms`, while
the baseline was `84.865 ms` mean and `128.056 ms` p95.

## Next Task

Stage Cycle 12.C: keep the useful overlap, but enforce single behavior job
in-flight semantics. The planned change is to build the current batch's crop
payloads, finalize the previous pending behavior job, then submit the current
behavior job. This should preserve overlap of previous behavior with current
person/crop preparation while avoiding two simultaneous behavior batches.

No Cycle 12.C decision is valid until the same `combined.mp4` production
benchmark and model-agreement gates complete.
