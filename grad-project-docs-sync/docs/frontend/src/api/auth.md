# frontend/src/api/auth.ts

## Related Documents

- [source](../../../../frontend/src/api/auth.ts)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

Auth API service — T035 Consumes: POST /auth/login/, POST /auth/logout/, GET /auth/me/, PATCH /auth/me/

## Architectural Role

Transport adapter that turns frontend intent into concrete backend REST calls and normalized responses.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `authApi` | Export | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["frontend/src/api/auth.ts\nAuth API service — T035 Consumes: POST /auth/login/, POST /auth/logout/, GET /au"]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["./client\nFrontend Module"]
        I2["@/types/api\nFrontend Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["authApi\nExport"]
    end
    subgraph Role["Architectural Role"]
        R1["Transport adapter that turns frontend intent into concrete backend REST calls and normalized responses."]
    end
    SRC --> I1
    SRC --> I2
    SRC --> E1
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `frontend/src/api/auth.ts` and acts as a concrete implementation boundary inside the repository. Transport adapter that turns frontend intent into concrete backend REST calls and normalized responses.

From a dependency perspective, the file currently reaches into `./client`, `@/types/api`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `authApi`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
