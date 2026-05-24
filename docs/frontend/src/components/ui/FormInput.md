# frontend/src/components/ui/FormInput.tsx

## Related Documents

- [source](../../../../../frontend/src/components/ui/FormInput.tsx)
- [system atlas](../../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

FormInput — labelled text input with error / helper message.

## Architectural Role

Reusable UI layer that renders a focused interaction or presentation concern inside the frontend.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `FormInput` | Export | Reflected directly from the current top-level implementation surface. |
| `FormInputProps` | Interface | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["frontend/src/components/ui/FormInput.tsx\nFormInput — labelled text input with error / helper message."]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["./FormInput.module.css\nFrontend Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["FormInput\nExport"]
        E2["FormInputProps\nInterface"]
    end
    subgraph Role["Architectural Role"]
        R1["Reusable UI layer that renders a focused interaction or presentation concern inside the frontend."]
    end
    SRC --> I1
    SRC --> E1
    SRC --> E2
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `frontend/src/components/ui/FormInput.tsx` and acts as a concrete implementation boundary inside the repository. Reusable UI layer that renders a focused interaction or presentation concern inside the frontend.

From a dependency perspective, the file currently reaches into `./FormInput.module.css`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `FormInput`, `FormInputProps`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
