# scripts/export-tensorrt-models.ps1

## Related Documents

- [source](../../scripts/export-tensorrt-models.ps1)
- [system atlas](../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

This document reflects the project operational surface that supports runtime, CI, and deployment workflows.

## Architectural Role

Operational automation script used to prepare, verify, or synchronize runtime infrastructure and model artifacts.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `Write-Step` | Function | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["scripts/export-tensorrt-models.ps1\nThis document reflects the project operational surface that supports runtime, CI"]
    end
    subgraph Dependencies["Project Dependencies"]
        I0["No project-local imports captured
File is locally scoped or infrastructure-only"]
    end
    subgraph Surface["Reflected Surface"]
        E1["Write-Step\nFunction"]
    end
    subgraph Role["Architectural Role"]
        R1["Operational automation script used to prepare, verify, or synchronize runtime infrastructure and model artifacts."]
    end
    SRC --> I0
    SRC --> E1
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `scripts/export-tensorrt-models.ps1` and acts as a concrete implementation boundary inside the repository. Operational automation script used to prepare, verify, or synchronize runtime infrastructure and model artifacts.

From a dependency perspective, the file currently reaches into no project-local imports in the captured surface. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `Write-Step`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
