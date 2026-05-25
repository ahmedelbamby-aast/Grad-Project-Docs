# Queueing, Retry, And Backpressure Governance

## Purpose

This document defines queue authority, retry/fallback boundaries at the Celery layer, queue wait telemetry, starvation and collapse detection, workflow state reconciliation, and backpressure integration. It closes observed live benchmark failures where queue waiting and Python orchestration dominated wall latency while GPU utilization remained low.

## Queue Authority

The queue map MUST define:

- queue name
- owner
- workload type
- runtime mode eligibility
- priority
- retry policy
- timeout policy
- DLQ policy
- backpressure action
- telemetry label

Queue metric names MUST match actual queue names. Alias-only reporting is forbidden for production evidence.

## Queue Wait Telemetry

Every task envelope MUST carry:

```text
request_id
parent_request_id
task_id
queue_name
routing_key
entered_queue_at
dequeued_at
worker_started_at
worker_finished_at
queue_wait_ms
worker_pickup_delay_ms
execution_ms
runtime_mode
profile
```

Queue wait MUST be included in the benchmark latency equation:

```text
total_latency = queue_wait + orchestration + serialization + triton_queue + gpu_compute + fallback_overhead + persistence
```

## Retry And Fallback Governance At Queue Boundary

Celery retry is one taxonomy type, not a generic escape path. A Celery task retry MUST NOT trigger application fallback recursion unless the retry policy explicitly allows it and a lineage record exists.

Forbidden queue behaviors:

- retrying a task without preserving original request lineage
- duplicating unresolved-index fanout through multiple tasks
- emitting task `SUCCESS` for silent degraded execution
- leaving DB state as `PROCESSING` after Celery `FAILURE`
- treating queue starvation as Triton model latency

## Starvation And Collapse Detection

The runtime MUST detect:

| Condition | Required Signal | Required Action |
|-----------|-----------------|-----------------|
| starvation | p95/p99 queue wait breach with low worker pickup rate | alert and optionally start bounded degradation |
| collapse | queue depth growth plus timeout spike | backpressure or fail-stop according to policy |
| priority inversion | low-priority queue consuming critical workers | route correction and evidence event |
| fallback amplification | fallback tasks exceed bounded ratio | reject benchmark and stop recursion |
| dead consumer | queue grows with no active consumer | health failure and remediation hint |

## Workflow State Reconciliation

The queueing layer MUST participate in the authoritative workflow state machine:

```text
QUEUED -> DISPATCHED -> PROCESSING -> PARTIAL_FAILURE -> DEGRADED_COMPLETED
QUEUED -> DISPATCHED -> PROCESSING -> COMPLETED
QUEUED -> DISPATCHED -> PROCESSING -> FAILED
```

The reconciliation watchdog MUST compare Celery result state, DB job state, runtime event state, and artifact terminal state. Any state divergence MUST produce a remediation event and evidence row.

## Backpressure Control

Backpressure decisions MUST depend on:

- queue depth
- queue wait p95/p99
- GPU duty cycle
- Triton timeout rate
- worker saturation
- fallback amplification ratio
- persistence latency

Backpressure actions MUST be explicit:

- cadence increase
- bounded frame dropping
- inference shedding
- degraded execution with attribution
- fail-stop when correctness is unsafe

Silent degradation is forbidden.

## Required Evidence

```text
ci_evidence/production/wave2/queue_routing_matrix.md
ci_evidence/production/wave2/active_queues.txt
ci_evidence/production/wave2/live_queue_wait_events.json
ci_evidence/production/wave2/backpressure_slo_report.md
ci_evidence/production/runtime/retry_lineage.json
ci_evidence/production/runtime/workflow_integrity_report.md
ci_evidence/production/runtime/terminal_state_reconciliation.json
ci_evidence/production/runtime/retry_amplification_matrix.json
```
