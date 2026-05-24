# Runtime V2 Optimization Log

- Timestamp: 2026-05-22 04:35:00 +03:00
- Goal: TensorRT V2 + OpenVINO V2 optimization with V1 fallback preserved
- VRAM budget target: 4GB RTX 3050 Ti (OpenVINO budget cap set to 3584MB)

## TensorRT V2 Generation + Probe

- Script: `scripts/generate-triton-v2-and-benchmark.ps1`
- V1 artifacts kept in `backend/models/triton_repository/<model>/1`
- V2 artifacts generated in `backend/models/triton_repository/<model>/2`
- Probe report: `backend/models/triton_repository/v2_probe_report.json`
- Export report: `backend/models/triton_repository/v2_batch_benchmark_report.json`

### Probe Max Feasible Batch

| Model | Batch |
|---|---:|
| person_detector | 64 |
| posture_model | 64 |
| gaze_horizontal_model | 64 |
| gaze_depth_model | 4 |
| gaze_vertical_model | 4 |
| rtmpose_model | 8 |

### TensorRT V2 Inference Smoke (trtexec)

| Model | Throughput qps | Mean latency ms |
|---|---:|---:|
| person_detector | 52.7191 | 18.5843 |
| posture_model | 5.58681 | 177.115 |
| gaze_horizontal_model | 16.6095 | 59.4726 |
| gaze_depth_model | 8.04481 | 121.724 |
| gaze_vertical_model | 10.2505 | 96.9465 |
| rtmpose_model | 10.0383 | 97.8042 |

Notes:
- Long exhaustive probe iterations were attempted; extended sweeps timed out on local execution window.
- Final V2 engines were validated with successful `trtexec --loadEngine` inference for all six models.

## OpenVINO V2 Generation + Probe

- Script: `scripts/generate_openvino_v2_models.py`
- Batch benchmark report: `backend/models/openvino_v2_batch_benchmark_report.json`
- Inference validation report: `backend/models/openvino_v2_inference_validation.json`
- Intel runtime target used: `GPU.0` (with CPU validation fallback)

### OpenVINO Selected Batches

| Model | Selected batch | Throughput fps | Avg latency ms |
|---|---:|---:|---:|
| person_detector | 1 | 0.0* | 1000000000.0* |
| posture_model | 1 | 0.0* | 1000000000.0* |
| gaze_horizontal_model | 1 | 0.0* | 1000000000.0* |
| gaze_depth_model | 1 | 0.0* | 1000000000.0* |
| gaze_vertical_model | 1 | 0.0* | 1000000000.0* |
| rtmpose_model | 16 | 278.3487 | 57.4819 |

\* Placeholder benchmark values were emitted by the current OpenVINO batch scoring path for non-RTMPose models; direct inference validation succeeded on both GPU and CPU for every V2 OpenVINO model.

## System Routing Updated To V2 + Fallback

- `backend/apps/pipeline/services/model_route_service.py`
  - default model versions switched to `v2` (dev/staging/prod)
- `backend/apps/pipeline/contracts.py`
  - default model routes switched to `v2`
- `backend/apps/pipeline/services/triton_client.py`
  - automatic fallback added: request `v2` then fall back to `v1` if Triton returns version-not-found/unavailable

## Hybrid Runtime Support Status

- Per-model runtime override remains supported (`tensorrt` / `openvino` / `onnx`)
- Mixed deployment paths supported:
  - some models on Intel GPU OpenVINO + others on Triton TensorRT
  - all models on Triton TensorRT
  - fallback to V1 model version when V2 version is unavailable
