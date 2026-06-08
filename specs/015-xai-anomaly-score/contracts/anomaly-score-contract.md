# Contract: Anomaly Review-Priority Score

**Contract ID**: `anomaly.review_priority.v1`
**Owner**: `apps.anomalies.scoring`

## Semantics

`review_priority_score` is a triage score from 0 to 100 indicating how strongly
valid, calibrated, reliable, and temporally supported evidence differs from a
governed baseline. It is not a cheating probability, intent estimate, or
misconduct verdict.

## Required Payload

```json
{
  "schema_version": "anomaly.review_priority.v1",
  "score_id": "uuid",
  "scope": {
    "job_id": "uuid",
    "track_ref": "source-scoped-ref",
    "source_start_ms": 1000,
    "source_end_ms": 10000
  },
  "route_snapshot_ref": "uuid",
  "score_profile": {
    "key": "transparent_hierarchical",
    "version": "v1"
  },
  "baseline_snapshot_ref": "uuid",
  "calibration_refs": ["uuid"],
  "review_priority_score": 72.4,
  "review_priority_band": "review-needed",
  "truth_state": "valid",
  "evidence_coverage": 0.87,
  "reliability": 0.79,
  "uncertainty": 0.19,
  "context_support": 0.75,
  "persistence_support": 0.83,
  "contributions": [
    {
      "component_key": "pose.head_yaw.repeated_pattern",
      "evidence_ref": "uuid",
      "calibrated_surprise": 0.81,
      "reliability": 0.85,
      "temporal_support": 0.88,
      "configured_weight": 1.0,
      "effective_contribution": 0.606,
      "validity_state": "valid",
      "reason_codes": ["sustained_deviation"],
      "counterfactual_delta": {
        "to_enter_threshold": -0.12,
        "to_recover_threshold": -0.29
      }
    }
  ],
  "contradictions": [],
  "withheld_reasons": [],
  "knowledge_limits": [
    "score_does_not_establish_intent",
    "requires_human_review"
  ],
  "reconstruction_digest": "sha256:..."
}
```

## Withholding Rules

The score is null and `truth_state` is degraded/suppressed/invalidated when any
configured blocking gate fails, including:

- route snapshot invalid or stale;
- baseline missing, contaminated, drifting, or quarantined;
- required calibration missing, stale, incompatible, or underpowered;
- valid evidence coverage below threshold;
- identity continuity below threshold for identity-dependent components;
- temporal window invalid or crosses a reconnect gap;
- uncertainty above threshold;
- reconstruction or persistence failure.

## Reconstruction

The scorer must reproduce the stored score from:

- contribution records;
- score profile and configured weights;
- baseline/threshold/calibration snapshots;
- context/persistence/uncertainty modifiers;
- exact missingness and validity states.

Reconstruction mismatch blocks acceptance.

## Vocabulary Gate

Persisted/user-facing summaries may use:

- `review priority`;
- `behavioral deviation`;
- `unusual relative to baseline`;
- `supporting evidence`;
- `contradicting evidence`;
- `requires human review`.

They may not state or imply:

- confirmed cheating;
- guilt;
- intent;
- misconduct probability;
- automatic penalty eligibility.
