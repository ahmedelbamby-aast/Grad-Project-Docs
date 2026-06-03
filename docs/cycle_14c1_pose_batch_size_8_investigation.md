# Cycle 14.C1 RTMPose Batch Size 8 Investigation

**Last updated:** 2026-06-03

**Status:** NOT ACCEPTED. Replay
`cycle14c-pose-batch-matrix-20260603T154945Z-batch8` completed on production
but regressed DB FPS and Step 2 through-pose wall against batch `16`.

## Problem Statement

The accepted cross-frame RTMPose batch cap is `16`. Batch `8` may reduce
per-flush latency or CPU/GPU contention, but it will also increase expected
RTMPose request count. The production benchmark must decide whether the latency
tradeoff helps total throughput.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Parent investigation | `docs/cycle_14c_pose_batch_size_matrix_investigation.md` | Defines the shared matrix and gates. |
| Baseline result | `docs/cycle_14b_rtmpose_scenario_results.md` | Accepted batch-16 reference. |
| Code | `backend/apps/video_analysis/tasks.py` | Uses `POSE_CROSS_FRAME_BATCH_SIZE` to flush prepared RTMPose crops. |
| Code | `backend/apps/pipeline/services/pose_runtime.py` | Chunks prepared crops to the RTMPose engine cap. |
| Tool | `tools/prod/prod_run_cycle14c_pose_batch_size_matrix.sh` | Runs this scenario reproducibly. |

## Candidate

```text
POSE_TAIL_OPTIMIZATION_MODE=cross_frame_batch
POSE_CROSS_FRAME_BATCH_SIZE=8
```

## Expected Direction

| Metric | Expected movement |
|---|---|
| RTMPose calls | Higher than batch 16. |
| Per-request latency | Potentially lower. |
| Pose tail | Unknown; must be measured. |
| DB FPS | Unknown; must be measured. |

## Acceptance Gate

Batch 8 is accepted only if the completed production benchmark improves
throughput or pose tail against batch 16, preserves correctness, and wins or is
clearly justified against Cycle 14.C2.
