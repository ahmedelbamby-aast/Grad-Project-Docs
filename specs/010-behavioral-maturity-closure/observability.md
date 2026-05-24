# Observability And Runtime Trust Governance

## Purpose

This document defines observability requirements for real runtime probes, per-attempt telemetry, GPU underutilization root-cause analysis, workflow integrity, retry lineage, benchmark causality, and frontend/backend truth alignment.

It exists because live benchmark execution exposed high wall latency with near-zero GPU utilization, Triton timeout spikes, fallback ambiguity, Celery/DB state divergence, and benchmark artifacts that could not explain runtime causality.

## Probe-Backed Truth

Production observability MUST be backed by real probes:

- Triton active endpoint health
- Triton metrics endpoint
- model readiness
- GPU visibility and utilization
- Celery worker/queue state
- Redis connectivity
- DB connectivity
- backend health
- frontend health
- workflow reconciliation state

Hardcoded availability, synthetic readiness, and fake pass states are forbidden in production evidence.

## GPU Underutilization Root-Cause Requirements

Benchmark acceptance MUST measure:

1. GPU occupancy
2. GPU duty cycle
3. CPU orchestration overhead
4. queue wait contribution
5. Python scheduling overhead
6. serialization overhead
7. Triton queue delay
8. model execution time
9. fallback-path ratio
10. idle GPU windows

Required latency decomposition:

```text
total_latency = queue_wait + orchestration + serialization + triton_queue + gpu_compute + fallback_overhead + persistence
```

A run with low GPU utilization and high wall latency is not accepted as throughput evidence unless attribution proves where the time was spent.

## Per-Attempt Observability

Every inference attempt MUST emit:

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
queue_wait_ms
triton_queue_ms
gpu_compute_ms
fallback_overhead_ms
persistence_ms
terminal_state
```

## Workflow Integrity Observability

The system MUST expose state divergence metrics:

- Celery result state vs DB job state
- runtime event state vs artifact terminal state
- reconciliation success/failure counts
- orphaned processing jobs
- exception-safe finalization failures

The combination `Celery FAILURE` and `DB processing` is a production correctness defect, not only an observability issue.

## Benchmark Causality Observability

Benchmark reports MUST correlate:

- runtime failures
- retry spikes
- timeout spikes
- queue collapse
- GPU idle windows
- fallback amplification
- Triton queue pressure
- partial-batch unresolved indexes
- workflow terminal-state divergence

Required artifacts:

```text
ci_evidence/production/runtime/benchmark_causality_report.md
ci_evidence/production/runtime/retry_amplification_matrix.json
ci_evidence/production/runtime/timeout_correlation_report.md
```

## Frontend Truth Requirements

Frontend runtime surfaces MUST distinguish:

| Value | Meaning |
|-------|---------|
| `null` | unknown or not yet measured |
| `0` | measured zero |
| `unavailable` | collector unavailable or failed |

Frontend code MUST NOT overwrite backend runtime KPIs to zero. If telemetry is unavailable, it must display unavailable state and link to probe evidence.

## Required Evidence

```text
ci_evidence/production/runtime/gpu_utilization_analysis.md
ci_evidence/production/runtime/latency_decomposition.json
ci_evidence/production/runtime/orchestration_overhead_report.md
ci_evidence/production/runtime/workflow_integrity_report.md
ci_evidence/production/runtime/benchmark_causality_report.md
ci_evidence/production/wave6/telemetry_probe_report.json
ci_evidence/production/wave6/frontend_kpi_truth_report.md
```
