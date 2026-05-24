# Pose Evaluation Workflow (Step 5)

This workflow compares `RTMPose` and `YOLO Pose` outputs from the same video/run context using shared metric definitions.

## Command

```powershell
python -m scripts.pose_eval.compare_models `
  --rtmpose-records "E:\path\to\rtmpose_pose_results.json" `
  --yolo-records "E:\path\to\yolo_pose_results.json" `
  --rtmpose-video "E:\path\to\rtmpose_pose_output.mp4" `
  --yolo-video "E:\path\to\yolo_pose_output.mp4" `
  --min-confidence 0.3
```

## Artifacts

Each run writes a new timestamped directory under:

- `scripts/pose_eval/reports/<timestamp>_<run_id>/`

Files generated:

- `pose_eval_summary.json`
- `pose_eval_summary.md`

The JSON and markdown include:

- run ID and UTC timestamp
- input settings and paths
- per-model metric values
- RTMPose minus YOLO Pose deltas

## Metrics

- `visible_joint_ratio`: visible joints divided by all parsed joints.
- `temporal_jitter`: mean frame-to-frame Euclidean displacement of visible joints grouped by tracking ID.
- `temporal_stability`: normalized score `1 / (1 + temporal_jitter)`.
- `overlay_completeness`: frames with at least one drawable skeleton divided by total pose frames.

