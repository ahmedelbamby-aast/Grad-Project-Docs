# Triton Phase 0 Baseline Lock Report (Skeleton)

Generated: 2026-05-22T11:23:09.952936+00:00
Host: DESKTOP-LN61TNQ
Python: 3.12.0

## Scope
- Phase: 0 (Baseline and Measurement Lock)
- Source plan: `docs/triton_inference_speed_stabilization_plan.md`
- Mode: fixed command lines + threshold placeholders for repeatable before/after checks

## Fixed Load Profile
- Triton URL: `localhost:8000`
- Protocol: `http`
- Perf Analyzer measurement interval: `10000` ms
- GPU telemetry window: `300` seconds
- Concurrency strategy:
  - Live: low-latency ranges (1..4, RTMPose 1..2)
  - Offline: throughput ranges (2..8, RTMPose 1..4)

## Locked Command Set
- `person_detector_live` (per-model/live)
  - `perf_analyzer -m person_detector -u localhost:8000 --concurrency-range 1:4:1 --protocol http --measurement-interval 10000 --measurement-mode time_windows --percentile 95 --collect-metrics`
- `posture_model_live` (per-model/live)
  - `perf_analyzer -m posture_model -u localhost:8000 --concurrency-range 1:4:1 --protocol http --measurement-interval 10000 --measurement-mode time_windows --percentile 95 --collect-metrics`
- `rtmpose_live` (per-model/live)
  - `perf_analyzer -m rtmpose -u localhost:8000 --concurrency-range 1:2:1 --protocol http --measurement-interval 10000 --measurement-mode time_windows --percentile 95 --collect-metrics`
- `person_detector_offline` (per-model/offline)
  - `perf_analyzer -m person_detector -u localhost:8000 --concurrency-range 2:8:2 --protocol http --measurement-interval 10000 --measurement-mode time_windows --percentile 95 --collect-metrics`
- `posture_model_offline` (per-model/offline)
  - `perf_analyzer -m posture_model -u localhost:8000 --concurrency-range 2:8:2 --protocol http --measurement-interval 10000 --measurement-mode time_windows --percentile 95 --collect-metrics`
- `rtmpose_offline` (per-model/offline)
  - `perf_analyzer -m rtmpose -u localhost:8000 --concurrency-range 1:4:1 --protocol http --measurement-interval 10000 --measurement-mode time_windows --percentile 95 --collect-metrics`
- `model_analyzer_person_detector` (sweep/per-model)
  - `model-analyzer profile -m person_detector --triton-launch-mode=remote --triton-http-endpoint localhost:8000 --objective perf_throughput,perf_latency_p95`
- `model_analyzer_posture_model` (sweep/per-model)
  - `model-analyzer profile -m posture_model --triton-launch-mode=remote --triton-http-endpoint localhost:8000 --objective perf_throughput,perf_latency_p95`
- `model_analyzer_rtmpose` (sweep/per-model)
  - `model-analyzer profile -m rtmpose --triton-launch-mode=remote --triton-http-endpoint localhost:8000 --objective perf_throughput,perf_latency_p95`
- `gpu_telemetry` (host-metrics)
  - `nvidia-smi --query-gpu=timestamp,utilization.gpu,utilization.memory,memory.used,power.draw --format=csv -l 1 > triton_phase0_gpu_metrics.csv`
- `pipeline_e2e_placeholder` (end-to-end)
  - `# Replace with project-specific fixed-load E2E command, e.g. replay script or pytest perf scenario`

## Threshold Placeholders (Exit Gate)
- Offline throughput gain vs baseline: `+30%` minimum
- GPU utilization median: `>=70%` with lower variance
- Live p99 latency regression vs baseline: `<=10%`

## Measurement Capture Fields
- Triton request durations: request, queue, compute
- Throughput and latency: infer/sec, p95, p99
- GPU telemetry: utilization.gpu, utilization.memory, memory.used, power.draw

## Run Results
- Dry run only (no command outputs collected).

## Fill-In Summary Template
- Baseline run id: `<fill>`
- Candidate run id: `<fill>`
- Offline throughput delta (%): `<fill>`
- Live p99 delta (%): `<fill>`
- GPU util median/variance delta: `<fill>`
- Pass/Fail decision: `<fill>`
- Notes and anomalies: `<fill>`
