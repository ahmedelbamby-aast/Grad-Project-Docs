# backend/apps/detections/consumers.py

## Related Documents

- [source](../../../../backend/apps/detections/consumers.py)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

This document reflects the backend feature module exactly as it is implemented in the repository today.

## Architectural Role

WebSocket interaction boundary that receives live events and emits real-time updates to subscribed clients.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `DETECTION_CONSUMER_CONTRACT` | Constant | Reflected directly from the current top-level implementation surface. |
| `DetectionConsumer` | Class | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["backend/apps/detections/consumers.py\nThis document reflects the backend feature module exactly as it is implemented i"]
    end
    subgraph Dependencies["Project Dependencies"]
        I0["No project-local imports captured
File is locally scoped or infrastructure-only"]
    end
    subgraph Surface["Reflected Surface"]
        E1["DETECTION_CONSUMER_CONTRACT\nConstant"]
        E2["DetectionConsumer\nClass"]
    end
    subgraph Role["Architectural Role"]
        R1["WebSocket interaction boundary that receives live events and emits real-time updates to subscribed clients."]
    end
    SRC --> I0
    SRC --> E1
    SRC --> E2
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `backend/apps/detections/consumers.py` and acts as a concrete implementation boundary inside the repository. WebSocket interaction boundary that receives live events and emits real-time updates to subscribed clients.

From a dependency perspective, the file currently reaches into no project-local imports in the captured surface. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `DETECTION_CONSUMER_CONTRACT`, `DetectionConsumer`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
