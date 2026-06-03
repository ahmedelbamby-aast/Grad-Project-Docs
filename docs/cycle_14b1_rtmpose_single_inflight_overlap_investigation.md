# Cycle 14.B1 RTMPose Single-Inflight Overlap Investigation

**Last updated:** 2026-06-03

**Status:** NOT ACCEPTED by production benchmark. Replay
`cycle14b-overlap-20260603T143000Z` / job
`b366807a-5e14-4a37-aade-a555ae85cdf0` preserved DB/model parity, but DB FPS
and pose-tail wall were flat versus the accepted Cycle 16.B baseline.

## Problem Statement

Cycle 14.A measured the remaining Step 2 through-pose tail and proved it is
dominated by RTMPose runtime/provider work after the frame loop. The current
pose upload loop waits for `PoseRuntime.infer_from_detections(...)` on each
frame before preparing the next frame's pose payload.

This sub-cycle tests whether a single-inflight worker can overlap frame
read/box selection/payload preparation for frame `N+1` while RTMPose provider
work for frame `N` is running.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Result doc | `docs/cycle_14a_pose_tail_decomposition_results.md` | Proves the measured pose tail and provider-runtime dominance. |
| Benchmark doc | `docs/production_inference_benchmark.md` | Section 32 is the latest production benchmark authority for this tail. |
| Plan doc | `docs/crop_frame_optimization_execution.md` | Sorts Cycle 14.B1 before later compact, Redis, streaming, and worker-count work. |
| Code | `backend/apps/video_analysis/tasks.py` | Owns the post-frame-loop pose upload loop and inference audit event. |
| Code | `backend/apps/pipeline/services/pose_runtime.py` | Owns RTMPose crop/resize, payload build, provider batch dispatch, and SimCC decode. |
| Tool | `tools/prod/prod_collect_benchmark_metrics.py` | Captures Step 2, RTT, GPU, DB parity, and pose-tail fields. |
| Tool | `tools/prod/prod_watch_benchmark_metrics.sh` | Live bounded watcher for production benchmark progress. |
| Result doc | `docs/cycle_14b_rtmpose_scenario_results.md` | Records the B1 non-acceptance decision and comparison with B2. |
| Constitution | `.specify/memory/constitution.md` | Requires production benchmark authority before decision. |

## Evidence From Previous Cycle

| Metric | Cycle 14.A value |
|---|---:|
| Step 2 frame wall | `465.856 s` |
| Step 2 through pose upload | `690.164 s` |
| Through-pose tail | `224.308 s` |
| Drain after frame loop | `200.778 s` |
| Runtime wall | `190.176 s` |
| Runtime provider batch wall | `186.090 s` |
| Provider async batch wall | `154.171 s` |
| Frame read | `7.493 s` |
| Payload build | `0.026 s` |
| Record build | `0.477 s` |

## Hypothesis

The overlap candidate can reduce wall time only by overlapping provider wait
with non-provider work in the next pose iteration. The expected ceiling is
therefore modest unless the current synchronous loop also causes avoidable
idle gaps in the client transport.

## Candidate Design

Add a disabled-by-default mode:

```text
POSE_TAIL_OPTIMIZATION_MODE=overlap
```

The implementation must:

1. Keep at most one RTMPose provider job in flight.
2. Build the current frame payload on the main thread.
3. Finalize the previous provider job before submitting the current one when
   required by ordering.
4. Preserve frame order, tracking IDs, pose records, and metadata.
5. Fall back to synchronous behavior when the flag is unset.

## Risk Assessment

| Risk | Level | Mitigation |
|---|---|---|
| Pose record order drift | Medium | Store frame index with each future and append results only through ordered finalization. |
| Extra thread contention | Medium | Use one worker only; no broad Celery worker increase. |
| Inference RTT regression | Medium | Benchmark behavior RTT and RTMPose RTT mean/p95. |
| Correctness drift | High | Require DB parity and model-agreement F1. |

## Acceptance Criteria

| Gate | Requirement |
|---|---|
| Production authority | Completed `combined.mp4` benchmark on production Linux RTX 5090. |
| Performance | Measurable DB FPS or Step 2 through-pose improvement versus Cycle 16.B and Cycle 14.A. |
| Protection | No material behavior RTT, GPU memory, DB, or Redis regression. |
| Correctness | Detection/BBox/embedding rows, tracks, and model-agreement F1 preserved. |
| Rollback | Set `POSE_TAIL_OPTIMIZATION_MODE=off` and restart Celery workers. |

## Production Decision

The completed benchmark did not meet the performance gate. DB FPS was
`5.347791 -> 5.345623`, Step 2 through-pose wall was
`682.475 s -> 682.405 s`, and pose tail was `221.777 s -> 222.567 s`.
Cycle 14.B1 is therefore not accepted. The side-by-side decision is recorded in
`docs/cycle_14b_rtmpose_scenario_results.md`.
