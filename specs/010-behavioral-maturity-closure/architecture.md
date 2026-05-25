# Architecture Governance - Runtime Remediation Addendum

## Purpose

This document defines the architecture-level remediation requirements introduced after live benchmark execution exposed timeout spikes, low GPU duty cycle, duplicate fallback dispatch, partial-batch unresolved indexes, Celery/DB terminal-state divergence, runtime mode ambiguity, and benchmark evidence that could not be traced back to runtime causality.

This document exists to prevent runtime implementation details from being treated as isolated bugs. The failures affect architecture boundaries: request identity, queue ownership, Triton execution authority, retry lineage, fallback semantics, workflow finalization, telemetry trust, benchmark validity, and scientific claim integrity.

If this document is omitted, future implementation can accidentally preserve high-level maturity claims while the runtime still contains hidden retry amplification, silent degraded execution, and unverifiable throughput measurements.

## Runtime Remediation Architecture

The production runtime MUST represent every inference request as a causally traceable execution graph:

```text
external_request
-> backend_request_context
-> Celery task envelope
-> queue wait interval
-> Triton request attempt
-> batch item resolution
-> fallback or retry attempt when allowed
-> persistence transaction
-> artifact write
-> telemetry event
-> benchmark evidence row
-> forensic trace node
```

The graph MUST remain valid for successful execution, partial success, degraded completion, timeout recovery, and failure. Missing edges MUST be represented explicitly as `unavailable` with reason, not silently omitted.

## Authoritative Components

| Component | Architectural Authority | Required Output |
|-----------|--------------------------|-----------------|
| Retry policy | `backend/apps/pipeline/retry_policy.py` | retry type, ceiling, generation, reason, rejection cause |
| Request lineage | `backend/apps/pipeline/request_lineage.py` | request_id, parent_request_id, batch_id, attempt_id |
| Inference runtime | `backend/apps/pipeline/inference_runtime.py` | inference_path, batch salvage status, unresolved indexes |
| Triton telemetry | `backend/apps/pipeline/triton_queue_telemetry.py` | Triton queue delay, dynamic batch wait, model execution time |
| GPU telemetry | `backend/apps/pipeline/gpu_telemetry.py` | occupancy, duty cycle, idle windows, memory pressure |
| Workflow integrity | `backend/apps/video_analysis/tasks.py` and reconciliation command | terminal-state consistency between Celery and DB |
| Benchmark causality | `tools/benchmarks/export_benchmark_causality.py` | timestamp-correlated failure and latency attribution |
| Forensic trace | `backend/apps/behavior/forensics.py` and frontend trace view | request-to-artifact traceability |

## Retry And Fallback Governance Boundary

Retry behavior MUST be architecture-governed, not left to individual call sites. The runtime MUST classify retries as one of:

| Retry Type | Owner | Allowed Scope | Forbidden Behavior |
|------------|-------|---------------|--------------------|
| Triton internal scheduling | Triton server | internal scheduler wait before execution | application treating scheduler wait as a new app retry |
| App-level fallback retry | inference runtime | bounded fallback from batch item to allowed fallback path | recursive fallback fanout |
| Celery task retry | task orchestration | bounded task-level retry for retryable infrastructure failures | retry after terminal DB failure without reconciliation |
| Degraded execution retry | degradation controller | bounded degraded attempt with explicit `degraded` attribution | silent degraded completion |
| Partial-batch retry | batch salvage controller | per-item unresolved-index retry | frame-wide redispatch or duplicate unresolved-index fanout |

Every retry attempt MUST emit `request_id`, `parent_request_id`, `retry_reason`, `retry_type`, `inference_path`, `original_batch_id`, `timestamp`, `queue_wait_ms`, and `gpu_wait_ms`. Retry lineage MUST persist through telemetry, artifacts, and benchmark evidence.

## Partial-Batch Salvage Architecture

Partial-batch failure MUST be item-scoped. One failed crop, unresolved index, timeout, or malformed ROI MUST NOT invalidate successful crop outputs for the same frame.

Required item states:

| State | Meaning | Persistence Requirement |
|-------|---------|-------------------------|
| `success` | item completed through batch execution | persist output with batch_id and item index |
| `retry_pending` | item unresolved and eligible for bounded retry | persist origin and retry generation |
| `failed` | item exhausted retry policy or non-retryable | persist failure reason and missing-data marker |
| `salvaged` | item completed through selective retry | persist original batch and fallback attempt |
| `degraded` | item completed through degraded path | persist degraded attribution and confidence impact |

Batch salvage MUST preserve source item ordering for artifact readability while never using list index as identity or scientific association authority.

## Workflow Integrity Architecture

The authoritative workflow state machine is:

```text
QUEUED -> DISPATCHED -> PROCESSING -> COMPLETED
QUEUED -> DISPATCHED -> PROCESSING -> PARTIAL_FAILURE -> DEGRADED_COMPLETED
QUEUED -> DISPATCHED -> PROCESSING -> FAILED
```

Terminal states are `FAILED`, `COMPLETED`, and `DEGRADED_COMPLETED`. Terminal transitions MUST be atomic with audit emission. Celery `FAILURE` while DB job state remains `PROCESSING` is forbidden and MUST be detected by reconciliation.

## Runtime Mode Architecture

The system preserves dual configured Triton endpoint profiles while enforcing one active production mode at runtime.

| Mode | Active Endpoint | Inactive Endpoint Requirement |
|------|-----------------|-------------------------------|
| `live` | live Triton ports 39000/39001/39002 | offline endpoint must not receive production inference or be marked production-ready |
| `offline` | offline Triton ports 39100/39101/39102 | live endpoint must not receive production inference or be marked production-ready |

Startup MUST fail if `required_offline=True required_live=True reason=forced_triton_not_ready`, active mode is ambiguous, active endpoint is unavailable, inactive endpoint is consumed, model version drift is detected, `config.pbtxt` fails validation, warmup shapes mismatch, or TensorRT engine lineage cannot be verified.

## Benchmark Causality Architecture

Benchmark acceptance MUST correlate runtime measurements with causality:

```text
total_latency = queue_wait + orchestration + serialization + triton_queue + gpu_compute + fallback_overhead + persistence
```

A run with low GPU utilization and high wall latency is not a throughput success unless the benchmark includes orchestration attribution, queue wait contribution, Triton queue delay, fallback ratio, idle GPU windows, and persistence overhead.

Required runtime evidence:

```text
ci_evidence/production/runtime/retry_lineage.json
ci_evidence/production/runtime/fallback_resolution_report.md
ci_evidence/production/runtime/gpu_utilization_analysis.md
ci_evidence/production/runtime/latency_decomposition.json
ci_evidence/production/runtime/orchestration_overhead_report.md
ci_evidence/production/runtime/workflow_integrity_report.md
ci_evidence/production/runtime/terminal_state_reconciliation.json
ci_evidence/production/runtime/benchmark_causality_report.md
ci_evidence/production/runtime/retry_amplification_matrix.json
ci_evidence/production/runtime/timeout_correlation_report.md
```

## Wave Ownership

| Wave | Runtime Remediation Ownership |
|------|-------------------------------|
| Wave 2 | queue wait, retry governance at Celery boundary, workflow reconciliation, starvation/collapse detection |
| Wave 4 | Triton runtime correctness, partial-batch salvage, inference-path attribution, dynamic batching behavior |
| Wave 6 | telemetry trust, GPU underutilization root-cause analysis, latency decomposition, benchmark causality |
| Wave 8 | final rejection of unresolved retry amplification, unattributed GPU starvation, terminal-state divergence, and benchmark/runtime disconnects |
