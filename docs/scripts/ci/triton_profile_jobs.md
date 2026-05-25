# scripts/ci/triton_profile_jobs.py

## Purpose

Phase 3 profiler automation entrypoint for deterministic Triton profiling job generation and post-run analysis.

The script now supports:
- workload-specific job specs
- Cartesian parameter sweeps across batch size, queue delay, model instance count, and client concurrency
- SLA filtering and Pareto-frontier extraction for measured results

## Determinism And Safety

- Sweep dimensions are normalized, deduplicated, and sorted before expansion.
- Workloads are expanded in lexicographic workload-name order.
- Command execution uses `subprocess.run(..., shell=False)` with tokenized args.
- No implicit randomness or timestamped output fields are produced.

## Spec Shape

```yaml
endpoint: "http://localhost:8000"
model_name: "rtmpose"
input_data: "random"   # optional, default: random

model_analyzer:
  objectives: ["perf_latency_p95", "perf_throughput"]   # optional

workloads:
  realtime:
    batch_sizes: [1]
    queue_delays_us: [0, 1000]
    instance_counts: [1]
    client_concurrency: [1, 2, 4]
  offline:
    batch_sizes: [1, 4, 8]
    queue_delays_us: [0, 5000]
    instance_counts: [1, 2]
    client_concurrency: [2, 4, 8]

analysis:
  sla:
    latency_p95_ms_max: 80
    throughput_min: 120
```

Backward compatibility:
- if `workloads` is omitted, script falls back to `perf.concurrency` and generates a `default` workload.

## Commands

Print generated profiling commands:

```powershell
python scripts/ci/triton_profile_jobs.py path/to/spec.yaml
```

Execute commands:

```powershell
python scripts/ci/triton_profile_jobs.py path/to/spec.yaml --execute
```

Analyze measured results and emit filtered output:

```powershell
python scripts/ci/triton_profile_jobs.py scripts/ci/specs/triton_phase3_profile_jobs.yaml `
  --results-in scripts/ci/specs/triton_phase3_results_in_example.yaml `
  --results-out scripts/ci/artifacts/triton_profile_filtered_results_example.json
```

If `--results-out` is omitted, output defaults to `triton_profile_filtered_results.json`.

## Results Input Contract

`--results-in` accepts YAML/JSON list records with at least:
- `workload`
- `batch_size`
- `queue_delay_us`
- `instance_count`
- `client_concurrency`
- `throughput`
- `latency_p95_ms`

## Output Contract

Generated JSON contains:
- `generated_jobs`: expanded deterministic sweep jobs
- `filtered_results.sla_pass`: rows that satisfy SLA thresholds
- `filtered_results.pareto`: non-dominated rows (maximize throughput, minimize p95 latency)
