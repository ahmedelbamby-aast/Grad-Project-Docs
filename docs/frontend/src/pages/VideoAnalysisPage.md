# frontend/src/pages/VideoAnalysisPage.tsx

## Related Documents

- [source](../../../../frontend/src/pages/VideoAnalysisPage.tsx)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

This document reflects the frontend runtime surface exactly as it is implemented in the current codebase.

## Architectural Role

Route-level UI composition layer that binds stores, hooks, API calls, and visual components into a user workflow.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `VIDEO_ANALYSIS_PAGE_INTERACTION_CONTRACT` | Export | Reflected directly from the current top-level implementation surface. |
| `VideoAnalysisPage` | Function | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["frontend/src/pages/VideoAnalysisPage.tsx\nThis document reflects the frontend runtime surface exactly as it is implemented"]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["@/components/VideoUploadTab/VideoUploadTab\nFrontend Module"]
        I2["@/components/ModelVisibilityToggles/ModelVisibilityToggles\nFrontend Module"]
        I3["@/components/VideoPlayer/VideoPlayer\nFrontend Module"]
        I4["@/components/VideoUploadTab/JobsList\nFrontend Module"]
        I5["@/api/videoAnalysis\nFrontend Module"]
        I6["@/stores/uploadStore\nFrontend Module"]
        I7["@/stores/visibilityStore\nFrontend Module"]
        I8["@/hooks/useVideoAnalysisLive\nFrontend Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["VIDEO_ANALYSIS_PAGE_INTERACTION_CONTRACT\nExport"]
        E2["VideoAnalysisPage\nFunction"]
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
    SRC --> I8
    SRC --> E1
    SRC --> E2
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `frontend/src/pages/VideoAnalysisPage.tsx` and acts as a concrete implementation boundary inside the repository. Route-level UI composition layer that binds stores, hooks, API calls, and visual components into a user workflow.

From a dependency perspective, the file currently reaches into `@/components/VideoUploadTab/VideoUploadTab`, `@/components/ModelVisibilityToggles/ModelVisibilityToggles`, `@/components/VideoPlayer/VideoPlayer`, `@/components/VideoUploadTab/JobsList`, `@/api/videoAnalysis`, `@/stores/uploadStore`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `VIDEO_ANALYSIS_PAGE_INTERACTION_CONTRACT`, `VideoAnalysisPage`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
