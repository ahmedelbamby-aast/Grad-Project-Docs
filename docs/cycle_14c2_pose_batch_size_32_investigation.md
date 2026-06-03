# Cycle 14.C2 RTMPose Batch Size 32 Investigation

**Last updated:** 2026-06-03

**Status:** NOT ACCEPTED. Replay
`cycle14c-pose-batch-matrix-20260603T154945Z-batch32` completed on production
and improved GPU utilization, but it regressed DB FPS, Step 2 through-pose
wall, and RTMPose p95 against batch `16`.

## Problem Statement

The accepted cross-frame RTMPose batch cap is `16`, matching the deployed
RTMPose TensorRT engine cap. Batch `32` cannot create a single 32-crop Triton
request because `PoseRuntime` chunks provider calls at the engine cap, but it
can reduce outer-loop flushes and may lower Python orchestration overhead.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Parent investigation | `docs/cycle_14c_pose_batch_size_matrix_investigation.md` | Defines the shared matrix and gates. |
| Baseline result | `docs/cycle_14b_rtmpose_scenario_results.md` | Accepted batch-16 reference. |
| Config | `backend/models/triton_repository_cuda12/rtmpose_model/config.pbtxt` | Proves the engine cap that prevents true 32-crop Triton requests. |
| Code | `backend/apps/video_analysis/tasks.py` | Controls outer-loop cross-frame flushes. |
| Code | `backend/apps/pipeline/services/pose_runtime.py` | Enforces engine-cap chunking inside provider inference. |
| Tool | `tools/prod/prod_run_cycle14c_pose_batch_size_matrix.sh` | Runs this scenario reproducibly. |

## Candidate

```text
POSE_TAIL_OPTIMIZATION_MODE=cross_frame_batch
POSE_CROSS_FRAME_BATCH_SIZE=32
```

## Expected Direction

| Metric | Expected movement |
|---|---|
| Outer flush count | Lower than batch 16. |
| Triton RTMPose calls | Similar to batch 16 because provider chunks at 16. |
| Memory and wait time | Potentially higher due to longer accumulation before flush. |
| Pose tail and DB FPS | Unknown; must be measured. |

## Acceptance Gate

Batch 32 is accepted only if the completed production benchmark improves
throughput or pose tail against batch 16, preserves correctness, and wins or is
clearly justified against Cycle 14.C1.
