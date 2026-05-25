# Acceptance Gates - Runtime Remediation Extension

## Purpose

This document defines acceptance gates for runtime failure remediation. These gates are mandatory additions to the wave acceptance model and apply especially to Waves 2, 4, 6, and 8.

## Gate A - Retry And Fallback Governance

Pass criteria:

- retry taxonomy exists and is enforced
- retry ceilings exist for fallback attempts, unresolved-index retries, degraded retries, and Celery retries
- uncontrolled retry recursion is rejected
- duplicate unresolved-index fanout is detected and bounded
- every retry attempt emits lineage telemetry
- retry lineage persists to evidence

Required evidence:

```text
ci_evidence/production/runtime/retry_lineage.json
ci_evidence/production/runtime/fallback_resolution_report.md
ci_evidence/production/runtime/retry_amplification_matrix.json
```

Failure condition: any benchmark with unresolved retry amplification is scientifically untrustworthy and cannot pass acceptance.

## Gate B - Partial-Batch Failure Governance

Pass criteria:

- successful crops survive partial batch failure
- unresolved items are isolated
- failed items are explicitly tagged
- per-item retry isolation is enforced
- duplicate dispatch ratio remains below configured ceiling
- frame-wide invalidation due to one crop failure is not observed

Required evidence:

```text
ci_evidence/production/wave4/pose_batch_partial_success.json
ci_evidence/production/runtime/fallback_resolution_report.md
```

Failure condition: silent partial-success degradation or undisclosed single-fallback execution fails acceptance.

## Gate C - GPU Underutilization And Latency Attribution

Pass criteria:

- GPU occupancy measured
- GPU duty cycle measured
- CPU orchestration overhead measured
- queue wait contribution measured
- Python scheduling overhead measured
- serialization overhead measured
- Triton queue delay measured
- model execution time measured
- fallback-path ratio measured
- idle GPU windows identified
- latency decomposition reconciles to wall latency

Required evidence:

```text
ci_evidence/production/runtime/gpu_utilization_analysis.md
ci_evidence/production/runtime/latency_decomposition.json
ci_evidence/production/runtime/orchestration_overhead_report.md
```

Failure condition: low GPU utilization with high wall latency invalidates throughput claims unless root-cause attribution is attached.

## Gate D - Workflow Integrity

Pass criteria:

- workflow state machine emits `QUEUED`, `DISPATCHED`, `PROCESSING`, `PARTIAL_FAILURE`, `FAILED`, `COMPLETED`, and `DEGRADED_COMPLETED`
- terminal transitions are atomic
- exception paths finalize terminal states
- Celery result state and DB state reconcile
- no Celery `FAILURE` remains with DB state `PROCESSING`
- reconciliation watchdog runs and reports results

Required evidence:

```text
ci_evidence/production/runtime/workflow_integrity_report.md
ci_evidence/production/runtime/terminal_state_reconciliation.json
```

Failure condition: runtime exception path bypassing status finalization blocks production readiness.

## Gate E - Triton Readiness And Mode Validation

Pass criteria:

- startup readiness graph validates active endpoint
- inactive endpoint is isolated
- mode-specific health validation passes
- Triton repository consistency passes
- `config.pbtxt` schema passes
- model warmup verification passes
- TensorRT engine lineage passes
- active mode ambiguity fails startup

Required evidence:

```text
ci_evidence/production/wave1/model_serving_health.json
ci_evidence/production/runtime/triton_repository_validation.json
ci_evidence/production/runtime/runtime_mode_validation_report.md
ci_evidence/production/wave4/rtmpose_config_validation.txt
```

Failure condition: `required_offline=True required_live=True reason=forced_triton_not_ready` fails startup and acceptance.

## Gate F - Inference Path Attribution

Pass criteria:

Every inference event includes:

```text
inference_path
batch_id
source_batch_size
effective_batch_size
retry_generation
unresolved_index_origin
```

Required paths:

```text
request -> queue -> Triton -> fallback -> persistence -> artifact -> forensic trace
```

Failure condition: benchmark artifacts without inference-path attribution cannot support maturity claims.

## Gate G - Benchmark Causality

Pass criteria:

- benchmark report contains timestamp-correlated failure matrix
- retry spikes correlate with request lineage
- timeout spikes correlate with Triton and queue metrics
- GPU idle windows correlate with orchestration attribution
- fallback amplification is quantified
- runtime mode and profile are attached to every benchmark run

Required evidence:

```text
ci_evidence/production/runtime/benchmark_causality_report.md
ci_evidence/production/runtime/retry_amplification_matrix.json
ci_evidence/production/runtime/timeout_correlation_report.md
```
