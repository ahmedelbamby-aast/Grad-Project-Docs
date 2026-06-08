# Contract: XAI Evidence Envelope

**Contract ID**: `xai.evidence.envelope.v1`
**Owner**: `apps.behavior.explainability`
**Compatibility**: Additive fields only within v1; semantic/type changes require
a new version.

## Required Payload

```json
{
  "schema_version": "xai.evidence.envelope.v1",
  "envelope_id": "uuid",
  "idempotency_key": "sha256:...",
  "signal": {
    "key": "pose.head_yaw",
    "version": "v1",
    "scope": "track_window",
    "unit": "category"
  },
  "scope": {
    "job_id": "uuid",
    "frame_id": "optional",
    "track_ref": "source-scoped-ref",
    "source_start_ms": 1000,
    "source_end_ms": 2000
  },
  "route_snapshot_ref": "uuid",
  "source_refs": ["pose-kinematics:uuid"],
  "timestamps": {
    "source_ms": 1000,
    "ingest_ms": 1010,
    "queue_ms": 1012,
    "inference_ms": 1030,
    "persistence_ms": 1040
  },
  "identity": {
    "local_id": "opaque-source-label",
    "local_namespace": "job-or-stream-ref",
    "canonical_ref": null,
    "continuity": 0.91,
    "reid_refs": [],
    "unresolved": false
  },
  "value_summary": {
    "value": "left",
    "raw_score": 0.84,
    "calibrated_confidence": 0.73
  },
  "quality": {
    "reliability": 0.81,
    "coverage": 0.92,
    "input_quality": 0.88,
    "calibration_ref": "uuid"
  },
  "missingness": {
    "state": "complete",
    "missing_fields": [],
    "unavailable_reasons": []
  },
  "contradictions": [],
  "truth_state": "valid",
  "confidence_band": "review-needed",
  "lineage_digest": "sha256:..."
}
```

## Invariants

- `source_end_ms >= source_start_ms`.
- A valid envelope has a valid immutable route snapshot and at least one source
  reference.
- Local identity is source-scoped; canonical identity is optional.
- Missing/degraded/unavailable values are explicit and are not numeric zero.
- Large arrays/images are artifact references, not embedded payloads.
- Persistence enforces a payload-size limit and idempotency key.
- No envelope text states that a student cheated or intended misconduct.

## Compatibility And Failure

| Condition | Result |
|---|---|
| Unknown signal key/version | reject as `signal_definition_missing` |
| Route snapshot unavailable | withhold production-valid envelope |
| Payload over bound | reject and record `payload_size_exceeded` |
| Missing source refs | `invalidated` |
| Identity continuity below gate | retain evidence, suppress identity-dependent use |
| Re-run same idempotency key | return existing envelope |

## Producer/Consumer Boundary

- Producers: registered `SignalProvider` implementations at existing source
  boundaries.
- Consumers: fast explainers, temporal evidence, anomaly scoring, explanation
  composer, API/WS serializers.
- `apps.anomalies` consumes this contract and does not import pipeline internals.
