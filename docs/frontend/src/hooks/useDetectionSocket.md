# frontend/src/hooks/useDetectionSocket.ts

## Related Documents

- [source](../../../../frontend/src/hooks/useDetectionSocket.ts)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

useDetectionSocket — T053 Connects to the detection WebSocket for a session and dispatches detection.frame and prediction.update messages to the detection store.

## Architectural Role

Side-effect and integration layer that encapsulates stateful browser, WebSocket, or API behavior.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `useDetectionSocket` | Function | Reflected directly from the current top-level implementation surface. |
| `UseDetectionSocketOptions` | Interface | Reflected directly from the current top-level implementation surface. |
| `UseDetectionSocketReturn` | Interface | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["frontend/src/hooks/useDetectionSocket.ts\nuseDetectionSocket — T053 Connects to the detection WebSocket for a session and "]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["@/hooks/useWebSocket\nFrontend Module"]
        I2["@/stores/detectionStore\nFrontend Module"]
        I3["@/types/websocket\nFrontend Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["useDetectionSocket\nFunction"]
        E2["UseDetectionSocketOptions\nInterface"]
        E3["UseDetectionSocketReturn\nInterface"]
    end
    subgraph Role["Architectural Role"]
        R1["Side-effect and integration layer that encapsulates stateful browser, WebSocket, or API behavior."]
    end
    SRC --> I1
    SRC --> I2
    SRC --> I3
    SRC --> E1
    SRC --> E2
    SRC --> E3
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `frontend/src/hooks/useDetectionSocket.ts` and acts as a concrete implementation boundary inside the repository. Side-effect and integration layer that encapsulates stateful browser, WebSocket, or API behavior.

From a dependency perspective, the file currently reaches into `@/hooks/useWebSocket`, `@/stores/detectionStore`, `@/types/websocket`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `useDetectionSocket`, `UseDetectionSocketOptions`, `UseDetectionSocketReturn`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
