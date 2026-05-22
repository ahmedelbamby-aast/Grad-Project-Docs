# Pipeline

Pipeline services for student detection, student cropping, behavior classification, and model lifecycle utilities.

## Public API

- `PersonDetector`
- `FrameCropper`
- `PredictionMerger`
- `PipelineService`
- `scan_model_inventory()`
- `detect_runtime_capabilities()`
- `ExportOrchestrator`
- `BenchmarkRunner`
- `DeploymentMatrix`
- `TritonAdapter`

## Dependencies

- `ultralytics`
- `opencv-python`
- `numpy`
- `pydantic`

## Usage

```python
from apps.pipeline.services import PipelineService

service = PipelineService()
status = service.get_model_lifecycle_status()
```

## RTMPose Source-Frame Integration Flow

For camera feeds of any resolution, the general inference pipeline and pose-estimation pipeline must use this fixed sequence:

1. Detect person boxes on the original source frame.
2. Crop each person ROI from the original frame.
3. Resize each ROI to RTMPose model input size (`256x192` by default, or configured fixed profile size).
4. Run RTMPose on resized ROIs.
5. Map predicted keypoints from ROI coordinates back to original source-frame coordinates.

Notes:
- Legacy posture models in `standing_sitting` remain valid for detection-pipeline behavior and are not removed by RTMPose integration.
- RTMPose path is an additional pose-estimation branch that feeds downstream behavior fusion.
# Boundary Summary

| Field | Value |
| --- | --- |
| Purpose | Model loading, inference clients, and route decisions |
| Responsibilities | Run model inference; publish prediction and health states |
| Public inputs | Frames; crops; model requests |
| Public outputs | Prediction results; provider health states |
| Consumers | Detections, video analysis, tracking |
| Dependencies | Model repository, Triton, OpenVINO, ONNXRuntime |
| Failure behavior | Timeouts, fallback, or degraded inference status are returned |
