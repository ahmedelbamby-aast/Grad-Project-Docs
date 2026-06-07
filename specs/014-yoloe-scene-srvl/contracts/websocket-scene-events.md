# Contract: WebSocket Scene Events

**Version**: `scene-ws.v1`  
**Feature**: `014-yoloe-scene-srvl`

## Event: scene.segmentation.summary

Emitted when a scene frame summary is ready or degraded.

```json
{
  "type": "scene.segmentation.summary",
  "schema_version": "scene-ws.v1",
  "job_id": "uuid",
  "frame_number": 1024,
  "timestamp_ms": 32000,
  "scene_status": "valid",
  "object_count": 24,
  "person_count": 18,
  "non_roi_object_count": 6,
  "confidence_summary": {
    "min": 0.31,
    "mean": 0.82,
    "threshold_env_key": "YOLOE_SCENE_CONFIDENCE_THRESHOLD",
    "threshold_value": 0.25
  },
  "artifact_manifest_ref": "artifact:manifest:...",
  "trace_ref": "artifact:trace:..."
}
```

## Event: scene.mismatch

Emitted when YOLOE people count and student-plus-teacher count differ.

```json
{
  "type": "scene.mismatch",
  "schema_version": "scene-ws.v1",
  "job_id": "uuid",
  "frame_number": 1024,
  "timestamp_ms": 32000,
  "yoloe_person_count": 19,
  "student_count": 17,
  "teacher_count": 1,
  "matched_existing_count": 18,
  "false_alarm_count": 0,
  "missing_candidate_count": 1,
  "unresolved_count": 0,
  "repass_enabled": false,
  "threshold_snapshot": {
    "YOLOE_SCENE_CONFIDENCE_THRESHOLD": 0.25,
    "YOLOE_SCENE_RECOVERY_MIN_SCORE": 0.7
  },
  "match_manifest_ref": "artifact:mismatch:..."
}
```

## Event: scene.contradiction

Emitted for append-only non-ROI overlap evidence.

```json
{
  "type": "scene.contradiction",
  "schema_version": "scene-ws.v1",
  "job_id": "uuid",
  "frame_number": 1024,
  "timestamp_ms": 32000,
  "event_id": "uuid",
  "event_type": "non_roi_overlap",
  "severity": "warning",
  "action": "flag_only",
  "overlap_ratio": 0.62,
  "source_confidence": 0.91,
  "decision_thresholds": {
    "YOLOE_SCENE_CONTRADICTION_OVERLAP": 0.4,
    "YOLOE_SCENE_CONTRADICTION_MIN_CONFIDENCE": 0.25
  }
}
```

## Event: scene.visualization.status

Emitted when map buffers, snapshots, or MP4 rendering change state.

```json
{
  "type": "scene.visualization.status",
  "schema_version": "scene-ws.v1",
  "job_id": "uuid",
  "frame_number": 1024,
  "timestamp_ms": 32000,
  "renderer": "pixijs_webgl",
  "status": "rendered",
  "dropped_visualization_frames": 2,
  "distance_heatmap_ref": "artifact:map:...",
  "correlation_map_ref": "artifact:map:..."
}
```

## Delivery Rules

- Events are optional and backward-compatible.
- Event delivery must not block analytics persistence.
- Slow frontend consumers receive latest usable state and drop counters.
- Large artifact payloads are never sent inline over WebSocket.
- Events include confidence and threshold evidence when a decision used it.
- Producers must read operational values from `.env`/settings, not hardcoded
  constants in WebSocket or frontend code.
