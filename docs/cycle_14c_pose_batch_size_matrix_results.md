# Cycle 14.C RTMPose Batch-Size Matrix Results

**Last updated:** 2026-06-03

**Decision:** Cycle 14.C1 batch `8` is NOT ACCEPTED. Cycle 14.C2 batch `32`
is NOT ACCEPTED. The accepted Cycle 14.B2 batch `16` remains the production
profile.

## Summary

Cycle 14.C benchmarked two cross-frame RTMPose batch-size scenarios against the
accepted batch-16 baseline and then compared the scenarios directly.

Batch `8` doubled RTMPose provider chunks and regressed total throughput. Batch
`32` reduced outer flushes and improved GPU utilization, but did not improve
DB FPS or Step 2 through-pose wall and regressed RTMPose p95 latency. Both
scenarios preserved DB/model parity, but neither beat the accepted baseline.

Cycle 14.C3 was run after this result as a narrow batch-32 repair candidate:
keep `POSE_CROSS_FRAME_BATCH_SIZE=32`, add guarded
`POSE_PROVIDER_CHUNK_PARALLELISM=2`, and benchmark whether concurrent internal
provider chunks reduce the prior batch-32 p95/provider-wall regression. It is
also not accepted; batch `16` remains accepted.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Baseline metrics | `backend/logs/cycle14c-pose-batch-matrix-20260603T154945Z-batch8/baseline_cycle16b_metrics.json` | Accepted batch-16 baseline metrics copied into the matrix evidence directory. |
| Batch 8 metrics | `backend/logs/cycle14c-pose-batch-matrix-20260603T154945Z-batch8/cross_frame_batch_metrics.json` | Production metrics for Cycle 14.C1. |
| Batch 8 agreement | `backend/logs/cycle14c-pose-batch-matrix-20260603T154945Z-batch8/model_agreement_cycle16b_vs_cross_frame_batch.json` | Correctness evidence for Cycle 14.C1 against batch 16. |
| Batch 32 metrics | `backend/logs/cycle14c-pose-batch-matrix-20260603T154945Z-batch32/cross_frame_batch_metrics.json` | Production metrics for Cycle 14.C2. |
| Batch 32 agreement | `backend/logs/cycle14c-pose-batch-matrix-20260603T154945Z-batch32/model_agreement_cycle16b_vs_cross_frame_batch.json` | Correctness evidence for Cycle 14.C2 against batch 16. |
| Scenario comparison | `backend/logs/cycle14c-pose-batch-matrix-20260603T154945Z/batch32_vs_batch8_metrics.json` | Direct batch-32 versus batch-8 metric comparison. |
| Scenario agreement | `backend/logs/cycle14c-pose-batch-matrix-20260603T154945Z/model_agreement_batch8_vs_batch32.json` | Direct batch-32 versus batch-8 correctness comparison. |
| Tool | `tools/prod/prod_run_cycle14c_pose_batch_size_matrix.sh` | Reproducible matrix wrapper. |
| Follow-up investigation | `docs/cycle_14c3_batch32_parallel_chunks_investigation.md` | Stages the batch-32-specific provider-chunk parallelism candidate. |
| Follow-up results | `docs/cycle_14c3_batch32_parallel_chunks_results.md` | Records the rejected Cycle 14.C3 production benchmark. |

## Production Runs

| Scenario | Replay | Job | Status |
|---|---|---|---|
| Batch 16 baseline | `cycle14b-cross-frame-batch16-r2-20260603T150000Z` | `6b42a557-b954-4954-a2f8-de54634229eb` | `completed`, `4541/4541` |
| 14.C1 batch 8 | `cycle14c-pose-batch-matrix-20260603T154945Z-batch8` | `d3cc149f-9ad3-4dd6-8c14-beed80f7c0d2` | `completed`, `4541/4541` |
| 14.C2 batch 32 | `cycle14c-pose-batch-matrix-20260603T154945Z-batch32` | `7a0b5819-9af4-46b4-a5ee-c46cdd7c1d9c` | `completed`, `4541/4541` |

## Comparison

