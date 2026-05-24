# backend/apps/video_analysis/README.md

## Related Documents

- [source](../../../../backend/apps/video_analysis/README.md)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

This document reflects the backend feature module exactly as it is implemented in the repository today.

## Architectural Role

Feature overview document that summarizes the current subsystem from a repository perspective.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `Video Analysis Module` | Document Section | Reflected directly from the current top-level implementation surface. |
| `Boundary Summary` | Document Section | Reflected directly from the current top-level implementation surface. |
| `Offline Prediction Flow` | Document Section | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["backend/apps/video_analysis/README.md\nThis document reflects the backend feature module exactly as it is implemented i"]
    end
    subgraph Dependencies["Project Dependencies"]
        I0["No project-local imports captured
File is locally scoped or infrastructure-only"]
    end
    subgraph Surface["Reflected Surface"]
        E1["Video Analysis Module\nDocument Section"]
        E2["Boundary Summary\nDocument Section"]
        E3["Offline Prediction Flow\nDocument Section"]
    end
    subgraph Role["Architectural Role"]
        R1["Feature overview document that summarizes the current subsystem from a repository perspective."]
    end
    SRC --> I0
    SRC --> E1
    SRC --> E2
    SRC --> E3
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `backend/apps/video_analysis/README.md` and acts as a concrete implementation boundary inside the repository. Feature overview document that summarizes the current subsystem from a repository perspective.

## 004 Feature Alignment

- Supports dual upload execution paths via `pipeline_mode` (`legacy_crop` and `full_frame`).
- Exposes playback bundles with optional `annotated_video_url` for rendered exports.
- Persists recoverable failure metadata (`completed_partial`, `component_failures`) and retry entrypoints used by the frontend analytics page.

From a dependency perspective, the file currently reaches into no project-local imports in the captured surface. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `Video Analysis Module`, `Boundary Summary`, `Offline Prediction Flow`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
