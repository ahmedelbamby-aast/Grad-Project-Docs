# Telemetry Contract - Retry, Inference Path, And Causality

## Purpose

This document defines machine-readable telemetry fields required to make retry behavior, fallback execution, Triton scheduling, GPU utilization, workflow state, and benchmark causality reviewable.

## Required Telemetry Envelopes

All runtime telemetry events MUST include:

```text
event_id
schema_version
timestamp
runtime_mode
profile
request_id
parent_request_id
attempt_id
job_id
session_id
camera_id
source
```

Events MUST be idempotent on `(event_id, session_id, camera_id)` or stronger scoped key.

## Retry Lineage Event

A retry telemetry record MUST include:

```json
{
  "request_id": "string",
  "parent_request_id": "string|null",
  "attempt_id": "string",
  "retry_generation": 0,
  "retry_type": "triton_internal_scheduling|app_fallback_retry|celery_task_retry|degraded_execution_retry|partial_batch_retry",
  "retry_reason": "controlled-vocabulary",
  "inference_path": "batch|single_fallback|degraded|retry|timeout_recovery",
  "original_batch_id": "string|null",
  "timestamp": "RFC3339",
  "queue_wait_ms": 0.0,
  "gpu_wait_ms": 0.0
}
```

## Inference Path Event

Every inference event MUST include:

```json
{
  "inference_path": "batch|single_fallback|degraded|retry|timeout_recovery",
  "batch_id": "string|null",
  "source_batch_size": 0,
  "effective_batch_size": 0,
  "retry_generation": 0,
  "unresolved_index_origin": "integer|null",
  "item_status": "success|retry_pending|failed|salvaged|degraded"
}
```

## Latency Decomposition Event

Accepted benchmark runs MUST emit:

```json
{
  "total_latency_ms": 0.0,
  "queue_wait_ms": 0.0,
  "orchestration_ms": 0.0,
  "serialization_ms": 0.0,
  "triton_queue_ms": 0.0,
  "gpu_compute_ms": 0.0,
  "fallback_overhead_ms": 0.0,
  "persistence_ms": 0.0,
  "gpu_occupancy_pct": 0.0,
  "gpu_duty_cycle_pct": 0.0,
  "idle_gpu_windows": []
}
```

The sum of latency components MUST reconcile with measured wall latency within the configured tolerance. Any unexplained residual MUST be reported as `unattributed_ms` and fails production acceptance when above threshold.

## Workflow State Event

Workflow telemetry MUST report transitions for:

```text
QUEUED
DISPATCHED
PROCESSING
PARTIAL_FAILURE
FAILED
COMPLETED
DEGRADED_COMPLETED
```

Every transition MUST include previous state, next state, actor, task ID, DB row ID, transaction ID if available, and exception context for failure transitions.

## Benchmark Causality Event

Benchmark causality telemetry MUST correlate:

- benchmark_run_id
- input_digest
- runtime_mode
- profile
- request_id list
- timeout spike windows
- retry spike windows
- fallback amplification windows
- queue collapse windows
- GPU idle windows
- Triton queue pressure windows
- terminal-state divergence windows

## Persistence And Export

Telemetry MUST be persisted before benchmark acceptance and exported to deterministic paths:

```text
ci_evidence/production/runtime/retry_lineage.json
ci_evidence/production/runtime/latency_decomposition.json
ci_evidence/production/runtime/terminal_state_reconciliation.json
ci_evidence/production/runtime/retry_amplification_matrix.json
ci_evidence/production/runtime/timeout_correlation_report.md
```