| Metric | Batch 16 baseline | Batch 8 | Delta 8 vs 16 | Batch 32 | Delta 32 vs 16 |
|---|---:|---:|---:|---:|---:|
| DB FPS | `5.680314` | `5.518671` | `-2.85 %` | `5.624745` | `-0.98 %` |
| DB elapsed | `799.428 s` | `822.843 s` | `+2.93 %` | `807.326 s` | `+0.99 %` |
| Step 2 frame wall | `462.188 s` | `468.768 s` | `+1.42 %` | `459.750 s` | `-0.53 %` |
| Step 2 through pose | `633.939 s` | `655.592 s` | `+3.42 %` | `643.184 s` | `+1.46 %` |
| GPU avg util | `12.168 %` | `11.792 %` | `-3.09 %` | `14.288 %` | `+17.42 %` |
| GPU peak util | `51.000 %` | `53.000 %` | `+3.92 %` | `86.000 %` | `+68.63 %` |
| Peak VRAM | `15731 MiB` | `15917 MiB` | `+1.18 %` | `15725 MiB` | `-0.04 %` |
| Behavior RTT mean | `84.171 ms` | `83.534 ms` | `-0.76 %` | `84.276 ms` | `+0.12 %` |
| Behavior RTT p95 | `130.802 ms` | `129.346 ms` | `-1.11 %` | `130.372 ms` | `-0.33 %` |
| RTMPose calls | `1199` | `2398` | `+100.00 %` | `1199` | `0.00 %` |
| RTMPose RTT mean | `44.055 ms` | `25.899 ms` | `-41.21 %` | `49.731 ms` | `+12.88 %` |
| RTMPose RTT p95 | `46.952 ms` | `43.111 ms` | `-8.18 %` | `100.563 ms` | `+114.18 %` |
| Cross-frame flushes | `1199` | `2398` | `+100.00 %` | `600` | `-49.96 %` |
| Provider chunks | `1199` | `2398` | `+100.00 %` | `1199` | `0.00 %` |
| Provider async wall | `108.482 s` | `122.897 s` | `+13.29 %` | `120.029 s` | `+10.64 %` |
| Pose subjects/results | `19180 / 19180` | `19180 / 19180` | preserved | `19180 / 19180` | preserved |
| DB rows | `72744 / 72744 / 72578` | `72744 / 72744 / 72578` | preserved | `72744 / 72744 / 72578` | preserved |
| Student tracks | `53` | `53` | preserved | `53` | preserved |
| Model agreement F1 | baseline | `100.000 %` | preserved | `100.000 %` | preserved |

## Decision Explanation

| Scenario | Decision | Explanation |
|---|---|---|
| 14.C1 batch 8 | NOT ACCEPTED | It preserved correctness, but DB FPS regressed `-2.85 %`, Step 2 through-pose wall regressed `+3.42 %`, and provider chunks doubled. |
| 14.C2 batch 32 | NOT ACCEPTED | It improved GPU average utilization, but DB FPS still regressed `-0.98 %`, Step 2 through-pose wall regressed `+1.46 %`, and RTMPose p95 regressed `+114.18 %`. |
| Batch 16 baseline | REMAINS ACCEPTED | It remains the best measured batch cap by total throughput and pose-tail wall. |

## Cycle 14.C3 Follow-Up

Cycle 14.C3 replay `cycle14c3-batch32-parallel2-20260603T170117Z`, job
`d71802b2-4c20-4f31-9c06-9854ebbc4eed`, tested batch `32` with
`POSE_PROVIDER_CHUNK_PARALLELISM=2`. It preserved exact DB/model parity but is
**NOT ACCEPTED** because DB FPS regressed `-3.79 %` versus accepted batch `16`,
RTMPose p95 regressed `+87.16 %` versus prior batch `32`, provider async wall
regressed `+15.91 %` versus prior batch `32`, and GPU average fell below both
comparators. Detailed result:
[`docs/cycle_14c3_batch32_parallel_chunks_results.md`](cycle_14c3_batch32_parallel_chunks_results.md).

## Production State

The matrix wrapper restored the accepted production profile:

```text
POSE_TAIL_OPTIMIZATION_MODE=cross_frame_batch
POSE_CROSS_FRAME_BATCH_SIZE=16
POSE_TAIL_PROFILING=0
EMBEDDING_STAGE_PROFILING=0
TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=1
EMBEDDING_PREFETCH_TRACK_LOOKUP=1
EMBEDDING_REDIS_SIDE_EFFECT_COALESCING=1
```

Rollback remains the accepted batch-16 profile above; no new runtime flag is
accepted from Cycle 14.C.
