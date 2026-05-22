# backend/apps/pipeline/model_bootstrap.py

## Source
- [backend/apps/pipeline/model_bootstrap.py](../../../../../backend/apps/pipeline/model_bootstrap.py)

## Purpose

Startup guard that ensures required model artifacts are present for configured runtime backends before inference workloads begin.

## Key behaviors

1. Reads bootstrap toggles and paths from env (`PYRAMID_MODEL_BOOTSTRAP_*`).
2. Skips bootstrap in test or maintenance commands (`pytest`, `makemigrations`, `collectstatic`, `shell`).
3. Resolves missing model files per role/runtime using pipeline config.
4. Invokes platform script (`bootstrap-models.ps1` on Windows, `.sh` on non-Windows).
5. Uses marker file `.model_bootstrap_complete` to avoid redundant full bootstrap.

## Bootstrap decision path

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TD
    Start["ensure_models_available()"] --> Enabled{"bootstrap enabled?"}
    Enabled -- no --> Exit["return"]
    Enabled -- yes --> Skip{"skip current process?"}
    Skip -- yes --> Exit
    Skip -- no --> Missing["missing_by_runtime()"]
    Missing --> Marker{"marker exists and no missing?"}
    Marker -- yes --> Exit
    Marker -- no --> Fresh{"marker missing?"}
    Fresh -- yes --> All["invoke bootstrap(all)"]
    All --> Missing2["recompute missing"]
    Missing2 --> PerRuntime["invoke bootstrap(runtime) for each missing runtime"]
    Fresh -- no --> PerRuntime
```

## Cross-links

- [config.md](config.md)
- [services/runtime_policy.md](services/runtime_policy.md)
- [../../scripts/check_rtmpose_paths.md](../../scripts/check_rtmpose_paths.md)

