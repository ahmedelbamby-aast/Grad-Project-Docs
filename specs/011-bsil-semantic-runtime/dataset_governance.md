# BSIL Dataset Governance

## Minimum Dataset Coverage

| Dataset type | Minimum count | Required coverage |
| --- | --- | --- |
| Offline classroom | 3 | varied duration, crowding, posture, attention and object interaction |
| Live RTSP | 2 | reconnect, queue pressure, frame drops and real-time state updates |
| Crowded interaction | 1 or more | multi-person proximity, gaze synchronization and identity separation |
| Occlusion/re-entry | 1 or more | track fragmentation, recovery and suppression behavior |
| Long-duration | 1 or more | state decay, drift, queue growth and operational stability |

## Scientific Requirements

- Dataset manifests MUST include source provenance, digests, capture context,
  allowed use, split policy, label provenance and excluded intervals.
- Labeled behavioral sequences MUST separate observation, heuristic label,
  model inference, anomaly candidate and adjudicated outcome.
- False-positive and false-negative tracking MUST be recorded for behavioral
  state and anomaly validation.
- Scientific maturity claims require repeated runs, variance, confidence,
  baseline/candidate causality and limitations.

