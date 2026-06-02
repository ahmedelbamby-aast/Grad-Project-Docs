# Pose Quality, Occlusion, and Per-Student Stability

**Last updated:** 2026-05-22

## Summary

Add a shared pose-quality layer for both offline uploads and live streaming. The system will keep raw RTMPose keypoints unchanged, then derive confidence-gated keypoints, per-frame quality, lower-body occlusion state, temporal stability, and per-student summaries grouped by `tracking_id`.

This will make hidden lower-body cases explicit instead of treating them as pose failures.

## Key Changes

### Shared Pose Quality Service

Create a shared backend service used by both offline and live paths.

Default policy:

- `POSE_JOINT_CONF_THRESHOLD=0.25`
- `POSE_STRICT_JOINT_CONF_THRESHOLD=0.35`
- `POSE_OCCLUSION_WINDOW_FRAMES=5`
- `POSE_STABILITY_WINDOW_FRAMES=10`
- `POSE_DISPLAY_SMOOTHING_ALPHA=0.6`

COCO-17 joint groups:

- `upper_body`: nose, eyes, ears, shoulders, elbows, wrists
- `lower_body`: hips, knees, ankles
- `core`: shoulders + hips

For every pose record, compute:

- `visible_joint_ratio`
- `upper_body_visible_ratio`
- `lower_body_visible_ratio`
- `mean_confidence`
- `upper_body_mean_confidence`
- `lower_body_mean_confidence`
- `lower_body_occluded`
- `pose_quality_label`: `good`, `upper_body_only`, `low_confidence`, `unstable`, `missing`
- `is_stable`
- `stability_score`
- `missing_joint_indexes`
- `usable_for_overlay`
- `usable_for_behavior_analysis`

### Offline Video Processing

Extend existing upload pose flow after RTMPose inference.

Artifacts:

- `pose_results.json`: keep existing records, but augment each record with `quality`.
- `pose_per_student.json`: new per-student artifact grouped by `tracking_id`.
- `pose_quality_summary.json`: new job-level summary.

Per-student structure:

```json
{
  "job_id": "...",
  "students": {
    "1": {
      "tracking_id": "1",
      "frame_count": 42,
      "pose_frame_count": 11,
      "stable_frame_count": 8,
      "lower_body_occluded_frame_count": 9,
      "quality_timeline": [],
      "summary": {
        "median_visible_joint_ratio": 0.64,
        "median_upper_body_visible_ratio": 0.91,
        "median_lower_body_visible_ratio": 0.17,
        "dominant_quality_label": "upper_body_only",
        "recommended_use": "upper_body_only"
      }
    }
  }
}
```

Rendering behavior:

- `final_output_video.mp4` draws skeleton only for confident joints/edges.
- `pose_output_video.mp4` draws pose skeleton + inherited tracker ID.
- Hidden lower body does not remove the whole skeleton if upper body is usable.
- Low-confidence joints are skipped visually.
- Optional display smoothing applies only to rendered/video/websocket overlays, not raw `pose_results.json`.

### Live Stream Processing

Add live RTMPose quality support inside `run_live_stream_inference`.

Live behavior:

- Run pose estimation on tracked person detections when `pose_estimation` is enabled.
- Apply the same quality service per `tracking_id`.
- Persist live pose artifacts under the live persistence job/session directory.
- Include quality flags in `overlay.frame` websocket payload.

Live artifacts:

- `pose_results.json`
- `pose_per_student.json`
- `pose_quality_summary.json`

Live websocket `poses[]` remains backward compatible and gains optional fields:

```json
{
  "tracking_id": "1",
  "bbox_xyxy": [0, 0, 0, 0],
  "keypoints": [[x, y, confidence]],
  "display_keypoints": [[x, y, confidence]],
  "score": 0.72,
  "quality": {
    "visible_joint_ratio": 0.65,
    "upper_body_visible_ratio": 0.91,
    "lower_body_visible_ratio": 0.17,
    "lower_body_occluded": true,
    "is_stable": true,
    "pose_quality_label": "upper_body_only"
  }
}
```

### API and Frontend

Extend frame/result payloads without breaking current clients.

API additions:

- Job metadata gains:
  - `pose_quality_artifact_path`
  - `pose_per_student_artifact_path`
  - `pose_quality_summary`
- Frame payload `pose_records[]` gains optional `quality`.
- Add optional endpoint:
  - `GET /api/v1/video-analysis/jobs/<job_id>/pose/students/`
  - returns `pose_per_student.json`.

Frontend:

- Overlay canvas ignores joints below configured confidence.
- Pose skeleton edges are drawn only when both endpoint joints pass threshold.
- Add visual handling for `upper_body_only` and `unstable` states.
- Keep pose visibility controlled by existing `pose_estimation` toggle.

## Stability and Occlusion Rules

Lower-body occlusion:

- `lower_body_occluded=true` when `lower_body_visible_ratio < 0.34` for at least `POSE_OCCLUSION_WINDOW_FRAMES` for the same `tracking_id`.
- If lower body is occluded but `upper_body_visible_ratio >= 0.6`, pose remains usable as `upper_body_only`.

Temporal stability:

- Compare each joint against the previous valid pose for the same `tracking_id`.
- Normalize movement by bbox diagonal.
- Mark unstable if median valid-joint movement exceeds `0.18` bbox diagonal across the stability window.
- Ignore joints that are missing in either frame.
- Raw keypoints stay unchanged; smoothed keypoints are stored separately as `display_keypoints`.

Recommended use:

- `full_body`: stable, enough upper and lower body visible.
- `upper_body_only`: stable upper body, lower body occluded.
- `visual_only`: enough confidence to render, not reliable for behavior logic.
- `not_usable`: missing or unstable.

## Test Plan

Backend unit tests:

- Confidence-gated joint filtering.
- Upper/lower visible ratio calculations.
- Lower-body occlusion window logic.
- Per-track stability scoring.

Offline integration tests:

- Rendered outputs skip low-confidence lower-body joints.

Live integration tests:

- Mock live frames with tracked detections and RTMPose outputs.
- Websocket `overlay.frame` includes `poses[].quality`.
- Live persistence writes per-student pose artifacts.
- Live lower-body occlusion is tracked per student, not globally.

Frontend tests:

- Pose overlay hides low-confidence joints.
- Skeleton edges are skipped when either endpoint is missing.
- `upper_body_only` pose still renders upper skeleton.
- `pose_estimation` toggle still controls all pose overlays.

## Assumptions

- RTMPose uses COCO-17 joint order.
- `tracking_id` is the student identity key for pose grouping.
- No DB migration is required; artifacts remain JSON sidecars.
- Raw pose data remains unchanged for auditability.
- Smoothing is for display/rendering only.
- If pose inference fails, existing silent-skip policy remains: job/live session continues and writes warning telemetry.
