**Last updated:** 2026-06-02

1. Executive Summary
- IMPLEMENTED: RTMPose runtime wiring performs source-frame ROI crop/resize, Triton inference, SimCC decode, and remap to source coordinates in active code paths (`E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:43`, `E:/grad_project/backend/apps/pipeline/services/rtmpose_pipeline.py:51`, `E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:240`).
- PARTIAL: ONNX/OpenVINO/TensorRT artifact lifecycle exists, but the active pose runtime path is Triton-orchestrated; a direct local ONNX/OpenVINO/TensorRT pose execution path is not evidenced in `PoseRuntime` (`E:/grad_project/backend/apps/pipeline/model_lifecycle/rtmpose_manifest.py:17`, `E:/grad_project/backend/apps/pipeline/model_lifecycle/rtmpose_convert_tensorrt.py:9`, `E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:183`).
- IMPLEMENTED: Pose confidence/quality/smoothing/jitter logic exists in artifact generation and temporal utilities (`E:/grad_project/backend/apps/video_analysis/tasks.py:749`, `E:/grad_project/backend/apps/video_analysis/services/pose_quality.py:31`, `E:/grad_project/backend/apps/pipeline/services/rtmpose_pipeline.py:92`).

2. Audit Scope
- Scope constrained to RTMPose + ROI/crop mapping + Triton/ONNX/TensorRT execution path + confidence/smoothing/jitter + artifact flow.
- Evidence sources (code/tests only):
- `E:/grad_project/backend/apps/pipeline/services/pose_runtime.py`
- `E:/grad_project/backend/apps/pipeline/services/rtmpose_pipeline.py`
- `E:/grad_project/backend/apps/pipeline/services/triton_client.py`
- `E:/grad_project/backend/apps/pipeline/services/model_route_service.py`
- `E:/grad_project/backend/apps/video_analysis/tasks.py`
- `E:/grad_project/backend/apps/video_analysis/services/pose_quality.py`
- `E:/grad_project/backend/apps/video_analysis/views.py`
- `E:/grad_project/backend/tests/system/test_rtmpose_variable_resolution_roi_mapping.py`
- `E:/grad_project/backend/tests/system/test_pose_jitter_reduction_benchmark.py`
- `E:/grad_project/frontend/src/components/VideoPlayer/OverlayCanvas.tsx`

3. Implemented (Evidence-Backed)
- IMPLEMENTED: Pose runtime consumes detections, extracts bbox (`bounding_box|bbox|xyxy`), crops per ROI, and runs inference (`E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:43`, `E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:50`, `E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:59`).
- IMPLEMENTED: ROI clamping and crop tensor preparation (BGR->RGB, normalize, CHW, batch dim) (`E:/grad_project/backend/apps/pipeline/services/rtmpose_pipeline.py:42`, `E:/grad_project/backend/apps/pipeline/services/rtmpose_pipeline.py:63`, `E:/grad_project/backend/apps/pipeline/services/rtmpose_pipeline.py:65`).
- IMPLEMENTED: Remap from crop space back to source coordinates via stored transform (`E:/grad_project/backend/apps/pipeline/services/rtmpose_pipeline.py:82`, `E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:76`).
- IMPLEMENTED: Triton route for `pose_estimation -> rtmpose_model:v2` (`E:/grad_project/backend/apps/pipeline/services/model_route_service.py:33`).
- IMPLEMENTED: Triton payload specialization for pose uses input name `input` and outputs `simcc_x/simcc_y` (`E:/grad_project/backend/apps/pipeline/services/triton_client.py:443`, `E:/grad_project/backend/apps/pipeline/services/triton_client.py:449`, `E:/grad_project/backend/apps/pipeline/services/triton_client.py:450`).
- IMPLEMENTED: SimCC decode computes argmax bins to `(x,y)` and confidence scores (`E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:240`, `E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:294`, `E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:300`).
- IMPLEMENTED: Upload pipeline runs pose estimation and persists `pose_results.json` and pose quality artifacts (`E:/grad_project/backend/apps/video_analysis/tasks.py:2792`, `E:/grad_project/backend/apps/video_analysis/tasks.py:2897`, `E:/grad_project/backend/apps/video_analysis/tasks.py:2898`).
- IMPLEMENTED: Live path emits pose records in overlay payload and can persist pose artifact (`E:/grad_project/backend/apps/video_analysis/tasks.py:3757`, `E:/grad_project/backend/apps/video_analysis/tasks.py:3772`, `E:/grad_project/backend/apps/video_analysis/tasks.py:3955`).
- IMPLEMENTED: Frame API attaches pose records with backward-compatible keys `pose_records` and `poses` (`E:/grad_project/backend/apps/video_analysis/views.py:559`, `E:/grad_project/backend/apps/video_analysis/views.py:565`, `E:/grad_project/backend/apps/video_analysis/views.py:566`).
- IMPLEMENTED: Frontend overlay prefers `display_keypoints` and falls back to `keypoints` (`E:/grad_project/frontend/src/components/VideoPlayer/OverlayCanvas.tsx:64`, `E:/grad_project/frontend/src/components/VideoPlayer/OverlayCanvas.tsx:141`).

