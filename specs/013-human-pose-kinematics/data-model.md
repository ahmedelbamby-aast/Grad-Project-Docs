# Data Model: Human Pose Kinematics Layer

**Last updated:** 2026-06-04

## Entity: Pose Observation

One raw RTMPose output for a tracked student in a frame.

| Field | Type | Notes |
| --- | --- | --- |
| `job_id` | UUID/null | Offline job scope when present. |
| `session_id` | string/null | Live/offline session scope. |
| `camera_id` | string/null | Required for live streams. |
| `frame_index` | integer | Source frame index. |
| `timestamp_ms` | integer | Source or derived frame timestamp. |
| `tracking_id` | integer/string | Local track identity from the current pipeline. |
| `bbox_xyxy` | number[4] | Source-frame ROI used for RTMPose. |
| `raw_keypoints` | artifact ref | Full `[x, y, confidence]` array. |
| `keypoint_scores` | artifact ref | Per-joint confidence values. |
| `pose_source` | object | RTMPose model route/version/config/runtime identity. |

**Validation rules**:

- Keypoint arrays must carry the expected COCO-style joint order or an explicit
  unavailable reason.
- A missing pose observation produces a kinematics state of `unavailable` or
  `unknown`; it must not be treated as negative behavior evidence.

## Entity: Kinematics Evidence Record

Compact relational summary for one pose-eligible `(scope, frame, tracking_id)`.

| Field | Type | Notes |
| --- | --- | --- |
| `record_id` | UUID | Primary key. |
| `scope` | object | `job_id`, `session_id`, `camera_id`. |
| `frame_index` | integer | Indexed for review/export. |
| `timestamp_ms` | integer | Source time where available. |
| `tracking_id` | integer/string | Local track id. |
| `schema_version` | string | Default `pose_kinematics.v1`. |
| `feature_version` | string | Code/config feature version. |
| `state` | enum | `unknown`, `unavailable`, `degraded`, `valid`, `contradiction`, `override_applied`. |
| `pose_quality` | enum | `excellent`, `good`, `weak`, `invalid`, `partial_body`, `occluded`. |
| `quality_metrics` | object | Mean/min confidence, visible count, missing/low-confidence joints. |
| `body_geometry` | object | Person height, torso height, shoulder/hip width, torso length, scale. |
| `body_anchors` | object | Head/neck/shoulder/hip/torso/body centers where available. |
| `joint_angles_deg` | object | Elbow, shoulder, hip, knee, neck/head/torso angles where available. |
| `limb_vectors` | object | Normalized vectors for torso, head, arms, legs where available. |
| `orientation` | object | Torso/head yaw, pitch, roll, and alignment. |
| `posture` | object | Sitting/standing/leaning class, confidence, supporting features. |
| `gesture` | object | Hand-raise class and per-hand supporting booleans. |
| `attention_pose` | object | Attention-support class, score, direction, reasons. |
| `motion` | object | Stability, velocities, sudden-motion/spike flags. |
| `ik` | object | Estimated/corrected flags, validity score, violations. |
| `artifact_ref` | string | Digest-addressed full evidence artifact. |
| `config_digest` | string | Digest of active kinematics config. |

**Validation rules**:

- Exactly one state is emitted for each pose-eligible track/frame pair.
- Compact fields may be `unknown`/`unavailable`, but must include a reason.
- `corrected_keypoints_available` and `smoothed_keypoints_available` are flags,
  not implicit replacements for raw keypoints.

## Entity: Pose Evidence Artifact

Versioned artifact containing full arrays and lineage.

| Field | Type | Notes |
| --- | --- | --- |
| `artifact_version` | string | Default `pose_kinematics_artifact.v1`. |
| `digest` | string | SHA-256 or equivalent content digest. |
| `records` | array | Per-frame/per-track full evidence entries. |
| `raw_keypoints` | array | Preserved raw keypoints. |
| `normalized_keypoints` | array | Scale-normalized view. |
| `corrected_keypoints` | array/null | Present only when correction is applied. |
| `smoothed_keypoints` | array/null | Present when bounded history exists. |
| `lineage` | object | Source video/stream, code SHA, runtime profile, model ids. |

**Validation rules**:

- The artifact must be immutable after publication.
- The DB summary must reference this artifact when full arrays are omitted from
  relational rows.

## Entity: Keypoint Quality Assessment

| Field | Type | Notes |
| --- | --- | --- |
| `confidence_mean` | float | Mean visible-joint score. |
| `confidence_min` | float | Minimum visible-joint score. |
| `visible_keypoints_count` | integer | Count above configured threshold. |
| `missing_keypoints` | string[] | Joint names not available. |
| `low_confidence_keypoints` | string[] | Joint names below threshold. |
| `quality_label` | enum | Quality state. |
| `reasons` | string[] | Machine-readable reasons. |

## Entity: Body Geometry Profile

| Field | Type | Notes |
| --- | --- | --- |
| `normalized_keypoints_ref` | artifact ref | Full normalized array. |
| `person_height_px` | float/null | From visible skeleton/bbox. |
| `torso_height_px` | float/null | Shoulder-to-hip estimate. |
| `shoulder_width_px` | float/null | Left/right shoulder distance. |
| `hip_width_px` | float/null | Left/right hip distance. |
| `torso_length_px` | float/null | Neck/shoulder to hip center. |
| `body_scale` | float/null | Normalization scale. |
| `anchors` | object | Head, neck, shoulder, hip, torso, body centers. |

