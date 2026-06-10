# Cycle 015.2 Investigation: Per-Model Calibration And Reliability

**Created:** 2026-06-11
**Status:** `implemented_local; awaiting production benchmark`
**Streaming compatibility:** `stream-safe` for lookup; fitting is `offline-only`

## Source-model calibration protocol

1. A calibration is fit ONLY from a governed HELD-OUT cohort manifest
   (`xai.calibration_cohort.v1`): `held_out: true` mandatory; target kind must
   be a source-model task label (`detection_label`, `classification_label`,
   `keypoint_visibility`, `segmentation_label`).
2. Fitting runs offline through `manage.py build_xai_calibration` and persists
   an append-only `SourceModelCalibrationSnapshot` (digest-addressed, TTL,
   route-fingerprint compatibility, raw-vs-calibrated evaluation).
3. Runtime is LOOKUP-ONLY: `calibration_service.lookup_snapshot` rejects
   invalidated / signal-mismatched / route-incompatible / expired /
   underpowered snapshots with a governed reason.

## Unavailable-evidence behavior (FR-014)

Where no task-appropriate held-out evidence exists, calibration is EXPLICITLY
unavailable: `CalibrationUnavailable(reason)` propagates one of
`calibration_evidence_missing | calibration_signal_mismatch |
calibration_route_incompatible | calibration_snapshot_expired |
calibration_underpowered | calibration_snapshot_invalidated`. Raw confidence
is never presented as certainty; nothing is inferred from anomaly patterns.

## No-anomaly-ground-truth boundary

Prohibited fitting targets are rejected at the manifest layer:
`anomaly, cheating, non_cheating, normality, abnormal_behavior,
reviewer_feedback, assumed_normal_history`. Every calibrated output carries
`knowledge_limits = [calibration_is_distributional_not_behavioral,
not_anomaly_ground_truth]`.

## Implemented (2026-06-11)

contracts `scoring/calibration.py`; cohort manifests
`scoring/calibration_artifacts.py`; fitters (temperature golden-section NLL,
isotonic PAVA, identity) `scoring/calibrator_registry.py`; lookup/rejection
`scoring/calibration_service.py`; ECE/reliability/Brier/bootstrap-CI/subgroup
evaluation `scoring/calibration_evaluation.py`; offline command
`build_xai_calibration.py`; model+migration 0006; 11 tests (6 unit + 5
integration) passing. Config: `XAI_CALIBRATION_MIN_SAMPLES` (default 200).

## Open acceptance gates

Production benchmark (post-015.17): bounded lookup overhead on the critical
path, reliability-diagram figures from real cohorts, ledger row, rollback
(= no snapshot rows -> all callers see explicit unavailable).
