# frontend/src/stores/anomalyStore.ts

## Related Documents

- [source](../../../../frontend/src/stores/anomalyStore.ts)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

T070 — Zustand anomaly store (US4) Active alerts sorted by severity/timestamp, history, triage actions with optimistic updates + rollback on first-write-wins rejection, notes per anomaly, WebSocket status_change handler.

## Architectural Role

Client state container that centralizes domain data and update logic for the frontend runtime.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `ANOMALY_STORE_TRANSITION_CONTRACT` | Export | Reflected directly from the current top-level implementation surface. |
| `useAnomalyStore` | Export | Reflected directly from the current top-level implementation surface. |
| `selectAnomalyWorkflowState` | Function | Reflected directly from the current top-level implementation surface. |
| `ConflictError` | Interface | Reflected directly from the current top-level implementation surface. |
| `AnomalyStoreState` | Interface | Reflected directly from the current top-level implementation surface. |
| `AnomalyWorkflowState` | Interface | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["frontend/src/stores/anomalyStore.ts\nT070 — Zustand anomaly store (US4) Active alerts sorted by severity/timestamp, h"]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["@/api/anomalies\nFrontend Module"]
        I2["@/types/api\nFrontend Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["ANOMALY_STORE_TRANSITION_CONTRACT\nExport"]
        E2["useAnomalyStore\nExport"]
        E3["selectAnomalyWorkflowState\nFunction"]
        E4["ConflictError\nInterface"]
        E5["AnomalyStoreState\nInterface"]
        E6["AnomalyWorkflowState\nInterface"]
    end
    subgraph Role["Architectural Role"]
        R1["Client state container that centralizes domain data and update logic for the frontend runtime."]
    end
    SRC --> I1
    SRC --> I2
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

This module sits at `frontend/src/stores/anomalyStore.ts` and acts as a concrete implementation boundary inside the repository. Client state container that centralizes domain data and update logic for the frontend runtime.

From a dependency perspective, the file currently reaches into `@/api/anomalies`, `@/types/api`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `ANOMALY_STORE_TRANSITION_CONTRACT`, `useAnomalyStore`, `selectAnomalyWorkflowState`, `ConflictError`, `AnomalyStoreState`, `AnomalyWorkflowState`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
