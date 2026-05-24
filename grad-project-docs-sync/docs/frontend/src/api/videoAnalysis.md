# frontend/src/api/videoAnalysis.ts

## Related Documents

- [source](../../../../frontend/src/api/videoAnalysis.ts)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

This document reflects the frontend runtime surface exactly as it is implemented in the current codebase.

## Architectural Role

Transport adapter that turns frontend intent into concrete backend REST calls and normalized responses.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `VIDEO_ANALYSIS_API_CONTRACT` | Export | Reflected directly from the current top-level implementation surface. |
| `videoAnalysisApi` | Export | Reflected directly from the current top-level implementation surface. |
| `normalizeVideoAnalysisJobs` | Function | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["frontend/src/api/videoAnalysis.ts\nThis document reflects the frontend runtime surface exactly as it is implemented"]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["./client\nFrontend Module"]
        I2["@/types/api\nFrontend Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["VIDEO_ANALYSIS_API_CONTRACT\nExport"]
        E2["videoAnalysisApi\nExport"]
        E3["normalizeVideoAnalysisJobs\nFunction"]
    end
    subgraph Role["Architectural Role"]
        R1["Transport adapter that turns frontend intent into concrete backend REST calls and normalized responses."]
    end
    SRC --> I1
    SRC --> I2
    SRC --> E1
    SRC --> E2
    SRC --> E3
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `frontend/src/api/videoAnalysis.ts` and acts as a concrete implementation boundary inside the repository. Transport adapter that turns frontend intent into concrete backend REST calls and normalized responses.

From a dependency perspective, the file currently reaches into `./client`, `@/types/api`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `VIDEO_ANALYSIS_API_CONTRACT`, `videoAnalysisApi`, `normalizeVideoAnalysisJobs`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
