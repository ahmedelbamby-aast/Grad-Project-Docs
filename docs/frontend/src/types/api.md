# frontend/src/types/api.ts

## Related Documents

- [source](../../../../frontend/src/types/api.ts)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

API TypeScript interfaces — aligned with rest-api.md contract.

## Architectural Role

Static contract layer that captures the shared shapes used by runtime code, stores, and APIs.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `DashboardResponse` | Type Alias | Reflected directly from the current top-level implementation surface. |
| `Instructor` | Type Alias | Reflected directly from the current top-level implementation surface. |
| `LoginResponse` | Type Alias | Reflected directly from the current top-level implementation surface. |
| `CameraStatus` | Type Alias | Reflected directly from the current top-level implementation surface. |
| `ConnectionEventType` | Type Alias | Reflected directly from the current top-level implementation surface. |
| `SessionStatus` | Type Alias | Reflected directly from the current top-level implementation surface. |
| `PersonClass` | Type Alias | Reflected directly from the current top-level implementation surface. |
| `BoundingBox` | Type Alias | Reflected directly from the current top-level implementation surface. |
| `SeverityLevel` | Type Alias | Reflected directly from the current top-level implementation surface. |
| `PredictionLevel` | Type Alias | Reflected directly from the current top-level implementation surface. |
| `AnomalyStatus` | Type Alias | Reflected directly from the current top-level implementation surface. |
| `RecordingStatus` | Type Alias | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["frontend/src/types/api.ts\nAPI TypeScript interfaces — aligned with rest-api.md contract."]
    end
    subgraph Dependencies["Project Dependencies"]
        I0["No project-local imports captured
File is locally scoped or infrastructure-only"]
    end
    subgraph Surface["Reflected Surface"]
        E1["DashboardResponse\nType Alias"]
        E2["Instructor\nType Alias"]
        E3["LoginResponse\nType Alias"]
        E4["CameraStatus\nType Alias"]
        E5["ConnectionEventType\nType Alias"]
        E6["SessionStatus\nType Alias"]
        E7["PersonClass\nType Alias"]
        E8["BoundingBox\nType Alias"]
        E9["SeverityLevel\nType Alias"]
        E10["PredictionLevel\nType Alias"]
        E11["AnomalyStatus\nType Alias"]
        E12["RecordingStatus\nType Alias"]
    end
    subgraph Role["Architectural Role"]
        R1["Static contract layer that captures the shared shapes used by runtime code, stores, and APIs."]
    end
    SRC --> I0
    SRC --> E1
    SRC --> E2
    SRC --> E3
    SRC --> E4
    SRC --> E5
    SRC --> E6
    SRC --> E7
    SRC --> E8
    SRC --> E9
    SRC --> E10
    SRC --> E11
    SRC --> E12
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `frontend/src/types/api.ts` and acts as a concrete implementation boundary inside the repository. Static contract layer that captures the shared shapes used by runtime code, stores, and APIs.

From a dependency perspective, the file currently reaches into no project-local imports in the captured surface. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `DashboardResponse`, `Instructor`, `LoginResponse`, `CameraStatus`, `ConnectionEventType`, `SessionStatus`, `PersonClass`, `BoundingBox`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
