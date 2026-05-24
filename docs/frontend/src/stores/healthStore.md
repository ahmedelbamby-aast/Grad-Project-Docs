# frontend/src/stores/healthStore.ts

## Related Documents

- [source](../../../../frontend/src/stores/healthStore.ts)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

Health Zustand store — TF-104 State: healthStatus, storageWarning, loading, error Updated from WebSocket health.update and storage.warning messages.

## Architectural Role

Client state container that centralizes domain data and update logic for the frontend runtime.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `HEALTH_STORE_TRANSITION_CONTRACT` | Export | Reflected directly from the current top-level implementation surface. |
| `useHealthStore` | Export | Reflected directly from the current top-level implementation surface. |
| `selectHealthWorkflowState` | Function | Reflected directly from the current top-level implementation surface. |
| `StorageWarningLevel` | Type Alias | Reflected directly from the current top-level implementation surface. |
| `HealthState` | Interface | Reflected directly from the current top-level implementation surface. |
| `HealthWorkflowState` | Interface | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["frontend/src/stores/healthStore.ts\nHealth Zustand store — TF-104 State: healthStatus, storageWarning, loading, erro"]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["@/api/health\nFrontend Module"]
        I2["@/types/api\nFrontend Module"]
        I3["@/types/websocket\nFrontend Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["HEALTH_STORE_TRANSITION_CONTRACT\nExport"]
        E2["useHealthStore\nExport"]
        E3["selectHealthWorkflowState\nFunction"]
        E4["StorageWarningLevel\nType Alias"]
        E5["HealthState\nInterface"]
        E6["HealthWorkflowState\nInterface"]
    end
    subgraph Role["Architectural Role"]
        R1["Client state container that centralizes domain data and update logic for the frontend runtime."]
    end
    SRC --> I1
    SRC --> I2
    SRC --> I3
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

This module sits at `frontend/src/stores/healthStore.ts` and acts as a concrete implementation boundary inside the repository. Client state container that centralizes domain data and update logic for the frontend runtime.

From a dependency perspective, the file currently reaches into `@/api/health`, `@/types/api`, `@/types/websocket`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `HEALTH_STORE_TRANSITION_CONTRACT`, `useHealthStore`, `selectHealthWorkflowState`, `StorageWarningLevel`, `HealthState`, `HealthWorkflowState`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
