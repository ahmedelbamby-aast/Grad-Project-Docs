# backend/apps/cameras/models.py

## Related Documents

- [source](../../../../backend/apps/cameras/models.py)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

This document reflects the backend feature module exactly as it is implemented in the repository today.

## Architectural Role

Persistence boundary that defines the database entities and relationships used by the surrounding feature.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `CameraSource` | Class | Reflected directly from the current top-level implementation surface. |
| `StreamRegistration` | Class | Reflected directly from the current top-level implementation surface. |
| `ConnectionEvent` | Class | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["backend/apps/cameras/models.py\nThis document reflects the backend feature module exactly as it is implemented i"]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["apps.cameras.fields\nProject Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["CameraSource\nClass"]
        E2["StreamRegistration\nClass"]
        E3["ConnectionEvent\nClass"]
    end
    subgraph Role["Architectural Role"]
        R1["Persistence boundary that defines the database entities and relationships used by the surrounding feature."]
    end
    SRC --> I1
    SRC --> E1
    SRC --> E2
    SRC --> E3
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `backend/apps/cameras/models.py` and acts as a concrete implementation boundary inside the repository. Persistence boundary that defines the database entities and relationships used by the surrounding feature.

From a dependency perspective, the file currently reaches into `apps.cameras.fields`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `CameraSource`, `StreamRegistration`, `ConnectionEvent`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