4. Partially Implemented (Evidence-Backed)
- PARTIAL: ONNX/OpenVINO/TensorRT artifact declarations/conversion are present, but runtime pose inference path shown in code is Triton orchestrator-centric (`E:/grad_project/backend/apps/pipeline/model_lifecycle/rtmpose_manifest.py:17`, `E:/grad_project/backend/apps/pipeline/model_lifecycle/rtmpose_convert_tensorrt.py:9`, `E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:183`).
- PARTIAL: Batch path exists with per-response failure raising; partial-success handling within one batch is not evidenced (`E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:135`, `E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:161`).
- PARTIAL: Temporal smoothing is implemented for display and quality, but a unified persisted post-smoothing canonical keypoint stream is not evidenced (raw `keypoints` plus `display_keypoints` coexist) (`E:/grad_project/backend/apps/video_analysis/services/pose_quality.py:101`, `E:/grad_project/backend/apps/video_analysis/tasks.py:776`, `E:/grad_project/backend/apps/video_analysis/tasks.py:779`).

5. Missing (Evidence-Backed)
- MISSING: Direct local ONNX Runtime execution path for RTMPose in active `PoseRuntime` is not evidenced (`E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:114`, `E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:183`).
- MISSING: Direct local OpenVINO execution path for RTMPose in active `PoseRuntime` is not evidenced (`E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:114`, `E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:183`).
- MISSING: Direct local TensorRT execution path for RTMPose in active `PoseRuntime` is not evidenced (`E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:114`, `E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:183`).

6. Architecturally Incorrect / Inconsistent
- INCONSISTENT: `PoseRuntime.input_size` is assigned `(height,width)` (`E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:36`), while `decode_simcc` interprets tuple as `(dst_h,dst_w)` then uses `x_scale <- dst_w`, `y_scale <- dst_h` (`E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:286`, `E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:291`). This is internally consistent if config values are aligned, but tuple semantic ambiguity remains.
- INCONSISTENT: Generic Triton adapter config template emits fixed dims `[640,640,3]` and generic output `[1]`, which does not match RTMPose SimCC IO schema used by runtime client (`E:/grad_project/backend/apps/pipeline/model_lifecycle/triton_adapter.py:28`, `E:/grad_project/backend/apps/pipeline/model_lifecycle/triton_adapter.py:29`, `E:/grad_project/backend/apps/pipeline/services/triton_client.py:449`, `E:/grad_project/backend/apps/pipeline/services/triton_client.py:450`).

7. Scientifically Weak / Unvalidated
- PARTIAL: Jitter reduction benchmark test exists and asserts >=30% reduction, but repo-evidenced deployment calibration/validation across production data distributions is not evidenced (`E:/grad_project/backend/tests/system/test_pose_jitter_reduction_benchmark.py:8`, `E:/grad_project/backend/tests/system/test_pose_jitter_reduction_benchmark.py:16`).
- NOT EVIDENCED: Uncertainty calibration for pose confidence (e.g., calibrated confidence reliability curves) is not evidenced in code/tests within scoped artifacts.

8. Bottlenecks
- PARTIAL: Fallback path can degrade to per-crop serial inference when batch size mismatch/error occurs (`E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:73`, `E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:87`).
- PARTIAL: Upload path reads frames from disk (`predict/input_frames/*.jpg`) before pose inference; this introduces IO overhead (`E:/grad_project/backend/apps/video_analysis/tasks.py:2806`, `E:/grad_project/backend/apps/video_analysis/tasks.py:2807`).
- IMPLEMENTED: Runtime includes stride and per-frame person budget controls to bound load (`E:/grad_project/backend/apps/video_analysis/tasks.py:2797`, `E:/grad_project/backend/apps/video_analysis/tasks.py:2798`, `E:/grad_project/backend/apps/video_analysis/tasks.py:2799`).

9. Production Readiness Assessment
- IMPLEMENTED: Runtime records pose provider/device and latency summaries into job/session metadata (`E:/grad_project/backend/apps/video_analysis/tasks.py:2916`, `E:/grad_project/backend/apps/video_analysis/tasks.py:2917`, `E:/grad_project/backend/apps/video_analysis/tasks.py:3950`).
- IMPLEMENTED: Confidence diagnostics and quality summaries are persisted as artifacts (`E:/grad_project/backend/apps/video_analysis/tasks.py:2884`, `E:/grad_project/backend/apps/video_analysis/tasks.py:2895`, `E:/grad_project/backend/apps/video_analysis/tasks.py:2899`).
- PARTIAL: End-to-end production readiness for non-Triton pose inference backends is not evidenced in active pose runtime code path (`E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:183`).

