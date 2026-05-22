# frontend/src/components/detection/StudentCard.tsx

## Related Documents

- [source](../../../../../frontend/src/components/detection/StudentCard.tsx)
- [system atlas](../../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

T072 — StudentCard component Displays tracking ID, posture, gaze directions, confidence bars, constraint violation warning.

## Architectural Role

Reusable UI layer that renders a focused interaction or presentation concern inside the frontend.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `StudentCard` | Function | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["frontend/src/components/detection/StudentCard.tsx\nT072 — StudentCard component Displays tracking ID, posture, gaze directions, con"]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["@/types/predictions\nFrontend Module"]
        I2["./StudentCard.module.css\nFrontend Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["StudentCard\nFunction"]
    end
    subgraph Role["Architectural Role"]
        R1["Reusable UI layer that renders a focused interaction or presentation concern inside the frontend."]
    end
    SRC --> I1
    SRC --> I2
    SRC --> E1
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `frontend/src/components/detection/StudentCard.tsx` and acts as a concrete implementation boundary inside the repository. Reusable UI layer that renders a focused interaction or presentation concern inside the frontend.

From a dependency perspective, the file currently reaches into `@/types/predictions`, `./StudentCard.module.css`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `StudentCard`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
