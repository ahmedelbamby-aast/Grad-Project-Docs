# Benchmarking And Causality Governance

## Purpose

This document defines benchmark requirements that make performance and scientific claims reproducible, causally explainable, and resistant to false pass conditions.

Live benchmark failures showed that long wall times can coexist with near-zero GPU utilization, fallback retries can amplify work, and benchmark artifacts can become disconnected from runtime causes. This document makes those conditions acceptance blockers.

## Benchmark Integrity Rules

Benchmarks MUST require:

- explicit baseline run set
- explicit candidate run set
- same input digest for baseline and candidate
- same active runtime mode within a comparison
- same endpoint profile class within a comparison
- at least 5 baseline and 5 candidate runs per profile/input for production acceptance
- confidence intervals and variance reporting
- mock vs real labeling
- CPU vs GPU labeling
- live vs offline labeling
- runtime profile labeling

Self-baseline pass logic, synthetic pass states, and missing comparison references are invalid.

## Latency Decomposition

Every accepted benchmark run MUST report:

```text
total_latency = queue_wait + orchestration + serialization + triton_queue + gpu_compute + fallback_overhead + persistence
```

Required fields:

| Field | Review Purpose |
|-------|----------------|
| queue_wait | detect Celery and routing bottlenecks |
| orchestration | detect Python scheduling/control overhead |
| serialization | detect payload encoding/decoding overhead |
| triton_queue | detect Triton scheduler wait and dynamic batch pressure |
| gpu_compute | detect actual model execution cost |
| fallback_overhead | detect single-call fallback and retry amplification |
| persistence | detect DB/artifact write amplification |

The sum MUST reconcile to wall latency within configured tolerance. Unattributed latency above threshold fails acceptance.

## GPU Underutilization Analysis

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

Low GPU utilization with high wall latency invalidates throughput claims unless `gpu_utilization_analysis.md`, `latency_decomposition.json`, and `orchestration_overhead_report.md` identify root cause.

## Retry And Fallback Benchmark Disclosure

Benchmark reports MUST disclose:

- retry count by retry type
- max retry generation
- fallback-path ratio
- single-fallback count
- timeout-recovery count
- unresolved-index count
- duplicate dispatch ratio
- degraded completion count
- partial-batch salvage count

Benchmark results with unresolved retry amplification are not scientifically trustworthy. Partial-batch silent fallback execution MUST be disclosed in benchmark evidence.

## Benchmark Causality Matrix

Each benchmark phase MUST produce a structured failure matrix:

| Window | Runtime Cause | Evidence Link | Acceptance Impact |
|--------|---------------|---------------|-------------------|
| retry spike | retry taxonomy and request lineage | retry lineage artifact | fail if unbounded |
| timeout spike | Triton queue, GPU, or orchestration metric | timeout correlation report | fail if unattributed |
| queue collapse | queue wait p95/p99 and worker pickup | queue evidence | fail if SLO breach unresolved |
| GPU idle | GPU duty cycle and orchestration overhead | GPU analysis | fail throughput claim if unexplained |
| fallback amplification | fallback ratio and retry generation | amplification matrix | fail if above ceiling |

## Required Evidence

```text
ci_evidence/production/runtime/gpu_utilization_analysis.md
ci_evidence/production/runtime/latency_decomposition.json
ci_evidence/production/runtime/orchestration_overhead_report.md
ci_evidence/production/runtime/benchmark_causality_report.md
ci_evidence/production/runtime/retry_amplification_matrix.json
ci_evidence/production/runtime/timeout_correlation_report.md
ci_evidence/production/wave6/statistical_repeatability_report.md
ci_evidence/production/wave8/final_profile_matrix_results.json
```
