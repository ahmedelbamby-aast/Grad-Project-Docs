# Runtime Policy Governance

## Purpose

This document defines production runtime authority for inference execution, retry/fallback behavior, runtime mode selection, workflow state finalization, and fail-closed behavior. It is mandatory for implementation review because live benchmark failures showed that timeout spikes, duplicated fallback dispatches, forced Triton readiness contradictions, and Celery/DB state divergence can invalidate both operational and scientific evidence.

## Production Runtime Authority

Production runtime MUST use Triton-only inference on Linux. Local ONNX, standalone TensorRT, OpenVINO, mock inference, and synthetic readiness are not production authority paths.

Production configuration MUST preserve dual Triton endpoint profiles but enforce one active production mode at runtime:

| Runtime Mode | Active Endpoint | Inactive Endpoint Semantics |
|--------------|-----------------|-----------------------------|
| `live` | live Triton HTTP/gRPC/metrics | offline endpoint receives no scheduler traffic and is not production-ready |
| `offline` | offline Triton HTTP/gRPC/metrics | live endpoint receives no scheduler traffic and is not production-ready |

The active mode MUST be selected by `TRITON_EXECUTION_MODE`. Startup MUST fail if the mode is missing, invalid, ambiguous, or contradicts endpoint readiness.

## Retry Taxonomy

Retry behavior MUST be explicit and classified before execution.

| Retry Type | Description | Ceiling Source | Telemetry Requirement |
|------------|-------------|----------------|-----------------------|
| `triton_internal_scheduling` | Triton scheduler wait before execution | Triton model config and latency SLO | Triton queue delay and dynamic batch wait |
| `app_fallback_retry` | application retry after allowed fallback decision | retry policy config | retry_reason, fallback origin, inference_path |
| `celery_task_retry` | Celery-level retry for retryable orchestration failure | task policy | task_id, retry_generation, DB state before/after |
| `degraded_execution_retry` | retry under degraded mode such as reduced cadence or smaller batch | degradation policy | degraded reason and output confidence impact |
| `partial_batch_retry` | per-item retry for unresolved batch entries | batch salvage policy | original_batch_id, unresolved_index_origin, item status |

## Retry Prohibitions

The runtime MUST forbid:

- uncontrolled retry recursion
- duplicate unresolved-index fanout
- hidden fallback amplification
- retry behavior without a parent request
- retry behavior that loses original batch attribution
- degraded execution that is reported as normal success
- Celery retry that leaves DB state in `PROCESSING` after task `FAILURE`

## Retry Ceilings

The runtime MUST define and enforce:

| Ceiling | Default Policy Requirement | Failure Behavior |
|---------|----------------------------|------------------|
| max fallback attempts | finite integer per inference request | mark unresolved item failed or degraded, not recursively retry |
| max unresolved-index retries | finite integer per batch item | persist failed item with unresolved origin |
| max degraded retries | finite integer per request | terminal `FAILED` or `DEGRADED_COMPLETED` with evidence |
| max Celery task retries | finite integer per retryable failure class | terminal DB transition through reconciliation |

Concrete numeric defaults belong in implementation config, but unlimited values are forbidden in production.

## Per-Attempt Telemetry Contract

Every retry attempt MUST emit and persist:

| Field | Requirement |
|-------|-------------|
| `request_id` | stable ID for the original user/job request |
| `parent_request_id` | immediate parent attempt or null for root |
| `attempt_id` | unique attempt ID |
| `retry_reason` | controlled vocabulary reason |
| `retry_type` | one taxonomy value |
| `inference_path` | `batch`, `single_fallback`, `degraded`, `retry`, or `timeout_recovery` |
| `original_batch_id` | source batch that created the attempt |
| `timestamp` | monotonic or wall timestamp with timezone context |
| `queue_wait_ms` | queue wait contribution before execution |
| `gpu_wait_ms` | GPU or Triton wait contribution |

Telemetry evidence MUST be exported to `ci_evidence/production/runtime/retry_lineage.json`.

## Workflow State Machine

The authoritative workflow states are:

```text
QUEUED
DISPATCHED
PROCESSING
PARTIAL_FAILURE
FAILED
COMPLETED
DEGRADED_COMPLETED
```

Terminal states are `FAILED`, `COMPLETED`, and `DEGRADED_COMPLETED`. Terminal transitions MUST be atomic, exception-safe, and reconciled against Celery result state.

Forbidden state combination:

```text
Celery result = FAILURE
DB job state = PROCESSING
```

Reconciliation watchdogs MUST emit `ci_evidence/production/runtime/terminal_state_reconciliation.json` and `ci_evidence/production/runtime/workflow_integrity_report.md`.

## Readiness Fail-Closed Rules

Startup MUST fail when:

- Triton readiness graph reports a mismatch
- `required_offline=True required_live=True reason=forced_triton_not_ready` appears for a single active production mode
- active endpoint is unavailable
- inactive endpoint receives scheduler or inference traffic
- model repository versions drift from deployment manifest
- `config.pbtxt` schema validation fails
- RTMPose warmup shape mismatches runtime bindings
- TensorRT engine lineage cannot be verified
- production local fallback path is enabled

## Acceptance Evidence

Runtime policy acceptance requires:

```text
ci_evidence/production/runtime/retry_lineage.json
ci_evidence/production/runtime/fallback_resolution_report.md
ci_evidence/production/runtime/workflow_integrity_report.md
ci_evidence/production/runtime/terminal_state_reconciliation.json
ci_evidence/production/wave1/model_serving_health.json
ci_evidence/production/wave1/boundary_verifier.txt
```
