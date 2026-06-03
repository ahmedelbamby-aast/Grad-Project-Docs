# Cycle 14.C RTMPose Cross-Frame Batch-Size Matrix Investigation

**Last updated:** 2026-06-03

**Status:** PHASE A STAGED. No decision exists until each sub-scenario has a
completed production Linux RTX 5090 `combined.mp4` benchmark against the
accepted Cycle 14.B2 baseline, followed by a direct scenario-to-scenario
comparison.

## Problem Statement

Cycle 14.B2 accepted ordered cross-frame RTMPose batching at
`POSE_CROSS_FRAME_BATCH_SIZE=16`. That setting reduced RTMPose calls and pose
tail wall, but it is still a tunable scheduling cap rather than a proven global
optimum. The deployed `rtmpose_model` TensorRT config has `max_batch_size: 16`,
so caps below 16 change Triton request size, while caps above 16 change
outer-loop flush frequency and still chunk internally at the engine cap.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Result doc | `docs/cycle_14b_rtmpose_scenario_results.md` | Accepted baseline: batch cap `16`, replay `cycle14b-cross-frame-batch16-r2-20260603T150000Z`. |
| Benchmark doc | `docs/production_inference_benchmark.md` | Production benchmark authority and before/after table location. |
| Plan doc | `docs/crop_frame_optimization_execution.md` | Sorted cycle map and acceptance governance. |
| Code | `backend/apps/video_analysis/tasks.py` | Owns `POSE_TAIL_OPTIMIZATION_MODE=cross_frame_batch` flush behavior. |
| Code | `backend/apps/pipeline/services/pose_runtime.py` | Owns prepared RTMPose input batching and engine-cap chunking. |
| Config | `backend/models/triton_repository_cuda12/rtmpose_model/config.pbtxt` | Declares RTMPose `max_batch_size: 16` and preferred batches. |
| Tool | `tools/prod/prod_run_cycle14c_pose_batch_size_matrix.sh` | Reproducible wrapper for the two sub-scenario benchmarks and direct comparison. |
| Constitution | `.specify/memory/constitution.md` | Requires real production benchmark authority for any decision. |

## Sub-Cycles

| Sub-cycle | Candidate | Rationale |
|---|---|---|
| 14.C1 | `POSE_CROSS_FRAME_BATCH_SIZE=8` | Tests whether smaller flushes reduce tail latency, queue wait, or side-stage interference enough to offset more RTMPose calls. |
| 14.C2 | `POSE_CROSS_FRAME_BATCH_SIZE=32` | Tests whether fewer outer-loop flushes reduce Python orchestration overhead while `PoseRuntime` still chunks provider calls at the engine cap. |

The current accepted baseline remains:

```text
POSE_TAIL_OPTIMIZATION_MODE=cross_frame_batch
POSE_CROSS_FRAME_BATCH_SIZE=16
```

## Hypotheses

| Hypothesis | Evidence required |
|---|---|
| Smaller cap helps latency | Batch 8 improves DB FPS or pose tail without increasing RTMPose RTT enough to lose total wall. |
| Larger cap helps orchestration | Batch 32 reduces flush/runtime overhead while preserving pose count and memory safety. |
| Batch 16 remains best | Both scenarios fail to improve DB FPS, pose tail, or correctness against the accepted baseline. |

## Required Metrics

| Metric | Gate |
|---|---|
| DB FPS and DB elapsed | Candidate must improve materially against accepted batch 16. |
| Step 2 frame wall and through-pose wall | Candidate must not hide a regression behind post-stage noise. |
| Pose tail and RTMPose call count | Candidate must reduce or preserve the measured RTMPose tail without losing signals. |
| Behavior RTT mean/p95 | Candidate must not regress the non-pose inference path. |
| GPU avg/peak and VRAM | Candidate must stay within accepted resource bounds. |
| DB row parity and model-agreement F1 | Candidate must preserve persisted outputs. |
| Pose records | Candidate must not drop pose signals; increases are allowed only when explained by better coverage. |

## Acceptance Criteria

| Gate | Requirement |
|---|---|
| Individual benchmark | Each sub-cycle completes `combined.mp4` on production Linux RTX 5090 against the accepted batch-16 baseline. |
| Direct comparison | The two sub-cycle results are compared directly before any acceptance decision. |
| Correctness | Detection/BBox/embedding rows, StudentTracks, model-agreement F1, and pose records pass gates. |
| Rollback | Restore batch `16`, keep `cross_frame_batch`, and restart Celery workers. |
| Documentation | Results are recorded in `docs/production_inference_benchmark.md`, this cycle result doc, `AGENTS.md`, and the plan docs. |

## Rollback

```text
POSE_TAIL_OPTIMIZATION_MODE=cross_frame_batch
POSE_CROSS_FRAME_BATCH_SIZE=16
POSE_TAIL_PROFILING=0
EMBEDDING_STAGE_PROFILING=0
```

Then restart Celery workers.
