# Triton Throughput + Steady Utilization Plan (Phased)

## Summary
- Objective: increase inference speed and make GPU usage steady (not bursty) for both live and offline workloads.
- Root issues from current state: models are effectively running at `batch=1` with single instances, offline queueing exists but scheduler-level tuning is minimal, and some Triton tuning env knobs are defined but not actively shaping execution.
- Target outcomes (vs. current baseline on the same workload): offline throughput `+30%` minimum, GPU utilization median `>=70%` with lower variance, live `p99` latency regression `<=10%`.

## Implementation Phases
1. **Phase 0 (Day 1): Baseline and Measurement Lock**  
   Goal: establish a repeatable before/after benchmark.  
   Changes: run per-model and end-to-end baselines with fixed concurrency/load, capture Triton queue/compute/request durations plus GPU util/memory/power, and freeze benchmark commands in one report.  
   Exit gate: baseline report with exact commands, load profile, and pass/fail thresholds.

2. **Phase 1 (Days 2-3): Immediate Queue/Runtime Stabilization**  
   Goal: reduce starvation gaps and improve request coalescing.  
   Changes: keep `TRITON_OFFLINE_FRAME_STRIDE=1`; tune offline queue to `max_frames=24`, `max_wait_ms=80`, `max_pending=512`, `max_concurrency=4`; enforce bounded offline job concurrency per GPU worker queue; wire currently inert runtime knobs so they are actually applied: `TRITON_DYNAMIC_BATCHING_ENABLED`, `TRITON_MAX_INFLIGHT_REQUESTS`, `TRITON_MODEL_BATCH_SIZE_OVERRIDES`.  
   Exit gate: `>=15%` throughput gain and no timeout/error regression.

3. **Phase 2 (Days 3-5): Split Live vs Offline Triton Scheduling**  
   Goal: remove one-profile compromise.  
   Changes: create dual scheduler configs per model using Triton `configs/<name>.pbtxt` pattern; run two Triton endpoints (`live` and `offline`) with `--model-config-name`.  
   Live profile defaults: `max_batch_size` up to 4 (RTMPose up to 2), preferred batches `[1,2,4]` (RTMPose `[1,2]`), queue delay `1500us` (RTMPose `800us`), instance count `1`.  
   Offline profile defaults: `max_batch_size` up to 8 (RTMPose up to 4), preferred batches `[2,4,8]` (RTMPose `[1,2,4]`), queue delay `10000us` (RTMPose `5000us`), instance count `2` for `person_detector`/`posture_model`, `1` for others.  
   Exit gate: higher GPU occupancy with stable live p99.

4. **Phase 3 (Days 5-7): Analyzer-Driven Auto-Tuning Loop**  
   Goal: empirically lock best configs instead of manual guessing.  
   Changes: use `scripts/ci/triton_profile_jobs.py` with workload-specific job specs; sweep batch sizes, queue delays, instance counts, and client concurrency; keep only Pareto-improving configs (or SLA-aligned tradeoffs).  
   Exit gate: locked config set for live/offline with measured wins and rollback snapshot.

5. **Phase 4 (Days 7-8): Spike Hardening**  
   Goal: remove cold-start and contention spikes.  
   Changes: add `model_warmup` blocks for all critical models; add Triton rate limiter priorities (live critical models higher priority); run memory pool sweep (`TRITON_PINNED_MEMORY_POOL_BYTES` and `TRITON_CUDA_MEMORY_POOL_BYTES`) and lock best non-OOM setting.  
   Exit gate: reduced first-request spikes and lower tail-latency jitter under mixed load.

6. **Phase 5 (Days 8-10): Transport Path Optimization (Conditional)**  
   Goal: reduce client-side serialization/transport overhead if still bottlenecked.  
   Changes: benchmark current HTTP path against gRPC (including streaming for live); adopt gRPC only if it improves throughput or p99 by `>=10%` at equal correctness.  
   Exit gate: protocol decision recorded and implemented only if objectively better.

## Public Interfaces / Configuration Changes
- Add workload-specific Triton endpoints: `TRITON_LIVE_URL`, `TRITON_OFFLINE_URL`.
- Add explicit runtime routing flag: `TRITON_EXECUTION_PROFILE` (`live` or `offline`) at call site/pipeline context.
- Activate and enforce: `TRITON_DYNAMIC_BATCHING_ENABLED`, `TRITON_MAX_INFLIGHT_REQUESTS`, `TRITON_MODEL_BATCH_SIZE_OVERRIDES`.
- Keep `TRITON_OFFLINE_FRAME_STRIDE=1` as fixed default for this optimization cycle.

## Test Plan
1. Unit: offline batching behavior, fallback behavior, and profile routing logic.
2. Integration: end-to-end offline job completion time and artifact correctness under tuned settings.
3. Performance: per-model and pipeline sweeps with fixed load; compare throughput, p95/p99, queue/compute/request durations, GPU utilization stability.
4. Reliability: overload test with concurrent jobs and mixed live+offline traffic; verify no deadlocks, no rising timeout trend, and bounded error rate.

## Assumptions and Defaults
- Single-GPU deployment is the primary target for this phase.
- Accuracy parity with current models is required; optimization cannot change model outputs beyond existing tolerance.
- If any phase regresses live p99 by more than `10%`, rollback to previous phase config and continue with smaller sweep ranges.
- Response cache and sequence batcher remain out of scope for this cycle unless duplicate-request or stateful-sequence evidence appears in profiling.
