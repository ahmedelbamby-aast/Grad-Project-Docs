# Production Inventory: XAI And Anomaly Planning Baseline

**Captured**: 2026-06-08T21:19:20+03:00
**Authority**: Native Linux production server, PostgreSQL, active offline Triton
profile
**Git SHA**: `4ee4cc8c54a0a505263933a9fb1cb497871e4b05`
**Branch**: `014-yoloe-scene-srvl`
**Status**: Point-in-time planning evidence; refresh before Cycle 015.0

This file records what was observed while planning. It is not a timeless
production claim and is not acceptance evidence for any future XAI cycle.

## Active Runtime Configuration

```text
GAZE_HORIZONTAL_HEAD_VARIANT=coco80
LPM_ENABLED=0
MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble
OFFLINE_DETECT_EVERY_N_FRAMES=5
POSE_KINEMATICS_ENABLED=1
SRVL_ENABLED=1
TRITON_BEHAVIOR_TOP_K_ENABLED=0
TRITON_BEHAVIOR_TOP_K_VALUE=100
TRITON_CROP_BEHAVIOR_INPUT_SIZE=320
TRITON_EXECUTION_MODE=offline
TRITON_GRPC_ENABLED=1
TRITON_LOAD_MODEL=person_detector,posture_model,gaze_horizontal_model,gaze_vertical_model,gaze_depth_model,rtmpose_model,behavior_ensemble,osnet_ain_x1_0,yoloe_scene_seg
TRITON_OFFLINE_FRAME_STRIDE=1
TRITON_PROTOCOL_PREFERENCE=grpc
VIDEO_UPLOAD_DEFAULT_PIPELINE_MODE=crop_frame
YOLOE_SCENE_ENABLED=1
```

## Triton Repository State

### Ready And Loaded

| Model | Version | State |
|---|---:|---|
| `behavior_ensemble` | 1 | READY |
| `gaze_depth_model` | 1 | READY |
| `gaze_horizontal_model` | 1 | READY |
| `gaze_vertical_model` | 1 | READY |
| `osnet_ain_x1_0` | 1 | READY |
| `person_detector` | 1 | READY |
| `posture_model` | 1 | READY |
| `rtmpose_model` | 1 | READY |
| `yoloe_scene_seg` | 1 | READY |

### Present But Not Ready/Loaded

`behavior_ensemble_gaze2`, `behavior_ensemble_gaze_slice`,
`behavior_ensemble_gaze_slice_topk`, `gaze_depth_topk_model`,
`gaze_horizontal_gaze2_model`, `gaze_horizontal_slice_adapter`,
`gaze_horizontal_slice_model`, `gaze_horizontal_slice_topk_model`,
`gaze_vertical_topk_model`, and `posture_topk_model`.

## PostgreSQL Record Counts

| Record type | Count |
|---|---:|
| `VideoAnalysisJob` | 193 |
| `Frame` | 363655 |
| `Detection` | 5786316 |
| `BoundingBox` | 5786316 |
| `StudentTrack` | 4910 |
| `FrameEmbedding` | 4886560 |
| `PoseKinematicsRecord` | 463960 |
| `SceneFrameSummary` | 24759 |
| `SceneObjectObservation` | 514216 |
| `SpatialRelationFrame` | 24759 |
| `TelemetryModelCall` | 528156 |
| `SemanticPoseState` | 0 |
| `BehavioralState` | 0 |
| `BehavioralEpisode` | 0 |
| `InteractionEdge` | 0 |
| `AnomalyPrimitiveEvent` | 0 |
| `BehaviorFeatureWindow` | 0 |
| `DecisionLineageRecord` | 0 |
| `AnomalyEvent` | 0 |
| `BSILBaselineSnapshot` | 0 |
| `BSILThresholdShift` | 0 |
| `BSILReviewLabelRecord` | 0 |

### Interpretation

The schemas and services for BSIL/anomaly evidence exist, but production had no
rows in the semantic, temporal, anomaly, baseline, threshold, or decision
lineage tables at capture time. Cycle 015.0 must prove activation, population,
and reconciliation before later cycles can claim anomaly-score or XAI maturity.

## Observed Persisted Model Names

### Bounding Boxes

`attention_tracking`, `hand_raising`, `person_detection`, and
`sitting_standing`.

### Historical Model-Call Telemetry

`behavior_ensemble`, `behavior_ensemble_gaze_slice`,
`behavior_ensemble_gaze_slice_topk`, `gaze_depth_model`,
`gaze_horizontal_model`, `gaze_horizontal_slice_adapter`,
`gaze_vertical_model`, `person_detector`, `posture_model`, `rtmpose_model`,
and `yoloe_scene_seg`.

## Latest Job At Capture

| Field | Value |
|---|---|
| Job | `62f99ecd-41d8-4c7a-826d-4b2b443752b3` |
| Status | `completed` |
| Pipeline mode | `crop_frame` |
| Frames | `261/261` |
| Created | `2026-06-08T17:22:06.766138+00:00` |

## Instrumentation Readiness

The production checker
`tools/prod/prod_check_instrumentation_tools.py` returned:

```text
status=ok
core_missing=[]
optional_missing=[]
```

Observed tooling included Python 3.12.3, `uv 0.11.19`, `psutil 7.2.2`,
`py-spy 0.4.2`, `pyinstrument 5.1.2`, NVIDIA Nsight Systems 2024.6.2,
PostgreSQL client 16.14, Redis CLI 7.0.15, and an RTX 5090 with 32607 MiB.

## Route Drift Finding

Repository benchmark documentation records accepted exact-slice and Top-K
routes, while the active production environment at capture time used:

```text
GAZE_HORIZONTAL_HEAD_VARIANT=coco80
MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble
TRITON_BEHAVIOR_TOP_K_ENABLED=0
```

This is not automatically a defect, but it proves that documentation alone is
not runtime authority. Cycle 015.0 must capture an immutable runtime route
snapshot and reconcile desired versus actual route before benchmark decisions.

## Source References

- `backend/apps/pipeline/services/model_route_service.py`
- `backend/apps/video_analysis/models.py`
- `backend/apps/behavior/models.py`
- `backend/apps/anomalies/models.py`
- `backend/apps/telemetry/models.py`
- `tools/prod/prod_check_instrumentation_tools.py`
- `docs/production_inference_benchmark.md`
