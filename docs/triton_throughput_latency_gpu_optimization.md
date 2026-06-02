# Triton Optimization Strategies for Live Streaming and Offline Video Processing

**Last updated:** 2026-05-10

## Scope
This guide focuses on optimizing:
- Throughput
- Latency (especially p95/p99)
- GPU utilization

for two workloads:
- **Live streaming tasks** (latency-sensitive, steady request flow)
- **Offline video processing tasks** (throughput-oriented, batch-friendly)

It is based on Triton and NVIDIA primary documentation (links at the end).

---

## 1) Foundation: Measure Before You Tune

### Strategy
Use **Perf Analyzer** and Triton metrics to establish baselines and detect bottlenecks.

### Key knobs/tools
- `perf_analyzer --concurrency-range ...`
- `--measurement-mode=[time_windows|count_windows]`
- `--collect-metrics --metrics-url ...`
- Triton Prometheus metrics:
  - `nv_inference_queue_duration_us`
  - `nv_inference_compute_infer_duration_us`
  - `nv_inference_request_duration_us`

### Why it helps
- Separates queue bottlenecks from model compute bottlenecks.
- Prevents blind tuning.

### Tradeoffs
- Benchmarking overhead and time.
- If traffic pattern in test differs from production, conclusions can mislead.

---

## 2) Dynamic Batching (Primary Lever)

### Strategy
Enable `dynamic_batching` in `config.pbtxt` for stateless models and tune:
- `preferred_batch_size`
- `max_queue_delay_microseconds`
- queue policy / priority

### Live streaming guidance
- Keep `max_queue_delay_microseconds` small (or zero-ish) to cap tail latency.
- Use moderate preferred batch sizes to avoid frame staleness.

### Offline guidance
- Increase queue delay and preferred batch sizes to maximize GPU occupancy.
- Push concurrency until throughput plateaus or p99 violates SLA.

### Tradeoffs
- Bigger batches usually improve throughput/utilization but increase queue latency.
- Over-aggressive batching can hurt real-time smoothness in live streams.

---

## 3) Multiple Model Instances (`instance_group`)

### Strategy
Scale parallelism using multiple instances per model/GPU.

### Live streaming guidance
- Add instances for burst tolerance and smoother latency under concurrent camera streams.

### Offline guidance
- Increase instance count to maximize parallel frame processing.

### Tradeoffs
- More instances increase memory pressure and context-switch overhead.
- Past a point, throughput can flatten or regress due to contention.

---

## 4) Sequence Batcher for Stateful Flows

### Strategy
For stateful or per-stream temporal models, use **Sequence Batcher** (instead of pure dynamic batching).

### Live streaming guidance
- Preserve stream identity/state across requests.
- Useful when model logic depends on frame history.

### Offline guidance
- Use if preserving sequence state is required; otherwise stateless batching is simpler/faster.

### Tradeoffs
- More scheduler complexity.
- Some Triton features (like response cache) are not applicable to sequence batcher paths.

---

## 5) Model Analyzer Auto-Search (Best Config Discovery)

### Strategy
Use **Model Analyzer** to search config space:
- batch sizes
- instance counts
- concurrency targets
- objective constraints (e.g., maximize throughput under p99 cap)

### Live streaming guidance
- Objective: minimize p99 + keep throughput above floor.

### Offline guidance
- Objective: maximize throughput / GPU utilization, constrain memory and p99 only if needed.

### Tradeoffs
- Profiling sweeps can be expensive and slow.
- Requires disciplined experiment setup and checkpoint hygiene.

---

## 6) Backend-Level Optimization (TensorRT and Precision)

### Strategy
Use TensorRT-optimized engines and appropriate precision (FP16/INT8 when valid).

### Live streaming guidance
- Prefer predictable low-latency engines and stable optimization profiles.

### Offline guidance
- Aggressively optimize for throughput, including larger optimization profiles.

### Tradeoffs
- Lower precision can affect accuracy.
- Engine build/profiling effort and potential model-specific constraints.

---

## 7) Warmup to Remove Cold-Start Spikes

### Strategy
Configure `model_warmup` so instances are initialized before serving production traffic.

### Live streaming guidance
- Strongly recommended to avoid first-request latency spikes when stream starts.

### Offline guidance
- Useful for batch jobs that should begin at stable speed immediately.

### Tradeoffs
- Longer model load/update time.
- Extra startup compute.

---

## 8) Response Cache (Selective)

### Strategy
Enable Triton response cache only for high request duplication workloads.

### Live streaming guidance
- Usually low benefit for unique frame-by-frame inputs.

### Offline guidance
- Possible benefit in repeated clip/chunk reprocessing scenarios.

### Tradeoffs
- Cache overhead can hurt when requests are mostly unique.
- CPU-memory-based cache limitations and scheduler limitations apply.

---

## 9) Rate Limiter for Multi-Model Stability

### Strategy
Use Triton rate limiter to control cross-model resource contention.

### Live streaming guidance
- Prioritize latency-critical models (e.g., person detection) over secondary analytics during overload.

