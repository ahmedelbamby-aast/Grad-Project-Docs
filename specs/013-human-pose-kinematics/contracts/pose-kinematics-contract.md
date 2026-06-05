# Contract: Pose Kinematics v1

**Last updated:** 2026-06-05

## Contract Version

`pose_kinematics.v1`

This contract governs compact PostgreSQL summaries, artifact/export payloads,
telemetry counters, and fusion override evidence for the Human Pose Kinematics
Layer.

## Compact Evidence Summary

Required for every pose-eligible tracked student frame.

```json
{
  "schema_version": "pose_kinematics.v1",
  "scope": {
    "job_id": "uuid-or-null",
    "session_id": "session-or-null",
    "camera_id": "camera-or-null",
    "frame_index": 1204,
    "timestamp_ms": 40133,
    "tracking_id": 7
  },
  "state": "valid",
  "pose_quality": "good",
  "quality_metrics": {
    "confidence_mean": 0.87,
    "confidence_min": 0.42,
    "visible_keypoints_count": 14,
    "missing_keypoints": ["left_ankle", "right_ankle"],
    "low_confidence_keypoints": ["left_wrist"],
    "reasons": []
  },
  "body_anchors": {
    "head_center": [421.0, 211.0],
    "neck_center": [423.0, 245.0],
    "shoulder_center": [424.0, 261.0],
    "hip_center": [430.0, 352.0],
    "torso_center": [427.0, 306.0],
    "body_center": [426.0, 318.0]
  },
  "body_geometry": {
    "person_height_px": 312.5,
    "torso_height_px": 86.2,
    "shoulder_width_px": 74.1,
    "hip_width_px": 62.4,
    "torso_length_px": 91.3,
    "body_scale": 1.0
  },
  "orientation": {
    "torso_yaw": "front_left",
    "torso_pitch": "upright",
    "torso_roll_deg": -4.2,
    "head_yaw": "right",
    "head_pitch": "down",
    "head_roll_deg": 3.5,
    "head_torso_alignment": "misaligned",
    "confidence": 0.81
  },
  "posture": {
    "class": "sitting",
    "confidence": 0.89,
    "supporting_features": {
      "hip_knee_angle_mean": 86.4,
      "torso_vertical_ratio": 0.72,
      "bbox_aspect_ratio": 1.18
    }
  },
  "gesture": {
    "hand_raise_class": "right_hand",
    "confidence": 0.93,
    "left_hand_raised": false,
    "right_hand_raised": true
  },
  "attention_pose": {
    "class": "possibly_attentive",
    "body_direction": "front",
    "head_direction": "down_right",
    "attention_support_score": 0.74
  },
  "motion": {
    "history_window_seconds": 5.0,
    "history_sample_count": 3,
    "history_bound_violation_count": 0,
    "pose_stability_score": 0.91,
    "sudden_jump_detected": false
  },
  "ik": {
    "estimated_keypoints_used": true,
    "correction_applied": true,
    "pose_physical_validity": 0.78,
    "constraint_violations": []
  },
  "lineage": {
    "feature_version": "human_pose_kinematics.v1",
    "config_digest": "sha256:...",
    "rtmpose_model": "rtmpose_model:v1",
    "artifact_ref": "sha256:..."
  }
}
```

## Artifact Entry

Full arrays are artifact/export data, not required in compact DB rows.

```json
{
  "artifact_version": "pose_kinematics_artifact.v1",
  "scope": {
    "job_id": "uuid-or-null",
    "session_id": "session-or-null",
    "camera_id": "camera-or-null",
    "frame_index": 1204,
    "tracking_id": 7
  },
  "raw_keypoints": [[421.0, 211.0, 0.94]],
  "normalized_keypoints": [[0.12, -0.31, 0.94]],
  "corrected_keypoints": [[421.0, 211.0, 0.94]],
  "smoothed_keypoints": [[420.4, 211.3, 0.92]],
  "correction_notes": ["right_wrist_smoothed_using_history"],
  "source_pose": {
    "bbox_xyxy": [302.0, 120.0, 540.0, 612.0],
    "model_name": "rtmpose_model",
    "model_version": "v1"
  }
}
```

