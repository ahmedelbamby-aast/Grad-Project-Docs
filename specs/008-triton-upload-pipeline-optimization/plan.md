# Implementation Plan: Triton Upload Pipeline Optimization

## Goal
Deliver end-to-end upload inference throughput gains by implementing:
- Robust Triton request batching for upload jobs.
- Higher and more stable NVIDIA GPU utilization under concurrent uploads.
- Tuned Celery worker/queue topology to keep Triton saturated without overloading DB/Redis.

## Scope
- Upload/offline inference path only (video upload pipeline).
- Triton-backed models used in upload processing (detector, pose, posture, gaze family).
- Celery routing and worker execution model for upload tasks.
- Observability and load-test evidence for throughput/latency/GPU utilization.

## Out of Scope
- Live RTSP pipeline behavior changes.
- UI/UX changes beyond surfacing existing processing status.
- Non-Triton inference backend optimizations.

## Target Outcomes
- Increase effective batch size processed per Triton request on upload workloads.
- Improve sustained GPU utilization during upload backlogs.
- Reduce queue wait and end-to-end upload processing time at equal hardware.
- Maintain correctness parity with current inference outputs.

## Architecture Strategy
1. Split queue intent and worker pools:
- Dedicated high-throughput upload inference queue.
- Optional shadow-precheck queue with bounded concurrency.
- Keep lightweight/non-GPU tasks isolated from heavy inference workers.

2. Introduce explicit batching stages:
- Frame staging and micro-batch assembly with max size + max wait.
- Per-model/per-request timeout and retry policy compatible with batch flow.
- Backpressure controls to prevent unbounded pending frames.

3. Align Triton and worker concurrency:
- Match Celery process/thread concurrency to GPU service capacity.
- Tune inflight request counts to avoid host-side queue thrash.
- Use queue depth + Triton latency + GPU utilization feedback loop.

## Workstreams
### WS1 - Baseline and Instrumentation
- Capture baseline throughput/latency/GPU metrics under representative upload concurrency.
- Confirm current Celery queue routing and worker process layout.
- Add/verify metrics emission and log correlation IDs for upload jobs.

### WS2 - Triton Batching Implementation
- Implement/complete upload micro-batching controls (max frames, wait window, pending caps, concurrency caps).
- Ensure model invocation path supports batched payloads where applicable.
- Add guardrails for timeout/retry/fallback behavior.

### WS3 - Worker and Queue Tuning
- Define queue names and routing for upload inference vs precheck/other tasks.
- Tune Celery worker counts, per-queue concurrency, prefetch, and ack strategy.
- Validate no starvation for non-upload background jobs.

### WS4 - Validation and Rollout
- Execute staged tests: smoke, contract-relevant pipeline checks, parallel upload load tests.
- Compare baseline vs optimized metrics and document deltas.
- Prepare rollout toggles and safe rollback path through env knobs.

## Configuration Contract
All tuning switches must be env-driven with safe defaults in `.env.example`, including:
- Shadow precheck concurrency cap.
- Celery queue/concurrency/prefetch/task-routing knobs.
- Triton batch queue and inflight/concurrency knobs.

## Risks and Mitigations
- Over-batching increases tail latency:
  - Mitigation: enforce max-wait windows and timeout ceilings.
- Worker oversubscription causes host contention:
  - Mitigation: queue isolation + bounded concurrency + monitored rollout.
- GPU underutilization persists due to upstream bottlenecks:
  - Mitigation: instrument queue wait, decode time, and request assembly time separately.

## Acceptance Criteria
- Upload pipeline runs fully on Triton batching path with env-configurable controls.
- Celery queues/workers are explicitly tuned and documented for upload workload.
- Evidence package includes before/after throughput, latency, and GPU utilization snapshots.
- No regression in core upload correctness checks.

