# frontend/src/pages/RecordingDetailPage.tsx

## Related Documents

- [source](../../../../frontend/src/pages/RecordingDetailPage.tsx)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

RecordingDetailPage — TF-089 Combines RecordingPlayer + AnomalyTimeline + ExportButton.

## Architectural Role

Route-level UI composition layer that binds stores, hooks, API calls, and visual components into a user workflow.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `RECORDING_DETAIL_PAGE_INTERACTION_CONTRACT` | Export | Reflected directly from the current top-level implementation surface. |
| `RecordingDetailPage` | Function | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["frontend/src/pages/RecordingDetailPage.tsx\nRecordingDetailPage — TF-089 Combines RecordingPlayer + AnomalyTimeline + Export"]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["@/api/recordings\nFrontend Module"]
        I2["@/api/anomalies\nFrontend Module"]
        I3["@/components/recording/RecordingPlayer\nFrontend Module"]
        I4["@/components/recording/AnomalyTimeline\nFrontend Module"]
        I5["@/components/recording/ExportButton\nFrontend Module"]
        I6["@/types/api\nFrontend Module"]
        I7["./RecordingDetailPage.module.css\nFrontend Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["RECORDING_DETAIL_PAGE_INTERACTION_CONTRACT\nExport"]
        E2["RecordingDetailPage\nFunction"]
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
    SRC --> I7
    SRC --> E1
    SRC --> E2
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `frontend/src/pages/RecordingDetailPage.tsx` and acts as a concrete implementation boundary inside the repository. Route-level UI composition layer that binds stores, hooks, API calls, and visual components into a user workflow.

From a dependency perspective, the file currently reaches into `@/api/recordings`, `@/api/anomalies`, `@/components/recording/RecordingPlayer`, `@/components/recording/AnomalyTimeline`, `@/components/recording/ExportButton`, `@/types/api`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `RECORDING_DETAIL_PAGE_INTERACTION_CONTRACT`, `RecordingDetailPage`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
