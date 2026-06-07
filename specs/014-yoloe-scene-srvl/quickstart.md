# Quickstart: YOLOE Scene Segmentation And SRVL

**Date**: 2026-06-07  
**Feature**: `014-yoloe-scene-srvl`

This quickstart defines the expected developer and production workflow after
implementation tasks add the helper scripts and runtime wiring. Commands are
contracts for reproduction; they are not acceptance evidence until they run on
the required environment and produce real artifacts.

## 1. Preflight

```powershell
git status --short
Get-Content AGENTS.md -TotalCount 80
Get-Content .specify\memory\constitution.md -TotalCount 120
```

Rules:

- Do not use SQLite for tests, migrations, benchmarks, or evidence.
- Keep `YOLOE_SCENE_ENABLED=0` and `SRVL_ENABLED=0` until wiring and rollback
  tests pass.
- Do not enable the live profile for V1.

## 2. Prompt-Locked Export

The export must apply prompts before ONNX or TensorRT export.

The helper code must read prompt classes and thresholds from `.env`/settings;
do not hardcode the class list or thresholds in export code.

```python
import os
from ultralytics import YOLOE

checkpoint = os.environ["YOLOE_SCENE_CHECKPOINT"]
prompt_classes = [
    value.strip()
    for value in os.environ["YOLOE_SCENE_PROMPT_CLASSES"].split(",")
    if value.strip()
]

if not prompt_classes:
    raise RuntimeError("YOLOE_SCENE_PROMPT_CLASSES must be declared")

model = YOLOE(checkpoint)
model.set_classes(prompt_classes)
export_model = model.export(format=os.environ["YOLOE_SCENE_ONNX_FORMAT"])
model = YOLOE(export_model)
```

Required default production benchmark `.env` profile:

```text
YOLOE_SCENE_PROFILE=classroom_roi_guard_v1
YOLOE_SCENE_PROMPT_CLASSES=person,table,desk,chair,wall,floor,ceiling,door,window,mirror,board,screen
YOLOE_SCENE_NON_ROI_CLASSES=table,desk,chair,wall,floor,ceiling,door,window,mirror,board,screen
YOLOE_SCENE_PROMPT_LOCK_REQUIRED=1
YOLOE_SCENE_CHECKPOINT=yoloe-26s-seg.pt
YOLOE_SCENE_CHECKPOINT_URL=https://github.com/ultralytics/assets/releases/download/v8.4.0/yoloe-26s-seg.pt
YOLOE_SCENE_EXPORT_FORMAT=onnx_then_tensorrt
YOLOE_SCENE_ONNX_FORMAT=onnx
YOLOE_SCENE_OUTPUT_CAPTURE_MODE=summary_and_artifact_refs
YOLOE_SCENE_PERSIST_RAW_OUTPUT_REFS=1
YOLOE_SCENE_CONFIDENCE_THRESHOLD=0.25
YOLOE_SCENE_MASK_CONFIDENCE_THRESHOLD=0.25
YOLOE_SCENE_MIN_MASK_AREA_PX=16
YOLOE_SCENE_CONTRADICTION_OVERLAP=0.40
YOLOE_SCENE_CONTRADICTION_MIN_CONFIDENCE=0.25
YOLOE_SCENE_PERSON_MASK_SUPPORT_THRESHOLD=0.25
YOLOE_SCENE_RECOVERY_CONFIDENCE_WEIGHT=0.25
YOLOE_SCENE_RECOVERY_GEOMETRY_WEIGHT=0.25
YOLOE_SCENE_RECOVERY_APPEARANCE_WEIGHT=0.50
```

If `YOLOE_SCENE_PROMPT_CLASSES` is changed in `.env`, the old ONNX/TensorRT
artifact must be treated as stale until a new export manifest is written and
validated.

YOLOE-Seg emits detection and instance segmentation together. The normalizer
must preserve boxes, class IDs/names, confidence scores, masks, shape metadata,
timing metadata, and raw-output references from the same result; masks are
expected through `results[0].masks`. If the deployed export cannot expose a
field, the trace must record an unavailable reason.

All thresholds, weights, cadences, limits, codecs, renderer choices, matrix
modes, and recovery gates used by the feature must appear in `.env` and the
tracked env contract. Source code and helper scripts must not carry independent
operational defaults.

## 3. Local Contract Checks

Expected checks after implementation adds code:

```powershell
cd E:\grad_project
uv run --project backend pytest backend/tests/unit -q
uv run --project backend pytest backend/tests/contract -q
cd frontend
npm run type-check
npm run test
npm run test:e2e
```

Documentation checks:

```powershell
cd E:\grad_project
python scripts\ci\verify_mermaid_diagrams.py --paths specs\014-yoloe-scene-srvl
python scripts\ci\verify_doc_dates_and_reading_order.py
```

Local checks validate contracts only. They cannot accept production latency,
GPU, model-route, or renderer claims.

## 4. Production Benchmark Flow

Run on the native Linux RTX 5090 production server, without Docker and without
`sudo`.

```bash
export YOLOE_SCENE_ENABLED=1
export YOLOE_SCENE_PROFILE=classroom_roi_guard_v1
export YOLOE_SCENE_MISMATCH_RECOVERY=0
export YOLOE_SCENE_RECOVERY_REPASS=0
export SRVL_ENABLED=1
export SCENE_RENDER_BACKEND=pixijs_webgl_candidate

tools/prod/prod_export_yoloe_scene_model.sh
tools/prod/prod_verify_yoloe_scene_export.sh
tools/prod/prod_run_yoloe_scene_benchmark.sh \
  --video "/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4" \
  --replay-policy new-attempt
tools/prod/prod_collect_yoloe_scene_metrics.py --latest
tools/prod/prod_generate_yoloe_scene_figures.py --latest
```

Required evidence:

- Baseline job ID and candidate job ID.
- Exact Git SHA and `.env` fingerprint.
- Export manifest and prompt digest.
- PostgreSQL/Redis/Celery/Triton runtime truth.
- FPS, latency percentiles, model RTT, per-step timings, GPU, VRAM, CPU/RSS,
  DB, Redis, frontend FPS, artifact sizes, mismatch counts, contradiction
  counts, unavailable metric reasons.
- Generated figures and figure manifest from the same raw inputs.
- Rollback proof.

## 5. Renderer Benchmark

PixiJS WebGL is the first candidate.

```bash
tools/prod/prod_benchmark_scene_renderers.sh \
  --candidate pixijs_webgl \
  --payload-root backend/logs/yoloe_scene/latest/payloads
```

Escalate to deck.gl or Three.js only after PixiJS fails measured FPS, frame
time, memory, context-loss, payload decode, or screenshot checks.

## 6. Rollback

```bash
export YOLOE_SCENE_ENABLED=0
export YOLOE_SCENE_MISMATCH_RECOVERY=0
export YOLOE_SCENE_RECOVERY_REPASS=0
export SRVL_ENABLED=0
tools/prod/prod_rollback_yoloe_scene.sh
```

Rollback evidence must prove:

- Scene lane is disabled.
- SRVL is disabled.
- Live remains disabled.
- Existing offline job path completes.
- Existing playback and artifacts remain usable.
- No scene-only mutation remains in existing detections.

## 7. Acceptance Reminder

The feature remains not accepted until all required tests, helper scripts,
workflow/checklist updates, real `combined.mp4` production benchmark artifacts,
generated figures, manifests, issue resolutions, and rollback proof are linked
from the implementation evidence docs.
