# Cycle 14.B2 RTMPose Cross-Frame Batching Investigation

**Last updated:** 2026-06-03

**Status:** PHASE A STARTED. No optimization decision exists until a completed
production Linux RTX 5090 `combined.mp4` benchmark compares this candidate with
the accepted Cycle 16.B baseline, Cycle 14.A measurement, and Cycle 14.B1.

## Problem Statement

Cycle 14.A measured `5047` RTMPose model calls for `19157` pose subjects. The
current pose upload loop batches crops only within one frame. Because the
median classroom frame contains far fewer people than the deployed RTMPose
batch cap, most RTMPose calls under-fill the TensorRT engine.

This sub-cycle tests whether cross-frame batching can reduce RTMPose request
count and provider wait by forming larger ordered crop batches across adjacent
frames.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Result doc | `docs/cycle_14a_pose_tail_decomposition_results.md` | Records `5047` RTMPose calls, `19157` subjects, and provider wall dominance. |
| Benchmark doc | `docs/production_inference_benchmark.md` | Section 32 is the benchmark authority for the baseline tail. |
| Plan doc | `docs/crop_frame_optimization_execution.md` | Sorts Cycle 14.B2 beside Cycle 14.B1 and before later ideas. |
| Code | `backend/apps/pipeline/services/pose_runtime.py` | Owns crop preparation, true-batch dispatch, and response decoding. |
| Code | `backend/apps/video_analysis/tasks.py` | Owns frame-order pose record construction and artifact writes. |
| Tool | `tools/prod/prod_collect_benchmark_metrics.py` | Captures pose-tail and model-call telemetry for benchmark evidence. |
| Constitution | `.specify/memory/constitution.md` | Requires production benchmark authority before decision. |

## Evidence From Previous Cycle

| Metric | Cycle 14.A value |
|---|---:|
| Pose subjects / results | `19157` |
| Provider chunk count | `5047` |
| RTMPose model calls | `5047` |
| RTMPose mean RTT | `18.809 ms` |
| RTMPose p95 RTT | `88.508 ms` |
| Runtime provider batch wall | `186.090 s` |
| Provider async batch wall | `154.171 s` |
| RTMPose input shape | `[4, 3, 256, 192]` |

## Hypothesis

If crops are grouped across frames up to the existing RTMPose batch cap, the
request count should fall toward `ceil(19157 / 16) = 1198` true-batch calls.
This should reduce Python/gRPC orchestration wall and improve GPU work
packing, provided order and per-frame pose records are preserved.

## Candidate Design

Add a disabled-by-default mode:

```text
POSE_TAIL_OPTIMIZATION_MODE=cross_frame_batch
POSE_CROSS_FRAME_BATCH_SIZE=16
```

The implementation must:

1. Prepare crop tensors and mapping transforms for multiple frames.
2. Submit crops in ordered cross-frame chunks capped by the RTMPose engine
   batch size.
3. Decode each response back to its original frame, tracking ID, and source
   box.
4. Preserve `pose_payload.records`, `pose_records_by_frame`, artifact content,
   and metadata shape.
5. Emit pose-tail profile fields for batch size, chunk count, and response
   count.

## Risk Assessment

| Risk | Level | Mitigation |
|---|---|---|
| Frame/order mapping bug | High | Keep explicit frame index and crop index metadata for every crop. |
| Memory growth | Medium | Flush bounded chunks; do not retain raw frames after crops are prepared. |
| TensorRT batch cap violation | Medium | Bound by `POSE_CROSS_FRAME_BATCH_SIZE` and existing `pose_estimation` override. |
| Correctness drift | High | Require exact DB parity and model-agreement F1 before acceptance. |

## Acceptance Criteria

| Gate | Requirement |
|---|---|
| Production authority | Completed `combined.mp4` benchmark on production Linux RTX 5090. |
| Performance | RTMPose calls/chunks decrease and DB FPS or Step 2 through-pose wall improves. |
| Protection | No material behavior RTT, GPU memory, DB, Redis, or render regression. |
| Correctness | Detection/BBox/embedding rows, tracks, and model-agreement F1 preserved. |
| Comparison | Compare Cycle 14.B2 directly against Cycle 14.B1 in the final table. |
| Rollback | Set `POSE_TAIL_OPTIMIZATION_MODE=off` and restart Celery workers. |

