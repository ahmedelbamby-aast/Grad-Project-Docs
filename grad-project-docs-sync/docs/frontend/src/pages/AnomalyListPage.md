# frontend/src/pages/AnomalyListPage.tsx

## Related Documents

- [source](../../../../frontend/src/pages/AnomalyListPage.tsx)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

AnomalyListPage — T091 (US4) Full alert history browsing with filtering, sorting, search, and inline triage.

## Architectural Role

Route-level UI composition layer that binds stores, hooks, API calls, and visual components into a user workflow.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `AnomalyListPage` | Function | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["frontend/src/pages/AnomalyListPage.tsx\nAnomalyListPage — T091 (US4) Full alert history browsing with filtering, sorting"]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["@/api/anomalies\nFrontend Module"]
        I2["@/stores/anomalyStore\nFrontend Module"]
        I3["@/components/ui\nFrontend Module"]
        I4["@/components/anomaly/TriageActions\nFrontend Module"]
        I5["@/types/api\nFrontend Module"]
        I6["./AnomalyListPage.module.css\nFrontend Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["AnomalyListPage\nFunction"]
    end
    subgraph Role["Architectural Role"]
        R1["Route-level UI composition layer that binds stores, hooks, API calls, and visual components into a user workflow."]
    end
    SRC --> I1
    SRC --> I2
    SRC --> I3
    SRC --> I4
    SRC --> I5
    SRC --> I6
    SRC --> E1
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `frontend/src/pages/AnomalyListPage.tsx` and acts as a concrete implementation boundary inside the repository. Route-level UI composition layer that binds stores, hooks, API calls, and visual components into a user workflow.

From a dependency perspective, the file currently reaches into `@/api/anomalies`, `@/stores/anomalyStore`, `@/components/ui`, `@/components/anomaly/TriageActions`, `@/types/api`, `./AnomalyListPage.module.css`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `AnomalyListPage`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
