# Contract: Production Helper Commands

**Version**: `scene-prod-helpers.v1`  
**Feature**: `014-yoloe-scene-srvl`

Production helpers must run on the native Linux production server without
Docker and without `sudo`. Windows/PowerShell variants may exist for contract
checks, but acceptance is Linux-only.

## prod_export_yoloe_scene_model

Purpose: download/verify `yoloe-26s-seg.pt`, resolve prompt classes, call
`model.set_classes(...)`, export ONNX/TensorRT, and write the export manifest.

Required inputs:

```text
YOLOE_SCENE_PROFILE=classroom_roi_guard_v1
YOLOE_SCENE_PROMPT_CLASSES=<optional ordered override>
YOLOE_SCENE_OUTPUT_CAPTURE_MODE=summary_and_artifact_refs
YOLOE_SCENE_CONFIDENCE_THRESHOLD=<declared numeric value>
YOLOE_SCENE_CONTRADICTION_OVERLAP=<declared numeric value>
YOLOE_SCENE_CHECKPOINT_URL=https://github.com/ultralytics/assets/releases/download/v8.4.0/yoloe-26s-seg.pt
YOLOE_SCENE_EXPORT_FORMAT=onnx_then_tensorrt
YOLOE_SCENE_ONNX_FORMAT=onnx
```

Required outputs:

- Prompt digest.
- Full normalized env/settings snapshot and fingerprint.
- ONNX/TensorRT artifact paths.
- Artifact SHA-256 digests.
- Export manifest JSON.
- Export timing and package versions.

Failure must be fail-closed when prompts are missing, prompt digest is stale,
checkpoint is wrong, or export validation fails.

## prod_verify_yoloe_scene_export

Purpose: verify manifest/runtime compatibility before a benchmark or service
start.

Checks:

- Runtime `.env` prompt digest equals manifest prompt digest.
- Every operational scene env key is declared in the tracked env contract.
- No helper-local default overrides backend/frontend settings.
- Checkpoint identity matches V1 requirement.
- TensorRT/ONNX artifacts exist and match digests.
- Model route and Triton state are ready.
- Live profile is disabled for V1.

## prod_run_yoloe_scene_benchmark

Purpose: run baseline/candidate `combined.mp4` benchmark with scene flags and
record durable evidence.

Required behavior:

- Use replay policy `new-attempt`.
- Record baseline and candidate job IDs.
- Record exact video path and digest when available.
- Capture FPS, step timings, model RTT, GPU, VRAM, CPU/RSS, PostgreSQL, Redis,
  frontend, artifact size, correctness, non-ROI, mismatch, and unavailable
  metric reasons.
- Exit nonzero on missing mandatory evidence.

## prod_collect_yoloe_scene_metrics.py

Purpose: normalize raw logs/CSV/JSON/DB queries into one metrics JSON and one
Markdown summary.

Required output keys:

```json
{
  "benchmark_key": "string",
  "job_id": "uuid",
  "video_path": "string",
  "git_sha": "string",
  "env_fingerprint": "sha256:...",
  "throughput": {},
  "latency": {},
  "gpu": {},
  "cpu_memory": {},
  "postgresql": {},
  "redis": {},
  "frontend": {},
  "scene": {},
  "srvl": {},
  "correctness": {},
  "configuration": {
    "declared_env_contract": "scene-env.v1",
    "hardcoded_operational_values_detected": false
  },
  "unavailable_metrics": []
}
```

## prod_generate_yoloe_scene_figures.py

Purpose: create benchmark plots and a figure manifest from the same raw inputs
used by the decision table.

Required figures:

- Baseline vs candidate FPS/latency.
- YOLOE and SRVL stage timing.
- GPU utilization/memory and unavailable GPU metrics.
- PostgreSQL/Redis timing.
- Frontend FPS/frame time and dropped visualization frames.
- Artifact size by type.
- Mismatch recovery and non-ROI contradiction counts.
- Confidence distributions and threshold overlays for YOLOE persons and
  non-ROI objects.

## prod_benchmark_scene_renderers

Purpose: compare PixiJS WebGL first against any later renderer candidates using
the same representative payloads.

Required metrics:

- FPS and frame time percentiles.
- Main-thread long tasks.
- Payload decode/transfer time.
- WebGL context loss.
- Memory growth.
- Screenshots for desktop and constrained viewport.

## prod_rollback_yoloe_scene

Purpose: disable scene features and prove current offline behavior is restored.

Required flag reset:

```text
YOLOE_SCENE_ENABLED=0
YOLOE_SCENE_MISMATCH_RECOVERY=0
YOLOE_SCENE_RECOVERY_REPASS=0
SRVL_ENABLED=0
SCENE_RENDER_BENCHMARK_ENABLED=0
```

Required proof:

- Runtime flags inspected.
- Triton route no longer used by scene lane.
- Existing offline job path completes.
- Existing artifacts/playback remain valid.
- Active `.env` flags and threshold keys match the rollback env contract.
- Rollback evidence stored with command output and digests.
