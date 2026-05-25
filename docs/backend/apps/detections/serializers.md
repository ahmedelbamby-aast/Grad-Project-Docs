# backend/apps/detections/serializers.py

## Related Documents

- [source](../../../../backend/apps/detections/serializers.py)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

This document reflects the backend feature module exactly as it is implemented in the repository today.

## Architectural Role

Contract boundary that shapes request validation and response serialization for external callers.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `DetectionSerializer` | Class | Reflected directly from the current top-level implementation surface. |
| `PyramidPredictionSerializer` | Class | Reflected directly from the current top-level implementation surface. |
| `DetectionFrameSerializer` | Class | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["backend/apps/detections/serializers.py\nThis document reflects the backend feature module exactly as it is implemented i"]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["apps.detections.models\nProject Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["DetectionSerializer\nClass"]
        E2["PyramidPredictionSerializer\nClass"]
        E3["DetectionFrameSerializer\nClass"]
    end
    subgraph Role["Architectural Role"]
        R1["Contract boundary that shapes request validation and response serialization for external callers."]
    end
    SRC --> I1
    SRC --> E1
    SRC --> E2
    SRC --> E3
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `backend/apps/detections/serializers.py` and acts as a concrete implementation boundary inside the repository. Contract boundary that shapes request validation and response serialization for external callers.

From a dependency perspective, the file currently reaches into `apps.detections.models`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `DetectionSerializer`, `PyramidPredictionSerializer`, `DetectionFrameSerializer`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
