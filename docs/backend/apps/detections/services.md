# backend/apps/detections/services.py

## Related Documents

- [source](../../../../backend/apps/detections/services.py)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

This document reflects the backend feature module exactly as it is implemented in the repository today.

## Architectural Role

Service layer that concentrates orchestration logic away from transport and persistence edges.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `_group_name` | Function | Reflected directly from the current top-level implementation surface. |
| `_append_live_inference_audit` | Function | Reflected directly from the current top-level implementation surface. |
| `publish_detection_frame` | Function | Reflected directly from the current top-level implementation surface. |
| `publish_prediction_update` | Function | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["backend/apps/detections/services.py\nThis document reflects the backend feature module exactly as it is implemented i"]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["apps.pipeline.config\nProject Module"]
        I2["apps.pipeline.services.runtime_policy\nProject Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["_group_name\nFunction"]
        E2["_append_live_inference_audit\nFunction"]
        E3["publish_detection_frame\nFunction"]
        E4["publish_prediction_update\nFunction"]
    end
    subgraph Role["Architectural Role"]
        R1["Service layer that concentrates orchestration logic away from transport and persistence edges."]
    end
    SRC --> I1
    SRC --> I2
    SRC --> E1
    SRC --> E2
    SRC --> E3
    SRC --> E4
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `backend/apps/detections/services.py` and acts as a concrete implementation boundary inside the repository. Service layer that concentrates orchestration logic away from transport and persistence edges.

From a dependency perspective, the file currently reaches into `apps.pipeline.config`, `apps.pipeline.services.runtime_policy`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `_group_name`, `_append_live_inference_audit`, `publish_detection_frame`, `publish_prediction_update`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
