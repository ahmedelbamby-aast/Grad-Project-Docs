# RTMPose Source-Frame Contract

Implemented integration flow (any camera frame resolution):

1. Detect person boxes on source frame (`PipelineService` detections).
2. Crop each person ROI from source frame (`crop_resize_people`).
3. Resize each ROI to fixed RTMPose input (`rtmpose_input_height`, `rtmpose_input_width`).
4. Run RTMPose runtime adapter (`PoseRuntime.infer_from_detections` boundary).
5. Map keypoints back to source-frame coordinates (`map_keypoints_to_source`).

Code paths:
- `backend/apps/pipeline/pipeline_service.py`
- `backend/apps/pipeline/services/pose_runtime.py`
- `backend/apps/pipeline/services/rtmpose_pipeline.py`
- `backend/apps/pipeline/config.py`

Regression test:
- `backend/tests/system/test_rtmpose_variable_resolution_roi_mapping.py` (PASS)

