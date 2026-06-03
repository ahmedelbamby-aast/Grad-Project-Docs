# Cycle 14.B RTMPose Scenario Results

**Last updated:** 2026-06-03

**Decision:** Cycle 14.B2 `cross_frame_batch` is ACCEPTED. Cycle 14.B1
`overlap` is NOT ACCEPTED. Cycle 14.B2 first run is REJECTED; the fixed rerun
is the accepted candidate.

## Summary

Cycle 14.B split the RTMPose tail optimization into two separately benchmarked
scenarios because Cycle 14.A proved the tail but did not prove which scheduling
strategy would win.

The result is clear:

1. `overlap` preserved DB/model parity but did not improve throughput.
2. first `cross_frame_batch` improved throughput but dropped pose records, so it
   was rejected.
3. fixed `cross_frame_batch` preserved DB/model parity, increased pose coverage,
   reduced RTMPose calls, and improved end-to-end FPS.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Baseline metrics | `backend/logs/cycle14b-overlap-20260603T143000Z/baseline_cycle16b_metrics.json` | Accepted Cycle 16.B baseline used for B1 and B2 comparison. |
| B1 metrics | `backend/logs/cycle14b-overlap-20260603T143000Z/overlap_metrics.json` | Production metrics for single-inflight overlap. |
| B1 agreement | `backend/logs/cycle14b-overlap-20260603T143000Z/model_agreement_cycle16b_vs_overlap.json` | DB/model correctness evidence for B1. |
| B2 rejected metrics | `backend/logs/cycle14b-cross-frame-batch16-20260603T144500Z/cross_frame_batch_metrics.json` | First B2 run; rejected because pose records dropped. |
| B2 accepted metrics | `backend/logs/cycle14b-cross-frame-batch16-r2-20260603T150000Z/cross_frame_batch_metrics.json` | Fixed B2 production metrics. |
| B2 accepted agreement | `backend/logs/cycle14b-cross-frame-batch16-r2-20260603T150000Z/model_agreement_cycle16b_vs_cross_frame_batch.json` | Fixed B2 DB/model correctness evidence. |
| Code | `backend/apps/video_analysis/tasks.py` | Implements `POSE_TAIL_OPTIMIZATION_MODE` scheduling. |
| Code | `backend/apps/pipeline/services/pose_runtime.py` | Implements prepared RTMPose inputs and prepared-input inference. |
| Tool | `tools/prod/prod_run_cycle14b_pose_tail_scenario_benchmark.sh` | Reproducible production benchmark wrapper. |

## Production Runs

| Scenario | Replay | Job | Status |
|---|---|---|---|
| Baseline | `cycle16b-redis-coalescing-20260603T025823Z` | `b2dfa987-afc5-4b96-ab12-6799b149ac25` | `completed`, `4541/4541` |
| B1 overlap | `cycle14b-overlap-20260603T143000Z` | `b366807a-5e14-4a37-aade-a555ae85cdf0` | `completed`, `4541/4541` |
| B2 first run | `cycle14b-cross-frame-batch16-20260603T144500Z` | `2dfc1470-e01e-4d81-96cf-5f493fdf898f` | `completed`, `4541/4541` |
| B2 fixed rerun | `cycle14b-cross-frame-batch16-r2-20260603T150000Z` | `6b42a557-b954-4954-a2f8-de54634229eb` | `completed`, `4541/4541` |

## Comparison

| Metric | Baseline | B1 overlap | B2 first run | B2 fixed rerun |
|---|---:|---:|---:|---:|
| DB FPS | `5.347791` | `5.345623` | `5.970799` | `5.680314` |
| DB elapsed | `849.136 s` | `849.480 s` | `760.535 s` | `799.428 s` |
| Step 2 frame wall | `460.698 s` | `459.838 s` | `450.515 s` | `462.188 s` |
| Step 2 through pose | `682.475 s` | `682.405 s` | `592.181 s` | `633.939 s` |
| Pose tail | `221.777 s` | `222.567 s` | `141.666 s` | `171.751 s` |
| GPU avg util | `11.030 %` | `11.582 %` | `12.939 %` | `12.168 %` |
| GPU peak util | `52.000 %` | `51.000 %` | `51.000 %` | `51.000 %` |
| Peak VRAM | `15725 MiB` | `15731 MiB` | `15731 MiB` | `15731 MiB` |
| RTMPose calls | `5047` | `5046` | `943` | `1199` |
| Pose records | `19157` | `19152` | `15075` | `19180` |
| Detection rows | `72744` | `72744` | `72744` | `72744` |
| BBox rows | `72744` | `72744` | `72744` | `72744` |
| Embedding rows | `72578` | `72578` | `72578` | `72578` |
| Student tracks | `53` | `53` | `53` | `53` |
| Model agreement F1 | baseline | `100.000 %` | `100.000 %` | `100.000 %` |

## B2 Fixed Rerun Deltas

| Metric | Baseline | B2 fixed rerun | Delta |
|---|---:|---:|---:|
| DB FPS | `5.347791` | `5.680314` | `+6.22 %` |
| DB elapsed | `849.136 s` | `799.428 s` | `-5.85 %` |
| Step 2 through pose | `682.475 s` | `633.939 s` | `-7.11 %` |
| Pose tail | `221.777 s` | `171.751 s` | `-22.56 %` |
| RTMPose calls | `5047` | `1199` | `-76.24 %` |
| Provider async batch wall | `154.171 s` | `108.482 s` | `-29.63 %` |
| Provider batch wall | `186.090 s` | `136.834 s` | `-26.47 %` |
| GPU avg util | `11.030 %` | `12.168 %` | `+10.32 %` |
| Pose records | `19157` | `19180` | `+0.12 %` |

## Decision

| Scenario | Decision | Reason |
|---|---|---|
| B1 overlap | NOT ACCEPTED | DB FPS was effectively flat and pose tail did not improve. |
| B2 first run | REJECTED | Performance improved, but pose records dropped `19157 -> 15075`; this violates the every-signal requirement. |
| B2 fixed rerun | ACCEPTED | Throughput, pose tail, RTMPose call count, provider wall, and GPU avg improved while DB/model parity stayed exact and pose coverage increased. |

## Production State

Accepted runtime flags:

```text
POSE_TAIL_OPTIMIZATION_MODE=cross_frame_batch
POSE_CROSS_FRAME_BATCH_SIZE=16
POSE_TAIL_PROFILING=0
EMBEDDING_STAGE_PROFILING=0
TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=1
EMBEDDING_PREFETCH_TRACK_LOOKUP=1
EMBEDDING_REDIS_SIDE_EFFECT_COALESCING=1
```

Rollback:

```text
POSE_TAIL_OPTIMIZATION_MODE=off
```

Then restart Celery workers.

