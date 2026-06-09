# Contract: Anomaly Review-Priority Score

**Contract ID**: `anomaly.review_priority.v1`
**Owner**: `apps.anomalies.scoring`

## Semantics

`review_priority_score` is a triage score from 0 to 100 indicating how strongly
valid, calibrated, reliable, and temporally supported evidence differs from a
governed observed-pattern profile. It is deterministic signal-pattern
comparison, not a trained anomaly classifier. It is not a cheating probability,
intent estimate, misconduct verdict, or behavioral ground-truth label.

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
  "baseline_snapshot_ref": null,
  "pattern_profile_ref": "uuid",
  "pattern_window_ref": "uuid",
  "calibration_refs": ["uuid"],
  "review_priority_score": 72.4,
  "review_priority_band": "review-needed",
  "pattern_state": "pattern_deviation",
  "ground_truth_status": "unavailable_for_anomaly_behavior",
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
      "pattern_deviation_magnitude": 0.81,
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
    "pattern_deviation_is_not_behavioral_ground_truth",
    "requires_human_review"
  ],
  "reconstruction_digest": "sha256:..."
}
```

## Withholding Rules

The score is null and `truth_state` is degraded/suppressed/invalidated when any
configured blocking gate fails, including:

- route snapshot invalid or stale;
- pattern profile/window missing, cold-start, incompatible, expired, or
  quarantined;
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
- pattern-profile/threshold and optional source-model calibration snapshots;
- optional legacy baseline lineage when present;
- pattern profile/window and feature-schema snapshots;
- context/persistence/uncertainty modifiers;
- exact missingness and validity states.

Reconstruction mismatch blocks acceptance.

## Vocabulary Gate

Persisted/user-facing summaries may use:

- `review priority`;
- `behavioral deviation`;
- `unusual relative to baseline`;
- `within observed pattern`;
- `pattern deviation`;
- `insufficient context`;
- `supporting evidence`;
- `contradicting evidence`;
- `requires human review`.

They may not state or imply:

- confirmed cheating;
- guilt;
- intent;
- misconduct probability;
- automatic penalty eligibility.

They must not imply that `within_observed_pattern` proves normal,
non-cheating, or innocent behavior, or that `pattern_deviation` proves
abnormal intent, misconduct, or cheating.

## Student Independence And Session Aggregation

- A review-approved deviation for one student is student-scoped evidence only.
  It MUST NOT increase, decrease, or relabel any other student's individual
  `review_priority_score`, contribution set, or pattern state.
- Session or classroom aggregation MAY increase when one or more students have
  review-approved deviation evidence, but that increase MUST be computed as a
  separate session-level term rather than by mutating peer student scores.
- Where product discussion uses "normal until proven otherwise", the persisted
  and user-facing state remains `within_observed_pattern`,
  `pattern_deviation`, `insufficient_context`, or `withheld`.

## Derived Session Aggregate Payload

When a session/classroom summary is emitted, it is a separate derived payload,
not a mutation of the student records:

```json
{
  "student_count": 2,
  "approved_deviation_count": 1,
  "base_mean_score": 45.0,
  "approved_deviation_boost": 15.0,
  "session_review_priority_score": 60.0,
  "students": [
    {
      "student_ref": "student-a",
      "review_priority_score": 70.0,
      "pattern_state": "pattern_deviation",
      "review_approved_deviation": true,
      "truth_state": "valid"
    },
    {
      "student_ref": "student-b",
      "review_priority_score": 20.0,
      "pattern_state": "within_observed_pattern",
      "review_approved_deviation": false,
      "truth_state": "valid"
    }
  ]
}
```

## Interaction-Graph Contributions

- Student-interaction-graph features may appear as score contributions under the
  `interaction_graph.*` `component_key` namespace (e.g.
  `interaction_graph.gaze_toward_peer.sustained`,
  `interaction_graph.proximity.dwell`).
- A graph contribution is computed deterministically from identity-gated
  relational evidence and is scoped to the producing student only; it MUST NOT
  add, remove, or relabel any peer student's contributions or score.
- A graph contribution is relational context, never proof of collusion or
  cheating. A learned graph model (`PROBE_ONLY`) MUST NOT supply a production
  contribution.

## No-Ground-Truth Invariants

- `ground_truth_status` is
  `unavailable_for_anomaly_behavior` under this plan.
- The scorer consumes deterministic pattern comparisons, never anomaly-model
  predictions trained/fine-tuned against behavioral targets.
- Reviewer feedback, heuristic outputs, source-model agreement, BSIL output,
  and assumed-normal history cannot be ground truth or training targets.
- Label-based anomaly accuracy/precision/recall/F1/AUROC/AUPRC is not a valid
  acceptance metric.
