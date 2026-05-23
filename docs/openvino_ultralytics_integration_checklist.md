# OpenVINO + Ultralytics Integration Safety Checklist

## Purpose
This checklist reduces integration failures when running Ultralytics-exported models with OpenVINO in this repository.

## Required Practices

| Area | Rule | Why |
|---|---|---|
| Export format | Keep OpenVINO IR artifacts as paired `model.xml` + `model.bin` per model version | OpenVINO runtime expects IR pair for deterministic loading |
| Source of truth | Keep source ONNX and validate it before regenerating IR | Prevent propagating corrupted exports |
| Device target | Validate all IR on `intel:gpu` before rollout | Ensures runtime behavior matches deployment device |
| Static shapes | Convert with explicit model-specific input shapes | Avoid dynamic-shape compile/reshape errors |
| Runtime hint | Use `PERFORMANCE_HINT=THROUGHPUT` for offline batch workloads | Matches OpenVINO guidance for throughput-oriented jobs |
| Cache | Use `CACHE_DIR` for compiled model caching | Reduces compile overhead and startup variance |
| Mixed strategy | Keep strategy env-controlled (`triton_only`, `openvino_only`, `hybrid`) | Avoid hidden routing behavior and operator confusion |
| Mock inference gate | Run mock inference for all IR models before end-to-end jobs | Catches device/plugin/model incompatibility early |

## Repository Scripts

| Script | Purpose |
|---|---|
| `backend/scripts/validate_regen_openvino_v2.py` | Validate source ONNX + Intel GPU IR compile, regenerate only broken IR models |
| `backend/scripts/openvino_mock_infer_all_v2.py` | Compile and run mock inference on all OpenVINO v2 models on Intel GPU |
| `backend/scripts/openvino_v2_gpu_sweep.py` | Throughput sweep (`nstreams`, `nireq`) for tuned configs |

## Recommended Preflight Sequence

1. `python backend/scripts/validate_regen_openvino_v2.py --device intel:gpu`
2. `python backend/scripts/openvino_mock_infer_all_v2.py --device intel:gpu`
3. Optional tuning pass: `python backend/scripts/openvino_v2_gpu_sweep.py`
4. Run the target offline benchmark job.

## References

- Ultralytics OpenVINO integration docs: https://docs.ultralytics.com/integrations/openvino
- Ultralytics export docs: https://docs.ultralytics.com/modes/export/
- Ultralytics OpenVINO backend reference: https://docs.ultralytics.com/reference/nn/backends/openvino/
- OpenVINO Python inference API: https://docs.openvino.ai/2023.3/openvino_docs_OV_UG_Python_API_inference.html
- OpenVINO `CompiledModel` API: https://docs.openvino.ai/2026/api/ie_python_api/_autosummary/openvino.CompiledModel.html
- OpenVINO performance hints: https://docs.openvino.ai/2023.3/openvino_docs_OV_UG_Performance_Hints.html

## Current Validation Snapshot

- Mock inference device: `intel:gpu` (`Intel(R) Iris(R) Xe Graphics (iGPU)`)
- Models tested: 7/7 passed
- Latest report: `docs/openvino_mock_inference_report.json`
