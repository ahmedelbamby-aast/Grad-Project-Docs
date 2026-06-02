# Model Runtime Packs

**Last updated:** 2026-05-10

This folder provides cross-platform scripts for packaging and bootstrapping
model artifacts by runtime:

- `onnx` (`*.onnx`)
- `tensorrt` (`*.engine`)
- `openvino` (`*.xml`, `*.bin`)
- `pt` (`*.pt`)

## Create Release Packs

PowerShell:

```powershell
./scripts/models/create-model-releases.ps1 -Runtime all -ModelsDir ./backend/models -OutDir ./releases/model-packs
```

Bash:

```bash
./scripts/models/create-model-releases.sh --runtime all --models-dir ./backend/models --out-dir ./releases/model-packs
```

## Bootstrap Models From Packs

PowerShell:

```powershell
./scripts/models/bootstrap-models.ps1 -Runtime all -ModelsDir ./backend/models -ReleasesDir ./releases/model-packs
```

Bash:

```bash
./scripts/models/bootstrap-models.sh --runtime all --models-dir ./backend/models --releases-dir ./releases/model-packs
```

Both bootstrap scripts create a marker file:

- default: `<models-dir>/.model_bootstrap_complete`

## Startup Auto-Bootstrap

`apps.pipeline` startup uses `backend/apps/pipeline/model_bootstrap.py`.

It triggers bootstrap when either condition is true:

- marker file does not exist (fresh install)
- any configured runtime model for required roles is missing

Environment controls:

- `PYRAMID_MODEL_BOOTSTRAP_ON_STARTUP=true|false`
- `PYRAMID_MODEL_RELEASES_DIR=/path/to/releases/model-packs`
- `PYRAMID_MODEL_BOOTSTRAP_MARKER=/path/to/custom-marker-file`

Default release pack directory:

- `<repo-root>/releases/model-packs`