10. Equations Actually Used
- IMPLEMENTED: EMA smoothing equation in code: `out[t] = alpha * out[t] + (1 - alpha) * out[t-1]` (`E:/grad_project/backend/apps/pipeline/services/rtmpose_pipeline.py:92`, `E:/grad_project/backend/apps/pipeline/services/rtmpose_pipeline.py:96`).
- IMPLEMENTED: Jitter metric uses mean L2 displacement over frame-to-frame diffs (`E:/grad_project/backend/apps/pipeline/services/rtmpose_pipeline.py:100`, `E:/grad_project/backend/apps/pipeline/services/rtmpose_pipeline.py:103`).
- IMPLEMENTED: Pose score uses arithmetic mean of keypoint confidences (`E:/grad_project/backend/apps/video_analysis/tasks.py:763`, `E:/grad_project/backend/apps/video_analysis/tasks.py:770`).
- IMPLEMENTED: SimCC coordinate mapping: `(argmax_bin + 0.5) * scale` and score from x/y peak aggregation (`E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:298`, `E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:300`).

11. Figures/Images Candidates from Repo
- IMPLEMENTED: `pose_results.json` artifact generated by upload/live jobs is available as candidate evidence artifact (runtime-produced JSON, not static figure) (`E:/grad_project/backend/apps/video_analysis/tasks.py:697`, `E:/grad_project/backend/apps/video_analysis/tasks.py:3955`).
- IMPLEMENTED: Additional candidate artifacts: `pose_per_student.json`, `pose_quality_summary.json` (`E:/grad_project/backend/apps/video_analysis/tasks.py:2898`, `E:/grad_project/backend/apps/video_analysis/tasks.py:2899`).
- NOT EVIDENCED: Pre-existing committed image figures (PNG/JPG/SVG) specifically for RTMPose temporal artifacts were not evidenced in scoped audit paths.

12. Contradictions (Code vs Docs vs Tests)
- INCONSISTENT: Tests validate ROI mapping contract (`E:/grad_project/backend/tests/system/test_rtmpose_variable_resolution_roi_mapping.py:8`), while runtime tuple semantics for input size remain ambiguous as noted in Section 6 (`E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:36`, `E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:286`).
- INCONSISTENT: Triton adapter generic config template IO does not match pose runtime client schema (`E:/grad_project/backend/apps/pipeline/model_lifecycle/triton_adapter.py:28`, `E:/grad_project/backend/apps/pipeline/services/triton_client.py:449`).

13. Open Questions / Unknowns
- NOT EVIDENCED: Which exact Triton model repository `config.pbtxt` for `rtmpose_model` is currently deployed in production from this repo snapshot.
- NOT EVIDENCED: Whether `POSE_QUALITY_SMOOTHING_WINDOW`, `POSE_DRAW_CONFIDENCE_THRESHOLD`, and `POSE_FRAME_BUDGET_MS` production values match benchmarked targets at deploy time (`E:/grad_project/backend/apps/video_analysis/tasks.py:2798`, `E:/grad_project/backend/apps/video_analysis/tasks.py:2874`, `E:/grad_project/backend/apps/video_analysis/tasks.py:2883`).
- NOT EVIDENCED: Quantitative drift/accuracy impact of fallback serial-per-crop path under sustained load.

14. Subsystem Coverage Checklist
- RTMPose runtime wiring: IMPLEMENTED (`E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:43`).
- ROI/crop preprocessing + source remap: IMPLEMENTED (`E:/grad_project/backend/apps/pipeline/services/rtmpose_pipeline.py:51`, `E:/grad_project/backend/apps/pipeline/services/rtmpose_pipeline.py:82`).
- Triton request/response pose schema: IMPLEMENTED (`E:/grad_project/backend/apps/pipeline/services/triton_client.py:443`).
- Route binding `pose_estimation -> rtmpose_model`: IMPLEMENTED (`E:/grad_project/backend/apps/pipeline/services/model_route_service.py:33`).
- ONNX/TensorRT artifact lifecycle: PARTIAL (`E:/grad_project/backend/apps/pipeline/model_lifecycle/rtmpose_manifest.py:17`).
- Local ONNX/OpenVINO/TensorRT pose runtime execution: MISSING (`E:/grad_project/backend/apps/pipeline/services/pose_runtime.py:183`).
- Confidence scoring + diagnostics: IMPLEMENTED (`E:/grad_project/backend/apps/video_analysis/tasks.py:749`, `E:/grad_project/backend/apps/video_analysis/tasks.py:783`).
- Temporal smoothing/jitter utilities: IMPLEMENTED (`E:/grad_project/backend/apps/pipeline/services/rtmpose_pipeline.py:92`, `E:/grad_project/backend/apps/pipeline/services/rtmpose_pipeline.py:100`).
- Artifact persistence and metadata propagation: IMPLEMENTED (`E:/grad_project/backend/apps/video_analysis/tasks.py:2897`, `E:/grad_project/backend/apps/video_analysis/tasks.py:2911`).
- API/frontend pose payload consumption: IMPLEMENTED (`E:/grad_project/backend/apps/video_analysis/views.py:565`, `E:/grad_project/frontend/src/components/VideoPlayer/OverlayCanvas.tsx:64`).