## Entity: Orientation Assessment

| Field | Type | Notes |
| --- | --- | --- |
| `torso_yaw` | enum | `front`, `front_left`, `front_right`, `left`, `right`, `back`, `unknown`. |
| `torso_pitch` | enum | `upright`, `leaning`, `down`, `unknown`. |
| `torso_roll_deg` | float/null | Shoulder/hip line roll estimate. |
| `head_yaw` | enum | `left`, `right`, `center`, `unknown`. |
| `head_pitch` | enum | `up`, `down`, `forward`, `unknown`. |
| `head_roll_deg` | float/null | Eye/ear line roll estimate. |
| `head_torso_alignment` | enum | `aligned`, `misaligned`, `unknown`. |
| `confidence` | float | Combined orientation confidence. |
| `supporting_features` | object | Distances/angles used. |

## Entity: Posture And Gesture Evidence

| Field | Type | Notes |
| --- | --- | --- |
| `posture_class` | enum | `sitting`, `standing`, `leaning`, `crouching`, `partial_body`, `unknown`. |
| `posture_confidence` | float | Heuristic confidence. |
| `hand_raise_class` | enum | `none`, `left_hand`, `right_hand`, `both_hands`, `uncertain`. |
| `hand_raise_confidence` | float | Heuristic confidence. |
| `supporting_features` | object | Joint positions, angle checks, visibility checks. |

## Entity: Inverse-Kinematic Correction Evidence

| Field | Type | Notes |
| --- | --- | --- |
| `estimated_keypoints_used` | boolean | True when missing joints are estimated. |
| `correction_applied` | boolean | True when corrected view differs from raw. |
| `correction_confidence` | float | Lower than detected-joint confidence. |
| `pose_physical_validity` | float | Validity score. |
| `constraint_violations` | object[] | Type, joint/segment, severity, effect. |
| `correction_notes` | string[] | Machine-readable correction reasons. |

## Entity: Temporal Pose State

| Field | Type | Notes |
| --- | --- | --- |
| `history_window_seconds` | float | Actual window used, never above configured cap. |
| `history_sample_count` | integer | Same-track samples used. |
| `history_bound_violation_count` | integer | Should be zero in accepted runs. |
| `pose_stability_score` | float | Stability metric. |
| `motion_velocity` | object | Head/wrist/body velocities. |
| `sudden_jump_detected` | boolean | Spike flag. |
| `spike_or_contradiction` | enum | `spike`, `persistent_contradiction`, `unknown`, `not_applicable`. |

## Entity: Fusion Pose Signal

| Field | Type | Notes |
| --- | --- | --- |
| `fusion_state` | enum | `support`, `contradiction`, `unavailable`, `override_applied`. |
| `target_model` | string | Gaze/behavior model being supported or contradicted. |
| `target_prediction` | object | Original model output summary. |
| `pose_basis` | object | Corrected/smoothed/quality basis. |
| `confidence_margin` | float | Pose confidence minus target confidence. |
| `history_support_count` | integer | Same-track supporting frames. |
| `root_cause_category` | enum | Missing pose, low confidence, spike, model disagreement, impossible pose, identity uncertainty, config gate. |

## Entity: Fusion Override Event

| Field | Type | Notes |
| --- | --- | --- |
| `event_id` | UUID/string | Idempotency key. |
| `scope` | object | Job/session/camera/frame/track. |
| `target_model` | string | Overridden model. |
| `original_prediction` | object | Preserved source prediction. |
| `replacement_signal` | object | Pose-derived correction/fusion signal. |
| `pose_confidence` | float | Pose confidence used by gate. |
| `target_confidence` | float | Target model confidence. |
| `override_margin` | float | Active configured margin. |
| `min_supporting_frames` | integer | Active configured minimum. |
| `history_window_seconds` | float | Active configured window. |
| `history_support_count` | integer | Observed support count. |
| `decision` | enum | `applied`, `blocked`, `unavailable`. |
| `reasons` | string[] | Traceable reason codes. |

## Relationships

- `Pose Observation` feeds `Kinematics Evidence Record`.
- `Kinematics Evidence Record` references one `Pose Evidence Artifact`.
- `Temporal Pose State` is scoped to one track within one session/camera/job.
- `Fusion Pose Signal` consumes one kinematics record and one target model
  prediction.
- `Fusion Override Event` is emitted only when the configured gate evaluates a
  support/contradiction decision.

## State Transitions

```text
unknown -> valid
unknown -> unavailable
valid -> contradiction
contradiction -> override_applied
contradiction -> valid
valid -> degraded
degraded -> valid
unavailable -> valid
```

Rules:

- `override_applied` requires higher pose confidence, configured margin,
  configured same-track history support, and traceable root-cause fields.
- `degraded` is used when evidence exists but one or more authorities failed.
- `unavailable` is used when pose evidence cannot be produced.
- Live corrections are append-only and never rewrite prior emitted evidence.
