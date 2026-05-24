# Production Readiness Checklist - Runtime Remediation

## Purpose

This checklist defines release-blocking runtime remediation checks required before behavioral intelligence maturity closure can be claimed.

## Runtime Mode And Triton Readiness

- [ ] `TRITON_EXECUTION_MODE` is set to exactly one valid mode.
- [ ] Active Triton endpoint health passes for the selected mode.
- [ ] Inactive Triton endpoint receives no production inference traffic.
- [ ] Startup fails on active-mode ambiguity.
- [ ] Startup fails on `required_offline=True required_live=True reason=forced_triton_not_ready`.
- [ ] Triton repository consistency check passes.
- [ ] `config.pbtxt` schema validation passes.
- [ ] RTMPose warmup verification passes.
- [ ] TensorRT engine lineage verification passes.
- [ ] Production local fallback inference is disabled.

## Retry And Fallback Governance

- [ ] Retry taxonomy is implemented.
- [ ] Max fallback attempts are configured and finite.
- [ ] Max unresolved-index retries are configured and finite.
- [ ] Max degraded retries are configured and finite.
- [ ] Retry recursion detector is active.
- [ ] Fallback amplification detector is active.
- [ ] Every retry attempt emits request lineage.
- [ ] `ci_evidence/production/runtime/retry_lineage.json` exists and is complete.
- [ ] `ci_evidence/production/runtime/fallback_resolution_report.md` exists and discloses fallback behavior.

## Partial-Batch Governance

- [ ] Successful batch items survive partial failures.
- [ ] Unresolved indexes are isolated.
- [ ] Failed items are explicitly tagged.
- [ ] Per-item retry isolation is enforced.
- [ ] Duplicate dispatch ratio is measured and within configured ceiling.
- [ ] Partial-batch fallback execution is disclosed in benchmark evidence.

## Workflow Integrity

- [ ] Workflow states include `QUEUED`, `DISPATCHED`, `PROCESSING`, `PARTIAL_FAILURE`, `FAILED`, `COMPLETED`, and `DEGRADED_COMPLETED`.
- [ ] Terminal transitions are atomic.
- [ ] Runtime exception paths finalize DB job state.
- [ ] Celery `FAILURE` with DB `PROCESSING` is absent.
- [ ] Reconciliation watchdog is scheduled.
- [ ] `ci_evidence/production/runtime/workflow_integrity_report.md` exists.
- [ ] `ci_evidence/production/runtime/terminal_state_reconciliation.json` exists.

## GPU Utilization And Latency Attribution

- [ ] GPU occupancy is measured.
- [ ] GPU duty cycle is measured.
- [ ] CPU orchestration overhead is measured.
- [ ] Queue wait contribution is measured.
- [ ] Python scheduling overhead is measured.
- [ ] Serialization overhead is measured.
- [ ] Triton queue delay is measured.
- [ ] Model execution time is measured.
- [ ] Fallback-path ratio is measured.
- [ ] Idle GPU windows are reported.
- [ ] `ci_evidence/production/runtime/latency_decomposition.json` reconciles to wall latency.
- [ ] Low GPU utilization with high wall latency has root-cause attribution.

## Benchmark Causality

- [ ] Benchmark report includes runtime failure matrix.
- [ ] Retry spikes are correlated with retry lineage.
- [ ] Timeout spikes are correlated with Triton/queue/GPU telemetry.
- [ ] Queue collapse windows are identified or absent.
- [ ] GPU idle windows are explained.
- [ ] Fallback amplification is quantified.
- [ ] Benchmark results with unresolved retry amplification are rejected.
- [ ] `ci_evidence/production/runtime/benchmark_causality_report.md` exists.
- [ ] `ci_evidence/production/runtime/retry_amplification_matrix.json` exists.
- [ ] `ci_evidence/production/runtime/timeout_correlation_report.md` exists.

## Scientific Claim Safety

- [ ] Throughput claims include GPU and orchestration attribution.
- [ ] Partial-batch fallback is disclosed.
- [ ] Silent degraded execution is absent.
- [ ] Benchmark acceptance distinguishes mock vs real execution.
- [ ] Benchmark acceptance distinguishes CPU vs GPU execution.
- [ ] Benchmark acceptance distinguishes live vs offline mode.
- [ ] Evidence package includes request lineage IDs and retry lineage IDs.
