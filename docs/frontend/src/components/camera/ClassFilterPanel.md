# frontend/src/components/camera/ClassFilterPanel.tsx

## Related Documents

- [source](../../../../../frontend/src/components/camera/ClassFilterPanel.tsx)
- [system atlas](../../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

ClassFilterPanel — T061 (US3: Bounding Box Class Filtering) Collapsible panel listing all detection classes with Toggle switches, "Show All" button, and a min-confidence slider.

## Architectural Role

Reusable UI layer that renders a focused interaction or presentation concern inside the frontend.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `ClassFilterPanel` | Function | Reflected directly from the current top-level implementation surface. |
| `ClassFilterPanelProps` | Interface | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["frontend/src/components/camera/ClassFilterPanel.tsx\nClassFilterPanel — T061 (US3: Bounding Box Class Filtering) Collapsible panel li"]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["@/stores/detectionStore\nFrontend Module"]
        I2["@/stores/detectionStore\nFrontend Module"]
        I3["@/components/ui\nFrontend Module"]
        I4["./ClassFilterPanel.module.css\nFrontend Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["ClassFilterPanel\nFunction"]
        E2["ClassFilterPanelProps\nInterface"]
    end
    subgraph Role["Architectural Role"]
        R1["Reusable UI layer that renders a focused interaction or presentation concern inside the frontend."]
    end
    SRC --> I1
    SRC --> I2
    SRC --> I3
    SRC --> I4
    SRC --> E1
    SRC --> E2
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `frontend/src/components/camera/ClassFilterPanel.tsx` and acts as a concrete implementation boundary inside the repository. Reusable UI layer that renders a focused interaction or presentation concern inside the frontend.

From a dependency perspective, the file currently reaches into `@/stores/detectionStore`, `@/stores/detectionStore`, `@/components/ui`, `./ClassFilterPanel.module.css`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `ClassFilterPanel`, `ClassFilterPanelProps`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
