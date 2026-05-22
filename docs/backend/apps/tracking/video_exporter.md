# backend/apps/tracking/video_exporter.py

## Related Documents

- [source](../../../../backend/apps/tracking/video_exporter.py)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

Annotated video export helpers.

## Architectural Role

Python implementation module that contributes behavior to the repository runtime.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `_video_job_root` | Function | Reflected directly from the current top-level implementation surface. |
| `_resolve_visible_models` | Function | Reflected directly from the current top-level implementation surface. |
| `_try_ultralytics_render` | Function | Reflected directly from the current top-level implementation surface. |
| `_resolve_ffmpeg_binary` | Function | Reflected directly from the current top-level implementation surface. |
| `render_annotated_video` | Function | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["backend/apps/tracking/video_exporter.py\nAnnotated video export helpers."]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["apps.pipeline.openvino_compat\nProject Module"]
        I2["apps.pipeline.config\nProject Module"]
        I3["apps.video_analysis.models\nProject Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["_video_job_root\nFunction"]
        E2["_resolve_visible_models\nFunction"]
        E3["_try_ultralytics_render\nFunction"]
        E4["_resolve_ffmpeg_binary\nFunction"]
        E5["render_annotated_video\nFunction"]
    end
    subgraph Role["Architectural Role"]
        R1["Python implementation module that contributes behavior to the repository runtime."]
    end
    SRC --> I1
    SRC --> I2
    SRC --> I3
    SRC --> E1
    SRC --> E2
    SRC --> E3
    SRC --> E4
    SRC --> E5
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `backend/apps/tracking/video_exporter.py` and acts as a concrete implementation boundary inside the repository. Python implementation module that contributes behavior to the repository runtime.

From a dependency perspective, the file currently reaches into `apps.pipeline.openvino_compat`, `apps.pipeline.config`, `apps.video_analysis.models`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `_video_job_root`, `_resolve_visible_models`, `_try_ultralytics_render`, `_resolve_ffmpeg_binary`, `render_annotated_video`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
