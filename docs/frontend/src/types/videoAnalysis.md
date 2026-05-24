# frontend/src/types/videoAnalysis.ts

## Related Documents

- [source](../../../../frontend/src/types/videoAnalysis.ts)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

This document reflects the frontend runtime surface exactly as it is implemented in the current codebase.

## Architectural Role

Static contract layer that captures the shared shapes used by runtime code, stores, and APIs.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `VideoAnalysisJobStatus` | Type Alias | Reflected directly from the current top-level implementation surface. |
| `VideoAnalysisPipelineMode` | Type Alias | Reflected directly from the current top-level implementation surface. |
| `VideoAnalysisJobMetadata` | Interface | Reflected directly from the current top-level implementation surface. |
| `VideoAnalysisJobSummary` | Interface | Reflected directly from the current top-level implementation surface. |
| `VideoVisibilityState` | Interface | Reflected directly from the current top-level implementation surface. |
| `VideoAnalysisBoundingBox` | Interface | Reflected directly from the current top-level implementation surface. |
| `VideoAnalysisFrame` | Interface | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["frontend/src/types/videoAnalysis.ts\nThis document reflects the frontend runtime surface exactly as it is implemented"]
    end
    subgraph Dependencies["Project Dependencies"]
        I0["No project-local imports captured
File is locally scoped or infrastructure-only"]
    end
    subgraph Surface["Reflected Surface"]
        E1["VideoAnalysisJobStatus\nType Alias"]
        E2["VideoAnalysisPipelineMode\nType Alias"]
        E3["VideoAnalysisJobMetadata\nInterface"]
        E4["VideoAnalysisJobSummary\nInterface"]
        E5["VideoVisibilityState\nInterface"]
        E6["VideoAnalysisBoundingBox\nInterface"]
        E7["VideoAnalysisFrame\nInterface"]
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
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `frontend/src/types/videoAnalysis.ts` and acts as a concrete implementation boundary inside the repository. Static contract layer that captures the shared shapes used by runtime code, stores, and APIs.

From a dependency perspective, the file currently reaches into no project-local imports in the captured surface. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `VideoAnalysisJobStatus`, `VideoAnalysisPipelineMode`, `VideoAnalysisJobMetadata`, `VideoAnalysisJobSummary`, `VideoVisibilityState`, `VideoAnalysisBoundingBox`, `VideoAnalysisFrame`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
