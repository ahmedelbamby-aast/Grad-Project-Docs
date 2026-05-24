# backend/apps/accounts/views.py

## Related Documents

- [source](../../../../backend/apps/accounts/views.py)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

This document reflects the backend feature module exactly as it is implemented in the repository today.

## Architectural Role

HTTP boundary that translates requests into validated application behavior and serialized responses.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `LoginView` | Class | Reflected directly from the current top-level implementation surface. |
| `LogoutView` | Class | Reflected directly from the current top-level implementation surface. |
| `MeView` | Class | Reflected directly from the current top-level implementation surface. |
| `ChangePasswordView` | Class | Reflected directly from the current top-level implementation surface. |
| `UserAdminViewSet` | Class | Reflected directly from the current top-level implementation surface. |
| `RoleAdminViewSet` | Class | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["backend/apps/accounts/views.py\nThis document reflects the backend feature module exactly as it is implemented i"]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["apps.accounts.models\nProject Module"]
        I2["apps.accounts.permissions\nProject Module"]
        I3["apps.accounts.serializers\nProject Module"]
        I4["apps.accounts.services\nProject Module"]
        I5["apps.audit.services\nProject Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["LoginView\nClass"]
        E2["LogoutView\nClass"]
        E3["MeView\nClass"]
        E4["ChangePasswordView\nClass"]
        E5["UserAdminViewSet\nClass"]
        E6["RoleAdminViewSet\nClass"]
    end
    subgraph Role["Architectural Role"]
        R1["HTTP boundary that translates requests into validated application behavior and serialized responses."]
    end
    SRC --> I1
    SRC --> I2
    SRC --> I3
    SRC --> I4
    SRC --> I5
    SRC --> E1
    SRC --> E2
    SRC --> E3
    SRC --> E4
    SRC --> E5
    SRC --> E6
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `backend/apps/accounts/views.py` and acts as a concrete implementation boundary inside the repository. HTTP boundary that translates requests into validated application behavior and serialized responses.

From a dependency perspective, the file currently reaches into `apps.accounts.models`, `apps.accounts.permissions`, `apps.accounts.serializers`, `apps.accounts.services`, `apps.audit.services`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `LoginView`, `LogoutView`, `MeView`, `ChangePasswordView`, `UserAdminViewSet`, `RoleAdminViewSet`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
