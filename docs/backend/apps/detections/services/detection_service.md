# backend/apps/detections/services/detection_service.py

## Related Documents

- [source](../../../../../backend/apps/detections/services/detection_service.py)
- [system atlas](../../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

Detection service with injected inference client (T039).

## Architectural Role

Service layer that concentrates orchestration logic away from transport and persistence edges.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `DetectionResult` | Class | Reflected directly from the current top-level implementation surface. |
| `DetectionService` | Class | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["backend/apps/detections/services/detection_service.py\nDetection service with injected inference client (T039)."]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["apps.pipeline.schemas.triton\nProject Module"]
        I2["apps.pipeline.services.base_inference_client\nProject Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["DetectionResult\nClass"]
        E2["DetectionService\nClass"]
    end
    subgraph Role["Architectural Role"]
        R1["Service layer that concentrates orchestration logic away from transport and persistence edges."]
    end
    SRC --> I1
    SRC --> I2
    SRC --> E1
    SRC --> E2
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `backend/apps/detections/services/detection_service.py` and acts as a concrete implementation boundary inside the repository. Service layer that concentrates orchestration logic away from transport and persistence edges.

From a dependency perspective, the file currently reaches into `apps.pipeline.schemas.triton`, `apps.pipeline.services.base_inference_client`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `DetectionResult`, `DetectionService`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
