# Tasks: Triton Upload Pipeline Optimization

## T0 - Setup and Baseline
- [x] T0.1 Record current upload throughput/latency baseline under controlled parallel uploads.
- [x] T0.2 Capture baseline GPU telemetry (utilization, memory, power, temperature).
- [x] T0.3 Capture current Celery queue depth/worker utilization snapshot.
- [x] T0.4 Save baseline commands, environment values, and commit SHA in evidence doc.

## T1 - Configuration Surface (Env + Parsing)
- [x] T1.1 Add/confirm all required env knobs in `.env.example` for Triton batching and worker tuning.
- [x] T1.2 Wire new env knobs into settings/config parsing with type validation and defaults.
- [x] T1.3 Add startup logs that print resolved tuning values for traceability (without secrets).

## T2 - Upload Batching Pipeline
- [x] T2.1 Implement/verify upload micro-batch builder with `max_frames` and `max_wait_ms`.
- [x] T2.2 Enforce bounded pending queue for upload batch buffer to avoid memory growth.
- [x] T2.3 Add bounded inflight batch request concurrency.
- [x] T2.4 Ensure per-model batch path uses Triton request payloads compatible with dynamic batching.
- [x] T2.5 Add timeout + retry policy for batch requests with clear failure accounting.
- [x] T2.6 Add fallback/skip behavior definition when batch request finally fails (no silent drops).

## T3 - Celery Queue and Worker Topology
- [x] T3.1 Define dedicated queue names for upload inference and shadow precheck.
- [x] T3.2 Route upload tasks explicitly to upload inference queue.
- [x] T3.3 Route optional shadow precheck tasks to dedicated queue with capped concurrency.
- [x] T3.4 Tune worker concurrency/prefetch for GPU-bound workload.
- [x] T3.5 Validate ack/retry semantics to prevent duplicate heavy inference on worker restarts.

## T4 - Observability and Diagnostics
- [x] T4.1 Add/verify metrics for batch size distribution, batch wait time, and queue wait time.
- [x] T4.2 Add/verify metrics for Triton request duration, timeout count, retry count, and failure count.
- [x] T4.3 Add/verify queue-level metrics by task type and worker pool.
- [x] T4.4 Correlate upload job ID across queue, batcher, and Triton request logs.

## T5 - Verification and Performance Validation
- [x] T5.1 Run smoke tests for upload inference correctness on known samples.
- [x] T5.2 Run backend parallel tests for impacted suites (`contract`, targeted upload/inference tests).
- [x] T5.3 Run staged upload load tests at low/medium/high concurrency.
- [x] T5.4 Compare before/after: throughput, P50/P95 latency, GPU utilization, error rate.
- [x] T5.5 Validate no starvation/regression on non-upload queues.

## T6 - Rollout and Guardrails
- [x] T6.1 Define recommended production default values per hardware tier (single GPU vs multi-GPU).
- [x] T6.2 Document rollback profile using env toggles (disable batching / reduce concurrency).
- [x] T6.3 Add release checklist entry for post-deploy metric watch window.

## Suggested Execution Order
1. T0 baseline
2. T1 configuration
3. T2 batching implementation
4. T3 queue/worker tuning
5. T4 observability
6. T5 validation
7. T6 rollout
