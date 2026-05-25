# backend/apps/tracking/rendering.py

## Related Documents

- [source](../../../../backend/apps/tracking/rendering.py)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

Bounding box rendering helpers for model overlays.

## Architectural Role

Python implementation module that contributes behavior to the repository runtime.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `RENDERING_OUTPUT_CONTRACT` | Constant | Reflected directly from the current top-level implementation surface. |
| `build_tracking_id_text` | Function | Reflected directly from the current top-level implementation surface. |
| `build_box_color` | Function | Reflected directly from the current top-level implementation surface. |
| `summarise_model_names` | Function | Reflected directly from the current top-level implementation surface. |
| `create_bounding_box` | Function | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["backend/apps/tracking/rendering.py\nBounding box rendering helpers for model overlays."]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["apps.tracking.colors\nProject Module"]
        I2["apps.video_analysis.models\nProject Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["RENDERING_OUTPUT_CONTRACT\nConstant"]
        E2["build_tracking_id_text\nFunction"]
        E3["build_box_color\nFunction"]
        E4["summarise_model_names\nFunction"]
        E5["create_bounding_box\nFunction"]
    end
    subgraph Role["Architectural Role"]
        R1["Python implementation module that contributes behavior to the repository runtime."]
    end
    SRC --> I1
    SRC --> I2
    SRC --> E1
    SRC --> E2
    SRC --> E3
    SRC --> E4
    SRC --> E5
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `backend/apps/tracking/rendering.py` and acts as a concrete implementation boundary inside the repository. Python implementation module that contributes behavior to the repository runtime.

From a dependency perspective, the file currently reaches into `apps.tracking.colors`, `apps.video_analysis.models`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `RENDERING_OUTPUT_CONTRACT`, `build_tracking_id_text`, `build_box_color`, `summarise_model_names`, `create_bounding_box`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
