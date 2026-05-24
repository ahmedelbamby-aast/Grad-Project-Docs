# backend/tools/export_models.py

## Related Documents

- [source](../../../backend/tools/export_models.py)
- [system atlas](../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

Bulk export script for YOLO `.pt` models.

## Architectural Role

Python implementation module that contributes behavior to the repository runtime.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `find_weights_dirs` | Function | Reflected directly from the current top-level implementation surface. |
| `delete_existing_exports` | Function | Reflected directly from the current top-level implementation surface. |
| `export_from_pt` | Function | Reflected directly from the current top-level implementation surface. |
| `main` | Function | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["backend/tools/export_models.py\nBulk export script for YOLO `.pt` models."]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["apps.pipeline.openvino_compat\nProject Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["find_weights_dirs\nFunction"]
        E2["delete_existing_exports\nFunction"]
        E3["export_from_pt\nFunction"]
        E4["main\nFunction"]
    end
    subgraph Role["Architectural Role"]
        R1["Python implementation module that contributes behavior to the repository runtime."]
    end
    SRC --> I1
    SRC --> E1
    SRC --> E2
    SRC --> E3
    SRC --> E4
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `backend/tools/export_models.py` and acts as a concrete implementation boundary inside the repository. Python implementation module that contributes behavior to the repository runtime.

From a dependency perspective, the file currently reaches into `apps.pipeline.openvino_compat`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `find_weights_dirs`, `delete_existing_exports`, `export_from_pt`, `main`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
