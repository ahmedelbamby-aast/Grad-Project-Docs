# Evidence Requirements - Runtime Remediation

## Purpose

This document defines deterministic evidence outputs required for runtime remediation review. Every artifact must be machine-readable where practical, timestamped, profile-attributed, runtime-mode-attributed, and labeled for mock vs real execution and CPU vs GPU execution.

## Global Evidence Labels

Every artifact introduced by runtime remediation MUST include:

```text
git_sha
created_at
runtime_mode
profile
active_triton_endpoint
input_digest
benchmark_run_id when applicable
mock_vs_real
cpu_vs_gpu
request_lineage_ids
retry_lineage_ids
```

## Required Runtime Evidence Artifacts

| Artifact | Format | Generator | Required Contents | Failure Condition |
|----------|--------|-----------|-------------------|------------------|
| `ci_evidence/production/runtime/retry_lineage.json` | JSON | retry telemetry exporter | request IDs, parent IDs, retry type, retry reason, inference path, timing | missing retry attempt or missing parent linkage |
| `ci_evidence/production/runtime/fallback_resolution_report.md` | Markdown | batch salvage reporter | unresolved indexes, selective retries, salvaged items, failed items, duplicate dispatch ratio | silent fallback or frame-wide invalidation |
| `ci_evidence/production/runtime/gpu_utilization_analysis.md` | Markdown | GPU analyzer | occupancy, duty cycle, idle windows, memory pressure, root-cause narrative | low GPU/high wall with no attribution |
| `ci_evidence/production/runtime/latency_decomposition.json` | JSON | latency decomposition exporter | queue, orchestration, serialization, Triton queue, GPU compute, fallback, persistence | wall latency does not reconcile |
| `ci_evidence/production/runtime/orchestration_overhead_report.md` | Markdown | perf/flamegraph exporter | Python scheduling overhead, serialization hotspots, queue orchestration cost | orchestration dominates without explanation |
| `ci_evidence/production/runtime/workflow_integrity_report.md` | Markdown | workflow watchdog | state-machine transitions and divergence summary | Celery/DB divergence unresolved |
| `ci_evidence/production/runtime/terminal_state_reconciliation.json` | JSON | reconciliation command | Celery result state, DB state, artifact state, reconciliation action | DB remains processing after failure |
| `ci_evidence/production/runtime/benchmark_causality_report.md` | Markdown | benchmark causality exporter | correlated failure windows and runtime causes | benchmark metrics detached from runtime data |
| `ci_evidence/production/runtime/retry_amplification_matrix.json` | JSON | retry amplification detector | retry generations, duplicate fanout, fallback ratio | unresolved amplification |
| `ci_evidence/production/runtime/timeout_correlation_report.md` | Markdown | timeout correlator | Triton timeout windows, queue pressure, GPU idle, fallback events | timeout spike unattributed |

## Structured Evidence Requirements

Artifacts MUST be deterministic enough for PR review:

- stable run IDs
- stable input digests
- stable request lineage IDs
- stable profile labels
- sorted records by timestamp and request ID
- explicit unavailable markers instead of omitted fields
- no synthetic pass values
- no hidden mock results in production acceptance

## Scientific Evidence Rejection Conditions

The following evidence packages are invalid:

- benchmark result without latency decomposition
- throughput claim with low GPU utilization and no orchestration attribution
- retry/fallback behavior without lineage
- partial-batch fallback not disclosed
- Celery failure with DB processing state unresolved
- readiness report that marks inactive endpoint production-ready
- benchmark pass generated from self-baseline comparison
