# backend/apps/tracking/pipeline.py

## Related Documents

- [source](../../../../backend/apps/tracking/pipeline.py)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

Pipeline execution helpers for legacy crop and full-frame modes.

## Architectural Role

Python implementation module that contributes behavior to the repository runtime.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `MODEL_NAMES` | Constant | Reflected directly from the current top-level implementation surface. |
| `_pick_unique_track_color` | Function | Reflected directly from the current top-level implementation surface. |
| `_ensure_student_track` | Function | Reflected directly from the current top-level implementation surface. |
| `_primary_detections` | Function | Reflected directly from the current top-level implementation surface. |
| `_secondary_boxes_from_primary` | Function | Reflected directly from the current top-level implementation surface. |
| `process_frame` | Function | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["backend/apps/tracking/pipeline.py\nPipeline execution helpers for legacy crop and full-frame modes."]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["apps.tracking.association\nProject Module"]
        I2["apps.tracking.colors\nProject Module"]
        I3["apps.tracking.pipeline_mode\nProject Module"]
        I4["apps.tracking.rendering\nProject Module"]
        I5["apps.tracking.tracker\nProject Module"]
        I6["apps.video_analysis.models\nProject Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["MODEL_NAMES\nConstant"]
        E2["_pick_unique_track_color\nFunction"]
        E3["_ensure_student_track\nFunction"]
        E4["_primary_detections\nFunction"]
        E5["_secondary_boxes_from_primary\nFunction"]
        E6["process_frame\nFunction"]
    end
    subgraph Role["Architectural Role"]
        R1["Python implementation module that contributes behavior to the repository runtime."]
    end
    SRC --> I1
    SRC --> I2
    SRC --> I3
    SRC --> I4
    SRC --> I5
    SRC --> I6
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

This module sits at `backend/apps/tracking/pipeline.py` and acts as a concrete implementation boundary inside the repository. Python implementation module that contributes behavior to the repository runtime.

From a dependency perspective, the file currently reaches into `apps.tracking.association`, `apps.tracking.colors`, `apps.tracking.pipeline_mode`, `apps.tracking.rendering`, `apps.tracking.tracker`, `apps.video_analysis.models`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `MODEL_NAMES`, `_pick_unique_track_color`, `_ensure_student_track`, `_primary_detections`, `_secondary_boxes_from_primary`, `process_frame`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
