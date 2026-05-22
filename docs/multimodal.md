# Multimodal Enhancements for Cheating Analysis and Prediction

1. Multi-modal cheating signals  
Combine pose, gaze, head direction, hand motion, phone/object detection, and audio cues into one risk score instead of relying on a single model.

2. Temporal behavior modeling  
Add sequence models (Transformer/LSTM) over 10-60s windows so the system detects patterns (repeated side glances, synchronized head turns, hidden object usage), not just frame-level events.

3. Identity-consistent tracking  
Strengthen ReID and long-term tracking to avoid ID switches; this improves per-student behavior history and reduces false accusations from track drift.

4. Seat-map and context awareness  
Use calibrated room geometry and seat zones. Flag interactions based on who is near whom, desk boundaries, and forbidden movement paths.

5. Exam-phase-aware rules  
Use different thresholds for pre-exam, active exam, and submission windows. Context-aware thresholds reduce false positives.

6. Personalized baseline per room/camera  
Learn normal behavior per camera, lighting condition, and class type. Global models + per-site calibration usually improve precision.

7. Better confidence calibration  
Calibrate output probabilities (temperature scaling / isotonic regression) so “high risk” actually means high risk; improves alert trust.

8. Human-in-the-loop feedback loop  
Let proctors confirm/reject alerts; feed that back into retraining. This is one of the fastest ways to improve real-world accuracy.

9. Hard-negative mining  
Continuously collect false positives (normal behavior incorrectly flagged) and retrain on them to reduce over-triggering.

10. Active learning for edge cases  
Automatically surface uncertain clips for annotation (low confidence, conflicting models, unusual scenes) to improve data efficiency.

11. Graph-based collusion detection  
Model student interactions as a graph over time (gaze exchange, synchronized gestures, passing objects) to detect collaborative cheating patterns.

12. Explainable alerts  
Attach evidence snippets (events timeline + boxes + track IDs + reasons). Better explanations improve reviewer consistency and label quality.

13. Robustness improvements  
Add domain adaptation for lighting/angle shifts, occlusion handling, and motion blur robustness to keep performance stable across rooms.

14. Ensemble decision layer  
Use multiple specialized detectors with weighted fusion and rule constraints rather than one monolithic model.

15. Strong evaluation protocol  
Track per-scenario metrics: precision/recall by camera, room type, crowd density, and cheating type. Optimize to reduce false positives first, then lift recall.
