# Pipeline

**Last updated:** 2026-05-25

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

## Human Pose Kinematics Services

Feature 013 adds deterministic post-RTMPose services under
`backend/apps/pipeline/services/`:

- `pose_kinematics.py`: pure keypoint-to-evidence functions for quality,
  anchors, geometry, joint angles, orientation, posture, gesture,
  physical-validity, temporal motion fields, and fusion signals.
- `pose_kinematics_config.py`: `.env`/Django-settings backed configuration
  loader and override-gate evaluator. Override thresholds must enter through
  `POSE_KINEMATICS_*` settings.
- `pose_kinematics_history.py`: bounded per-source, per-track history capped
  by `POSE_KINEMATICS_HISTORY_SECONDS` and
  `POSE_KINEMATICS_HISTORY_MAX_SAMPLES`.
- `logical_path_matrix.py`: exposes `with_pose_kinematics_context(...)` so
  pose fusion context can be carried beside LPM inputs without mutating the
  original behavior/gaze probabilities.

The kinematics layer does not add a model and does not replace RTMPose. It
consumes RTMPose keypoints and emits evidence for downstream persistence,
telemetry, API/export, and review.

## Runtime Contract (Hybrid Validation)

- Runtime routing is model-identity based, not runtime-backend-only:
  - each model task resolves to a dedicated `worker_identity`
  - each route records `runtime_backend`, `effective_device`, `selected_authority`, and worker proof
- Supported stress-lane backends are `triton` and `openvino`; assignments are persisted per run/job.
- Telemetry parity for comparable runs requires identical `telemetry_mode_id`.

## Buffering Contract (Tiered + Bounded)

- Buffering is application-controlled and observable through explicit keys:
  - `buffer.ram.max_frames`
  - `buffer.ram.max_bytes`
  - `buffer.spill.max_frames`
  - `buffer.spill.max_bytes`
  - `buffer.spill.path`
  - `buffer.overflow_policy` (`block` | `drop_oldest` | `drop_newest`)
- Tiering uses a bounded RAM queue plus bounded disk spill queue with frame-order preservation.
- Overflow outcomes are deterministic and tracked as decisions:
  - `queued_ram`, `spilled_disk`, `blocked`, `dropped`, `drained`

## Telemetry Contract (Normalized Persistence)

- Runtime ingestion persists normalized events for dashboard and MCP projections from shared storage, not from direct collectors.
- Buffer pressure and worker isolation telemetry include queue/spill depths, overflow policy, worker health/backlog, and runtime assignments.
- Projection outputs include summary/timeline/artifact readiness and comparison support keyed by run/job/session context.

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

## Maturity Closure Notes
- Runtime mode and evidence contracts are enforced for wave-driven closure.

