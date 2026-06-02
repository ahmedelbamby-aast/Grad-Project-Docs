# Equations Manifest (Implementation Grounded)

**Last updated:** 2026-06-02

| equation_id | equation | label | codepath_or_dependency_usage | evidence_refs |
|---|---|---|---|---|
| EQ-01 | `visible_joint_ratio = visible / total` | IMPLEMENTED | Pose evaluation metric computation | `E:/grad_project/scripts/pose_eval/metrics.py:81-91`; `E:/grad_project/paper/01_architecture_deployment.md` |
| EQ-02 | `overlay_completeness = len(drawable_frames) / len(frames_with_pose)` | IMPLEMENTED | Pose overlay completeness metric | `E:/grad_project/scripts/pose_eval/metrics.py:94-104`; `E:/grad_project/paper/01_architecture_deployment.md` |
| EQ-03 | `temporal_jitter = mean(hypot(dx,dy))` | IMPLEMENTED | Temporal jitter from per-joint displacement | `E:/grad_project/scripts/pose_eval/metrics.py:107-131`; `E:/grad_project/paper/01_architecture_deployment.md` |
| EQ-04 | `temporal_stability_score = 1 / (1 + max(jitter,0))` | IMPLEMENTED | Stability score in pose metrics | `E:/grad_project/scripts/pose_eval/metrics.py:134-135`; `E:/grad_project/paper/01_architecture_deployment.md`; `E:/grad_project/paper/06_observability_testing_science.md` |
| EQ-05 | `delay = base * 2^(attempt-1)` (bounded by helper) | IMPLEMENTED | RTSP reconnect backoff helper + use in task sleep delay | `E:/grad_project/backend/apps/tracking/rtsp_worker.py:6`; `E:/grad_project/backend/apps/cameras/tasks.py:144`; `E:/grad_project/paper/02_ingestion_orchestration.md` |
| EQ-06 | `effective_fps = 1000 / frame_delta_ms` (for positive delta) | IMPLEMENTED | Runtime ingest effective FPS derivation | `E:/grad_project/backend/apps/pipeline/runtime_ingestion.py:165`; `E:/grad_project/paper/02_ingestion_orchestration.md` |
| EQ-07 | `error_rate = error_count / total_count` | IMPLEMENTED | Runtime telemetry summary error-rate calculation | `E:/grad_project/backend/apps/pipeline/runtime_ingestion.py:588`; `E:/grad_project/paper/06_observability_testing_science.md` |
| EQ-08 | `timeout_rate = timeout_count / total_count` | IMPLEMENTED | Runtime telemetry timeout-rate calculation | `E:/grad_project/backend/apps/pipeline/runtime_ingestion.py:758-759`; `E:/grad_project/paper/02_ingestion_orchestration.md` |
| EQ-09 | `cosine_similarity = dot(a,b)/(||a||*||b||)` | IMPLEMENTED | ReID similarity scoring primitive | `E:/grad_project/backend/apps/tracking/reid.py:22-31`; `E:/grad_project/paper/03_detection_tracking.md` |
| EQ-10 | `detect_now = ((frame_idx - 1) % detect_every_n) == 0` | IMPLEMENTED | Detection cadence decision | `E:/grad_project/backend/apps/video_analysis/tasks.py:1320`; `E:/grad_project/paper/03_detection_tracking.md` |
| EQ-11 | `interp = start + ratio * (end - start)` | IMPLEMENTED | Linear bbox interpolation for continuity path | `E:/grad_project/backend/apps/video_analysis/tasks.py:1457-1463`; `E:/grad_project/paper/03_detection_tracking.md` |
| EQ-12 | `ema[t] = alpha*x[t] + (1-alpha)*ema[t-1]` | IMPLEMENTED | Pose smoothing in RTMPose pipeline | `E:/grad_project/backend/apps/pipeline/services/rtmpose_pipeline.py:92-96`; `E:/grad_project/paper/04_pose_temporal_artifacts.md` |
| EQ-13 | `pose_score = mean(keypoint_confidences)` | IMPLEMENTED | Per-pose score used in artifact preparation | `E:/grad_project/backend/apps/video_analysis/tasks.py:763-770`; `E:/grad_project/paper/04_pose_temporal_artifacts.md` |
| EQ-14 | `throughput_ratio = optimized_throughput / baseline_throughput` | IMPLEMENTED | Benchmark profile comparison metric | `E:/grad_project/backend/apps/pipeline/benchmark.py:160`; `E:/grad_project/paper/06_observability_testing_science.md` |
| EQ-15 | `latency_improvement = (baseline_p95 - optimized_p95)/baseline_p95` | IMPLEMENTED | Benchmark latency improvement metric | `E:/grad_project/backend/apps/pipeline/benchmark.py:164`; `E:/grad_project/paper/06_observability_testing_science.md` |
| EQ-16 | `canary_pass = (p95<=120 && p99<=220 && fallback<=0.05 && error<=0.03)` | IMPLEMENTED | Rollout canary gate predicate | `E:/grad_project/backend/apps/pipeline/services/rollout_execution.py:77`; `E:/grad_project/paper/06_observability_testing_science.md` |
| EQ-17 | Confidence intervals/significance/power-analysis equations in benchmark gate | NOT EVIDENCED | No implementation evidence in reviewed benchmark/validation modules | `E:/grad_project/backend/apps/pipeline/benchmark.py`; `E:/grad_project/backend/apps/pipeline/validation.py`; `E:/grad_project/paper/06_observability_testing_science.md` |
