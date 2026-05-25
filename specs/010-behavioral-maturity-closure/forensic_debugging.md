# Forensic Debugging And Request Lineage Governance

## Purpose

This document defines forensic traceability requirements for runtime debugging and scientific review. It ensures a reviewer can start from a benchmark anomaly, runtime event, or behavior feature and follow the complete causality chain through queues, Triton, fallback, persistence, artifacts, and UI evidence.

## Required Trace Path

The forensic trace MUST resolve:

```text
request
-> queue
-> Triton
-> fallback or retry
-> persistence
-> artifact
-> behavior feature
-> benchmark/profile context
```

Missing nodes MUST be represented as unavailable with explicit reason. Silent gaps are forbidden.

## Inference Trace Fields

Every trace node related to inference MUST include:

```text
request_id
parent_request_id
attempt_id
retry_generation
retry_type
retry_reason
inference_path
batch_id
source_batch_size
effective_batch_size
unresolved_index_origin
item_status
queue_wait_ms
triton_queue_ms
gpu_compute_ms
fallback_overhead_ms
persistence_ms
terminal_state
```

## Retry And Fallback Trace Requirements

Forensic views MUST distinguish:

| Path | Required Display |
|------|------------------|
| `batch` | batch ID, effective size, Triton timing |
| `single_fallback` | original batch ID, item index, fallback reason |
| `degraded` | degradation reason and confidence impact |
| `retry` | parent attempt, retry type, retry generation |
| `timeout_recovery` | timeout source and recovery outcome |

## Workflow Trace Requirements

The trace MUST show state transitions:

```text
QUEUED -> DISPATCHED -> PROCESSING -> PARTIAL_FAILURE -> DEGRADED_COMPLETED
QUEUED -> DISPATCHED -> PROCESSING -> COMPLETED
QUEUED -> DISPATCHED -> PROCESSING -> FAILED
```

If Celery state and DB state diverge, the forensic trace MUST show the reconciliation event and final terminal state.

## Benchmark Causality Trace

Forensic debugging MUST allow a benchmark failure to resolve to:

- timeout spike window
- retry spike window
- queue wait contribution
- GPU idle window
- fallback amplification section
- Triton queue pressure
- partial-batch unresolved index
- terminal-state divergence

## Required Evidence

```text
ci_evidence/production/runtime/benchmark_causality_report.md
ci_evidence/production/runtime/retry_lineage.json
ci_evidence/production/runtime/fallback_resolution_report.md
ci_evidence/production/runtime/terminal_state_reconciliation.json
ci_evidence/production/wave7/forensic_trace_e2e_report.md
```
