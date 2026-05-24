# Phase 5 Protocol Decision Harness (HTTP vs gRPC)

`scripts/ci/protocol_decision_harness.py` enforces the Phase 5 rollout gate for protocol selection.

## Gate Rule
- gRPC is selected only if both are true versus HTTP baseline:
- Throughput improvement >= `10%` (default, configurable).
- Latency improvement >= `10%` using `p99` when available, else `p95` fallback.
- If threshold checks fail, gate fails and recommendation stays HTTP.

## Inputs
Use either artifact files or executable commands per protocol.

1. Artifact mode:
```bash
python scripts/ci/protocol_decision_harness.py \
  --http-artifact scripts/ci/specs/triton_phase5_http_artifact_example.json \
  --grpc-artifact scripts/ci/specs/triton_phase5_grpc_artifact_example.json
```

2. Command mode (command must print JSON object to stdout):
```bash
python scripts/ci/protocol_decision_harness.py \
  --http-command "python scripts/ci/mock_http_benchmark.py" \
  --grpc-command "python scripts/ci/mock_grpc_benchmark.py"
```

3. Mixed mode:
```bash
python scripts/ci/protocol_decision_harness.py \
  --http-artifact path/to/http_results.json \
  --grpc-command "python scripts/ci/run_grpc_benchmark.py"
```

## Output Artifacts
- Machine-readable decision report JSON:
  - default: `scripts/ci/artifacts/protocol_decision_report.json`
- Markdown summary with gate verdict:
  - default: `docs/protocol_decision_summary.md`

Override paths:
```bash
python scripts/ci/protocol_decision_harness.py \
  --http-artifact ... \
  --grpc-artifact ... \
  --decision-report scripts/ci/artifacts/phase5_decision.json \
  --summary-report docs/phase5_protocol_decision.md
```

## Recognized Metric Keys
The harness recursively scans JSON for numeric fields and accepts aliases.

- Throughput aliases:
  - `throughput`, `perf_throughput`, `infer_per_sec`, `inferences_per_second`, `requests_per_second`, `rps`, `fps`
- p99 aliases:
  - `p99`, `latency_p99`, `p99_latency`, `perf_latency_p99`
- p95 fallback aliases:
  - `p95`, `latency_p95`, `p95_latency`, `perf_latency_p95`

Notes:
- Both protocols must resolve to the same latency percentile for comparison.
- HTTP baseline throughput and latency must be > 0.

## CI Integration
- Exit code `0`: gate pass (recommend gRPC).
- Exit code `1`: gate fail (recommend HTTP).