### Offline guidance
- Prioritize throughput-heavy stages while preventing OOM in mixed-model pipelines.

### Tradeoffs
- Can reduce peak throughput if configured too conservatively.
- Requires policy tuning by workload criticality.

---

## 10) Memory & I/O Path Tuning

### Strategy
Tune server memory pools and transfer paths:
- pinned memory pool size
- CUDA memory pool size
- efficient request transport and batching shape consistency

### Live streaming guidance
- Reduce transfer jitter to stabilize latency.

### Offline guidance
- Keep large data flow efficient for sustained throughput.

### Tradeoffs
- Over-allocation can starve other services.
- Under-allocation increases transfer overhead and queueing.

---

## 11) Protocol and Client Load Pattern

### Strategy
Benchmark HTTP vs gRPC (and gRPC streaming where appropriate), plus realistic client concurrency/rate models.

### Live streaming guidance
- gRPC streaming can reduce overhead in continuous streams.

### Offline guidance
- Use whichever path yields better aggregate throughput at target batch/concurrency.

### Tradeoffs
- Operational complexity (clients, observability, compatibility).
- Gains depend on payload sizes and network topology.

---

## 12) Workload-Specific Playbooks

## Live Streaming Playbook (Low Latency First)
1. Start with dynamic batching enabled, low queue delay.
2. Use enough instances to handle expected concurrent streams.
3. Use sequence batcher if stream state is required.
4. Warmup all critical models.
5. Use rate limiter priorities for graceful overload behavior.
6. Track p95/p99 queue and request durations continuously.

### Live tradeoff summary
- You intentionally sacrifice some max throughput to protect tail latency and real-time behavior.

## Offline Video Playbook (Throughput First)
1. Increase preferred batch sizes and queue delay.
2. Raise model instance counts until throughput saturates.
3. Use Model Analyzer for objective-driven search.
4. Optimize TensorRT precision/profiles for throughput.
5. Tune memory pools and client load for sustained high GPU occupancy.
6. Use rate limiter only if multi-model contention causes instability.

### Offline tradeoff summary
- You accept higher per-request latency to maximize total processed frames/sec and GPU utilization.

---

## 13) Practical Decision Matrix

| Situation | Prefer | Avoid |
|---|---|---|
| Strict real-time SLA (live) | small queue delay, moderate batch, priority control | large queue delays / oversized preferred batches |
| GPU underutilized (offline) | larger dynamic batches, more instances, analyzer sweeps | single-instance, no batching |
| Tail latency spikes | warmup + queue tuning + contention control | tuning throughput only |
| Frequent duplicate requests | response cache | cache for unique frame streams |
| Multi-model OOM/contention | rate limiter + instance rebalance | unconstrained parallel execution |

---

## 14) Validation Checklist (Per Change)

For each config change:
1. Record baseline throughput, p95/p99 latency, GPU util, GPU memory.
2. Change one variable at a time (batch size, queue delay, instance count, etc.).
3. Re-run Perf Analyzer with same load mode and measurement mode.
4. Confirm effect on:
   - `queue_duration` (scheduler pressure)
   - `compute_infer_duration` (model speed)
   - end-to-end request latency
5. Keep only Pareto-improving configs (or SLA-aligned tradeoffs).

---

## References (Primary Sources)
- Triton Dynamic Batching & Concurrent Execution tutorial:  
  https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/tutorials/Conceptual_Guide/Part_2-improving_resource_utilization/README.html
- Triton Batcher (Dynamic/Sequence):  
  https://docs.nvidia.com/deeplearning/triton-inference-server/archives/triton-inference-server-2630/user-guide/docs/user_guide/batcher.html
- Triton Model Configuration (instance groups, warmup, response cache):  
  https://docs.nvidia.com/deeplearning/triton-inference-server/archives/triton-inference-server-2630/user-guide/docs/user_guide/model_configuration.html
- Triton Perf Analyzer measurement modes/metrics:  
  https://docs.nvidia.com/deeplearning/triton-inference-server/archives/triton-inference-server-2610/user-guide/docs/perf_analyzer/docs/measurements_metrics.html
- Triton Perf Analyzer CLI:  
  https://docs.nvidia.com/deeplearning/triton-inference-server/archives/triton-inference-server-2591/user-guide/docs/perf_analyzer/docs/cli.html
- Triton Metrics reference:  
  https://docs.nvidia.com/deeplearning/triton-inference-server/archives/triton-inference-server-2591/user-guide/docs/user_guide/metrics.html
- Triton Model Analyzer CLI/objectives:  
  https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/model_analyzer/docs/cli.html
- Triton Response Cache:  
  https://docs.nvidia.com/deeplearning/triton-inference-server/archives/triton-inference-server-2620/user-guide/docs/user_guide/response_cache.html
- Triton Rate Limiter:  
  https://docs.nvidia.com/deeplearning/triton-inference-server/archives/triton-inference-server-2350/user-guide/docs/user_guide/rate_limiter.html
- NVIDIA TensorRT Performance Best Practices:  
  https://docs.nvidia.com/deeplearning/tensorrt/10.16.1/performance/best-practices.html

