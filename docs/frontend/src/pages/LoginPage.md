# frontend/src/pages/LoginPage.tsx

## Related Documents

- [source](../../../../frontend/src/pages/LoginPage.tsx)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

LoginPage — T041 FR-043: Email + password login form, full-bleed dark background, centred card.

## Architectural Role

Route-level UI composition layer that binds stores, hooks, API calls, and visual components into a user workflow.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `LoginPage` | Function | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["frontend/src/pages/LoginPage.tsx\nLoginPage — T041 FR-043: Email + password login form, full-bleed dark background"]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["@/stores/authStore\nFrontend Module"]
        I2["@/components/ui\nFrontend Module"]
        I3["./LoginPage.module.css\nFrontend Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["LoginPage\nFunction"]
    end
    subgraph Role["Architectural Role"]
        R1["Route-level UI composition layer that binds stores, hooks, API calls, and visual components into a user workflow."]
    end
    SRC --> I1
    SRC --> I2
    SRC --> I3
    SRC --> E1
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `frontend/src/pages/LoginPage.tsx` and acts as a concrete implementation boundary inside the repository. Route-level UI composition layer that binds stores, hooks, API calls, and visual components into a user workflow.

From a dependency perspective, the file currently reaches into `@/stores/authStore`, `@/components/ui`, `./LoginPage.module.css`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `LoginPage`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