## Fusion Override Event

```json
{
  "schema_version": "pose_kinematics_override.v1",
  "event_id": "pose-override-job-frame-track-model",
  "scope": {
    "job_id": "uuid-or-null",
    "session_id": "session-or-null",
    "camera_id": "camera-or-null",
    "frame_index": 1204,
    "tracking_id": 7
  },
  "target_model": "attention_tracking",
  "original_prediction": {
    "label": "attention_right",
    "confidence": 0.62
  },
  "pose_signal": {
    "label": "head_yaw_left",
    "confidence": 0.84,
    "basis": "corrected_smoothed_keypoints"
  },
  "gate": {
    "pose_confidence": 0.84,
    "target_confidence": 0.62,
    "override_margin": 0.15,
    "min_supporting_frames": 3,
    "history_window_seconds": 5.0,
    "history_support_count": 4
  },
  "decision": "applied",
  "spike_or_contradiction": "persistent_contradiction",
  "root_cause_category": "model_disagreement",
  "reasons": [
    "pose_confidence_margin_satisfied",
    "history_support_satisfied",
    "same_student_not_spike"
  ]
}
```

## Configuration Contract

These keys must be available through `.env` and Django settings. Service logic
must read active values from configuration and must not hardcode them.

| Key | Default | Required behavior |
| --- | --- | --- |
| `POSE_KINEMATICS_ENABLED` | `1` | Master feature flag, default-enabled by initial production exception. |
| `POSE_KINEMATICS_HISTORY_SECONDS` | `5` | Maximum same-track history window. |
| `POSE_KINEMATICS_HISTORY_MAX_SAMPLES` | implementation bounded default | Maximum samples retained per track. |
| `POSE_KINEMATICS_OVERRIDE_MARGIN` | `0.15` | Pose confidence must exceed target by this margin. |
| `POSE_KINEMATICS_OVERRIDE_MIN_FRAMES` | `3` | Supporting same-track frames required inside the history window. |
| `POSE_KINEMATICS_MIN_KEYPOINT_CONFIDENCE` | implementation bounded default | Joint visibility threshold. |
| `POSE_KINEMATICS_ARTIFACTS_ENABLED` | `1` | Enables full array artifacts/exports. |
| `POSE_KINEMATICS_TELEMETRY_ENABLED` | `1` | Enables kinematics metrics/counters. |

## Telemetry Contract

Benchmark and live watcher surfaces must expose:

| Metric | Meaning |
| --- | --- |
| `pose_kinematics_records_total` | Total summaries emitted. |
| `pose_kinematics_valid_total` | Valid summaries emitted. |
| `pose_kinematics_unavailable_total` | Unavailable summaries emitted. |
| `pose_kinematics_degraded_total` | Degraded summaries emitted. |
| `pose_kinematics_contradiction_total` | Contradiction states emitted. |
| `pose_kinematics_override_applied_total` | Overrides applied. |
| `pose_kinematics_override_blocked_total` | Overrides blocked by config/history/confidence gate. |
| `pose_kinematics_ms_total` | Total kinematics wall time. |
| `pose_kinematics_ms_per_frame` | Mean wall time per processed frame. |
| `pose_kinematics_ms_per_subject` | Mean wall time per pose-eligible subject. |
| `pose_kinematics_history_bound_violations` | Count of history windows exceeding active config; must be zero. |

## Consumer State Contract

Consumers must treat the following as distinct states:

- `unknown`: measurement not established.
- `unavailable`: pose evidence could not be produced.
- `degraded`: evidence exists but a required authority failed.
- `valid`: evidence satisfies the contract.
- `contradiction`: pose conflicts with another model and does not meet override gate.
- `override_applied`: configured high-confidence override applied with lineage.
