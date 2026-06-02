# Hybrid Runtime Inference Adventure (Arguing_004.mp4)

**Last updated:** 2026-05-22

## Scope
Use the same offline test video (`E:\grad_project\Raw Data\Arguing Students only\Arguing_004.mp4`) with a **hybrid runtime** strategy by splitting model execution across:
- **NVIDIA GPU** (TensorRT path)
- **Intel GPU** (OpenVINO GPU runtime path)

Goal: reduce end-to-end wall time and lower timeout pressure while preserving full artifact correctness.

## Baseline From Previous Run
Job: `e2e83ad0-7e15-4821-8a91-5f8d18611ade`

| Metric | Value |
|---|---:|
| Total wall time | `757,905.83 ms` (~12.63 min) |
| Decode total | `114.82 ms` |
| Preprocess total | `2,862.37 ms` |
| Inference total | `562,497.47 ms` |
| Postprocess total | `25,735.63 ms` |

Observation: inference dominates runtime. Logs also showed repeated `TRITON_TIMEOUT` for `person_detector` and `posture_model` during frame-level loop.

## Latest Full-Frame Validation Run (2026-05-22)
Job: `3a5989c0-f8fc-4276-880a-050ca6d3fbc2`  
Command:

```powershell
.\.venv\Scripts\python.exe backend/manage.py benchmark_offline_video_job --video "E:\grad_project\Raw Data\Arguing Students only\Arguing_004.mp4" --pipeline-mode full_frame
```

| Metric | Baseline (`e2e83ad0...`) | Latest (`3a5989c0...`) | Delta |
|---|---:|---:|---:|
| Job status | `completed` | `completed` | same |
| Total wall time (DB duration) | `835,716.724 ms` (~13.93 min) | `637,069.179 ms` (~10.62 min) | `-198,647.545 ms` (~23.8% faster) |
| Decode total | `114.82 ms` | `0.00 ms`* | N/A |
| Preprocess total | `2,862.37 ms` | `0.00 ms`* | N/A |
| Inference total | `562,497.47 ms` | `0.00 ms`* | N/A |
| Postprocess total | `25,735.63 ms` | `0.00 ms`* | N/A |

\* Latest audit payload schema did not include `stage_timings_ms` totals for this run path; wall-time remains valid.

### Acceptance Criteria Verification (Pass)
- Pose data present: `pose_record_count = 125`.
- Pose output video rendered: `pose_output_video.mp4` exists.
- Final output video rendered: `final_output_video.mp4` exists.
- JSON artifacts fully exported and parseable:
  - `inference_audit.json`
  - `pose_results.json`
  - `pose_per_student.json`
  - `pose_quality_summary.json`

## Model Inventory Available
Repository currently includes these models:
- `person_detector`
- `posture_model`
- `gaze_horizontal_model`
- `gaze_depth_model`
- `gaze_vertical_model`
- `rtmpose_model`
- `rtmpose_onnx`

`v2_probe_report.json` indicates high feasible batch ceilings for most models, so queueing/concurrency placement is the practical bottleneck now.

## Hybrid Placement Decision

| Model | Primary Device | Secondary/Fallback | Decision Rationale |
|---|---|---|---|
| `person_detector` | NVIDIA GPU (TensorRT) | Intel GPU OpenVINO | Core gating model for all downstream tasks. Seen in timeout logs; keep on fastest accelerator with mature TRT path and strict SLA. |
| `posture_model` | NVIDIA GPU (TensorRT) | Intel GPU OpenVINO | Also showed timeout pressure. Keep with detector on NVIDIA to minimize additional routing overhead and maintain low-latency primary path. |
| `rtmpose_model` / `rtmpose_onnx` | NVIDIA GPU (TensorRT/ONNX runtime on NVIDIA path) | Intel GPU OpenVINO | Pose is compute-heavy and quality-sensitive. Prefer NVIDIA for stable per-person latency and keypoint quality consistency. |
| `gaze_horizontal_model` | Intel GPU (OpenVINO) | NVIDIA GPU | Lower criticality than detector/pose; high call frequency makes it ideal offload target to reduce NVIDIA contention. |
| `gaze_depth_model` | Intel GPU (OpenVINO) | NVIDIA GPU | Same as above; offloading reduces queue depth and timeout risk on NVIDIA. |
| `gaze_vertical_model` | Intel GPU (OpenVINO) | NVIDIA GPU | Same offload pattern; vertical gaze is a good candidate for Intel GPU throughput path. |

## Why This Split Is Best For Us (Current Evidence)

| Constraint / Signal | Implication | Hybrid Action |
|---|---|---|
| Inference occupies the majority of wall time | Optimize compute placement first | Split workload by criticality and cost |
| Timeouts observed in `person_detector` and `posture_model` | NVIDIA contention/latency pressure on critical path | Reserve NVIDIA for detector + posture + pose first |
| Gaze models are frequent and parallelizable | Good candidates for separate execution lane | Move all gaze models to Intel OpenVINO GPU lane |
| Pose quality is currently sensitive | Avoid extra uncertainty in first hybrid pass | Keep pose on NVIDIA until pose stability KPI is green |

## OpenVINO Artifacts Required (All Models)
Generate OpenVINO IR (`.xml` + `.bin`) for every model so we can route flexibly and fallback safely.

| Model | Source Artifact | Required OpenVINO Artifacts | Priority |
|---|---|---|---|
| `person_detector` | `model.onnx` | `model.xml`, `model.bin` | High |
| `posture_model` | `model.onnx` | `model.xml`, `model.bin` | High |
| `gaze_horizontal_model` | `model.onnx` | `model.xml`, `model.bin` | High |
| `gaze_depth_model` | `model.onnx` | `model.xml`, `model.bin` | High |
| `gaze_vertical_model` | `model.onnx` | `model.xml`, `model.bin` | High |
| `rtmpose_onnx` | `model.onnx` | `model.xml`, `model.bin` | Medium |
| `rtmpose_model` (plan) | Prefer export from ONNX source if available | `model.xml`, `model.bin` | Medium |

## Rollout Plan For The Same Video Test

| Step | Action | Pass Criteria |
|---|---|---|
| 1 | Build OpenVINO IR for all models and add Triton/OpenVINO model variants | All model variants load healthy |
| 2 | Add routing policy: detector/posture/pose -> NVIDIA, gaze trio -> Intel | Route logs confirm target device per model |
| 3 | Run same offline benchmark on `Arguing_004.mp4` | Job completes with full artifacts |
| 4 | Compare wall time and stage telemetry against baseline | Inference time reduced and timeout count reduced |
| 5 | Validate quality: detections, tracking IDs, pose keypoints, rendered video | No regression in functional outputs |

## Success KPIs

| KPI | Baseline | Target (Hybrid Pass 1) |
|---|---:|---:|
| Total wall time | ~12.63 min | <= 8-9 min |
| Inference total time | ~562.5 s | >= 25% reduction |
| Timeout count (`person/posture`) | non-zero | materially reduced |
| Pose records | previously unstable in some runs | non-zero and consistent |
| Artifact completeness | final + pose/json/audit | 100% generated |

## Notes
- This first hybrid split is intentionally conservative: keep mission-critical stages on NVIDIA, offload gaze workload to Intel.
- After Pass 1 stability, we can test moving `posture_model` to Intel as Pass 2 sensitivity experiment.
- If Intel queue builds up, cap per-model concurrency and increase queue delay slightly before changing placement.
