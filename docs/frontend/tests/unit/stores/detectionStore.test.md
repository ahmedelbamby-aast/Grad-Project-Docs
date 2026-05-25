# frontend/tests/unit/stores/detectionStore.test.ts

## Related Documents

- [source](../../../../../frontend/tests/unit/stores/detectionStore.test.ts)
- [system atlas](../../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

T045 — Unit tests for detectionStore Tests: add frame, clear frames, frame buffer management

## Architectural Role

Client state container that centralizes domain data and update logic for the frontend runtime.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `file-scoped behavior` | Internal | This file is primarily declarative or locally scoped at the top level. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["frontend/tests/unit/stores/detectionStore.test.ts\nT045 — Unit tests for detectionStore Tests: add frame, clear frames, frame buffe"]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["@/stores/detectionStore\nFrontend Module"]
        I2["@/types/api\nFrontend Module"]
    end
    subgraph Surface["Reflected Surface"]
        E0["No exported top-level symbols captured
Behavior is file-scoped or declarative"]
    end
    subgraph Role["Architectural Role"]
        R1["Client state container that centralizes domain data and update logic for the frontend runtime."]
    end
    SRC --> I1
    SRC --> I2
    SRC --> E0
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `frontend/tests/unit/stores/detectionStore.test.ts` and acts as a concrete implementation boundary inside the repository. Client state container that centralizes domain data and update logic for the frontend runtime.

From a dependency perspective, the file currently reaches into `@/stores/detectionStore`, `@/types/api`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares file-scoped declarative behavior. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.

## Test Notes

- Validates ring-buffer eviction behavior via ordered `getFrames(cameraId)` snapshots.
- Validates `latestFrame(cameraId)` and O(1) metrics via `getCameraMetrics(cameraId)`.
