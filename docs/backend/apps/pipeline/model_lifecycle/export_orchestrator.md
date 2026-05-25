# backend/apps/pipeline/model_lifecycle/export_orchestrator.py

## Related Documents

- [source](../../../../../backend/apps/pipeline/model_lifecycle/export_orchestrator.py)
- [system atlas](../../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

Export planning for missing model formats.

## Architectural Role

Python implementation module that contributes behavior to the repository runtime.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `ExportDecision` | Class | Reflected directly from the current top-level implementation surface. |
| `ExportOrchestrator` | Class | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["backend/apps/pipeline/model_lifecycle/export_orchestrator.py\nExport planning for missing model formats."]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["apps.pipeline.model_lifecycle.export_worker\nProject Module"]
        I2["apps.pipeline.model_lifecycle.inventory\nProject Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["ExportDecision\nClass"]
        E2["ExportOrchestrator\nClass"]
    end
    subgraph Role["Architectural Role"]
        R1["Python implementation module that contributes behavior to the repository runtime."]
    end
    SRC --> I1
    SRC --> I2
    SRC --> E1
    SRC --> E2
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `backend/apps/pipeline/model_lifecycle/export_orchestrator.py` and acts as a concrete implementation boundary inside the repository. Python implementation module that contributes behavior to the repository runtime.

From a dependency perspective, the file currently reaches into `apps.pipeline.model_lifecycle.export_worker`, `apps.pipeline.model_lifecycle.inventory`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `ExportDecision`, `ExportOrchestrator`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
