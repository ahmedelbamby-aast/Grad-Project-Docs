# Identity And Temporal Integrity Under Runtime Degradation

## Purpose

This document defines how retry, fallback, partial success, queue delay, and workflow finalization affect identity continuity and temporal validity. Runtime degradation must not corrupt behavioral sequences or silently attach evidence to the wrong student, camera, session, frame, or timestamp.

## Runtime Degradation Constraint

When inference retries, fallback execution, partial-batch salvage, or degraded completion occurs, the system MUST preserve:

- `session_id`
- `camera_id`
- `canonical_track_id`
- `local_track_id`
- source timestamp
- ingest timestamp
- queue timestamp
- processing timestamp
- persistence timestamp
- batch ID
- inference path
- retry generation

A fallback or retry attempt MUST NOT create a new scientific identity. It is an attempt under the same request lineage unless ReID explicitly creates a canonical identity decision with auditable evidence.

## Partial-Batch Temporal Semantics

Partial-batch failure creates item-level temporal gaps or degraded observations, not frame-level erasure unless the entire frame is invalid.

| Item Outcome | Temporal Representation |
|--------------|-------------------------|
| success | normal sequence row with source batch attribution |
| failed | explicit missing-data marker with failure reason |
| retry_pending | no mature feature until retry resolves or expires |
| salvaged | sequence row includes fallback/retry lineage |
| degraded | sequence row includes degraded confidence semantics |

Silent fallback is forbidden because it hides observation quality from behavior features and anomaly primitives.

## Sequence Validity Requirements

Sequence records derived from runtime attempts MUST include:

```text
request_id
attempt_id
batch_id
inference_path
retry_generation
item_status
source_timestamp_ms
processing_timestamp_ms
identity_scope
missing_data_reason
```

Behavioral features MUST treat missing, degraded, and salvaged observations differently. A salvaged observation can be usable with lower confidence; a failed observation must become an explicit mask or gap.

## Scientific Rejection Rules

The following conditions invalidate temporal behavior claims unless explicitly excluded or explained:

- retry amplification changes observation density without disclosure
- fallback execution bypasses pose quality semantics
- unresolved indexes are silently filled
- queue delays reorder temporal windows
- workflow failure leaves incomplete sequences marked processing
- low GPU/high wall latency changes frame cadence without attribution

## Evidence Requirements

```text
ci_evidence/production/runtime/fallback_resolution_report.md
ci_evidence/production/runtime/retry_lineage.json
ci_evidence/production/runtime/latency_decomposition.json
ci_evidence/production/wave5/sequence_schema_contract.md
ci_evidence/production/wave5/feature_output_samples.json
```
