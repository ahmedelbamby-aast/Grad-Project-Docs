# frontend/src/hooks/useAnomalySocket.ts

## Related Documents

- [source](../../../../frontend/src/hooks/useAnomalySocket.ts)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

T071 — useAnomalySocket hook Connects to wss://{host}/ws/anomalies/{session_id}/ Processes anomaly.alert, anomaly.behavior_end, and anomaly.status_change WS messages, dispatches to anomaly store.

## Architectural Role

Side-effect and integration layer that encapsulates stateful browser, WebSocket, or API behavior.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `useAnomalySocket` | Function | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["frontend/src/hooks/useAnomalySocket.ts\nT071 — useAnomalySocket hook Connects to wss://{host}/ws/anomalies/{session_id}/"]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["./useWebSocket\nFrontend Module"]
        I2["@/stores/anomalyStore\nFrontend Module"]
        I3["@/types/api\nFrontend Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["useAnomalySocket\nFunction"]
    end
    subgraph Role["Architectural Role"]
        R1["Side-effect and integration layer that encapsulates stateful browser, WebSocket, or API behavior."]
    end
    SRC --> I1
    SRC --> I2
    SRC --> I3
    SRC --> E1
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `frontend/src/hooks/useAnomalySocket.ts` and acts as a concrete implementation boundary inside the repository. Side-effect and integration layer that encapsulates stateful browser, WebSocket, or API behavior.

From a dependency perspective, the file currently reaches into `./useWebSocket`, `@/stores/anomalyStore`, `@/types/api`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `useAnomalySocket`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
