# YOLO26-Pose Medium Integration Plan

## Integration Plan (YOLO26-Pose Medium)

1. Define model contract and outputs
- Confirm input size, keypoint format, confidence fields, class mapping, latency target.
- Standardize output schema for both offline and live paths: `frame_id`, `track_id`, `keypoints[17/33]`, `kp_conf`, `bbox`, `pose_conf`.

2. Add model artifact + runtime config
- Place YOLO26-Pose-M beside person detector in model registry.
- Add config flags: `POSE_ENABLED`, `POSE_MODEL_PATH`, `POSE_CONF`, `POSE_NMS`, `POSE_INPUT_SIZE`, `POSE_FPS_STRIDE`.

3. Extend inference orchestrator
- Add pose model as a parallel branch in the same per-frame pipeline used by:
  - Video upload task
  - Live streaming task
- Ensure shared preprocessing where possible to minimize duplicated compute.

4. Align with tracking IDs
- Fuse pose detections with person detector tracks (IoU + center distance + class gate).
- Persist keypoints under stable `track_id` so you can compute temporal behavior features.

5. Data persistence + event schema
- Store compact pose data per frame (DB/JSONL/Redis stream): only needed keypoints + derived features.
- Add versioned schema (`pose_schema_v1`) for backward compatibility.

6. Build derived feature extractor layer
- Per frame: joint angles, limb velocity, torso inclination, head orientation proxy, hand-to-head distance, hand raise state, sitting/standing confidence.
- Per window (5-30s): repetition count, dwell time, anomaly score, peer-synchrony features.

7. Integrate into risk/anomaly scoring
- Add pose features into existing cheating-risk engine as weighted signals (not standalone verdict).
- Keep explainability payload with "why flagged" evidence from keypoints and temporal stats.

8. Performance optimization
- Add adaptive sampling: run pose every `N` frames when load is high; interpolate lightweight features between inferences.
- Use batching for upload pipeline; bounded queue for live pipeline.

9. Validation and QA
- Unit tests for model output parsing, track association, feature math.
- Integration tests for both RTSP/TCP live streams + uploaded videos.
- Regression suite for latency, frame throughput, false positives.

10. Deployment + rollout
- Shadow mode first: compute pose + features without affecting final decisions.
- Compare metrics vs baseline; then enable gated production scoring.

## Benefits of YOLO Pose Models in This System

1. Stronger behavioral understanding
- Pose gives body mechanics, not just object presence.

2. Better cheating pattern detection
- Captures subtle cues: repeated side turns, hand-to-ear, hidden phone posture, signaling gestures.

3. Improved temporal reasoning
- Keypoints are ideal for sequence modeling and behavior timelines.

4. Higher explainability
- Easier to justify alerts with concrete body-joint evidence.

5. Better robustness in crowded scenes
- Person bbox + keypoints together improve interpretation under partial occlusion.

6. Supports richer analytics
- Engagement, posture quality, motion anomalies, and interaction patterns become measurable.

## Engines That Can Be Built on Top of Pose Data

1. Pose-to-Cheating Risk Engine
- Weighted rules + ML score from pose features over time windows.

2. Temporal Pattern Engine
- Detect repeated suspicious micro-patterns (glance cycles, hand movement loops).

3. Head/Gaze Proxy Engine
- Estimate head orientation and gaze direction changes from face/shoulder keypoints.

4. Hand Activity Engine
- Detect hand raise, hand-to-mouth/ear, desk-underreach, object handling posture.

5. Peer Synchrony / Collusion Engine
- Cross-student timing correlation for coordinated behaviors.

6. Posture Compliance Engine
- Sitting/standing transitions, abnormal leaning, seat boundary violations.

7. Session Drift Engine
- Detect sudden behavior distribution changes within a session (possible policy breach onset).

8. Confidence & Quality Engine
- Track keypoint quality and uncertainty; suppress low-quality evidence to reduce false alarms.

9. Explainability Engine
- Generate human-readable incident summaries with timestamped pose evidence.

10. Active Learning Engine
- Select uncertain/high-impact pose segments for annotation and retraining.
