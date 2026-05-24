# backend/apps/tracking/pipeline_mode.py

## Related Documents

- [source](../../../../backend/apps/tracking/pipeline_mode.py)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

Pipeline mode helpers for legacy crop vs full-frame processing.

## Architectural Role

Python implementation module that contributes behavior to the repository runtime.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `PIPELINE_MODE_LEGACY` | Constant | Reflected directly from the current top-level implementation surface. |
| `PIPELINE_MODE_FULL_FRAME` | Constant | Reflected directly from the current top-level implementation surface. |
| `PipelineMode` | Class | Reflected directly from the current top-level implementation surface. |
| `normalize_pipeline_mode` | Function | Reflected directly from the current top-level implementation surface. |
| `pipeline_mode_label` | Function | Reflected directly from the current top-level implementation surface. |
| `is_full_frame_mode` | Function | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["backend/apps/tracking/pipeline_mode.py\nPipeline mode helpers for legacy crop vs full-frame processing."]
    end
    subgraph Dependencies["Project Dependencies"]
        I0["No project-local imports captured
File is locally scoped or infrastructure-only"]
    end
    subgraph Surface["Reflected Surface"]
        E1["PIPELINE_MODE_LEGACY\nConstant"]
        E2["PIPELINE_MODE_FULL_FRAME\nConstant"]
        E3["PipelineMode\nClass"]
        E4["normalize_pipeline_mode\nFunction"]
        E5["pipeline_mode_label\nFunction"]
        E6["is_full_frame_mode\nFunction"]
    end
    subgraph Role["Architectural Role"]
        R1["Python implementation module that contributes behavior to the repository runtime."]
    end
    SRC --> I0
    SRC --> E1
    SRC --> E2
    SRC --> E3
    SRC --> E4
    SRC --> E5
    SRC --> E6
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `backend/apps/tracking/pipeline_mode.py` and acts as a concrete implementation boundary inside the repository. Python implementation module that contributes behavior to the repository runtime.

From a dependency perspective, the file currently reaches into no project-local imports in the captured surface. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `PIPELINE_MODE_LEGACY`, `PIPELINE_MODE_FULL_FRAME`, `PipelineMode`, `normalize_pipeline_mode`, `pipeline_mode_label`, `is_full_frame_mode`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
