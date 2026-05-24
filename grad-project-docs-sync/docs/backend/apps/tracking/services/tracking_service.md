# backend/apps/tracking/services/tracking_service.py

## Related Documents

- [source](../../../../../backend/apps/tracking/services/tracking_service.py)
- [system atlas](../../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

Tracking service with explicit DTO boundaries (T040).

## Architectural Role

Service layer that concentrates orchestration logic away from transport and persistence edges.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `TRACKING_OUTPUT_CONTRACT` | Constant | Reflected directly from the current top-level implementation surface. |
| `BoundingBoxDTO` | Class | Reflected directly from the current top-level implementation surface. |
| `FrameDetectionDTO` | Class | Reflected directly from the current top-level implementation surface. |
| `TrackStateDTO` | Class | Reflected directly from the current top-level implementation surface. |
| `TrackingResultDTO` | Class | Reflected directly from the current top-level implementation surface. |
| `TrackingService` | Class | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["backend/apps/tracking/services/tracking_service.py\nTracking service with explicit DTO boundaries (T040)."]
    end
    subgraph Dependencies["Project Dependencies"]
        I0["No project-local imports captured
File is locally scoped or infrastructure-only"]
    end
    subgraph Surface["Reflected Surface"]
        E1["TRACKING_OUTPUT_CONTRACT\nConstant"]
        E2["BoundingBoxDTO\nClass"]
        E3["FrameDetectionDTO\nClass"]
        E4["TrackStateDTO\nClass"]
        E5["TrackingResultDTO\nClass"]
        E6["TrackingService\nClass"]
    end
    subgraph Role["Architectural Role"]
        R1["Service layer that concentrates orchestration logic away from transport and persistence edges."]
    end
    SRC --> I0
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

This module sits at `backend/apps/tracking/services/tracking_service.py` and acts as a concrete implementation boundary inside the repository. Service layer that concentrates orchestration logic away from transport and persistence edges.

From a dependency perspective, the file currently reaches into no project-local imports in the captured surface. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `TRACKING_OUTPUT_CONTRACT`, `BoundingBoxDTO`, `FrameDetectionDTO`, `TrackStateDTO`, `TrackingResultDTO`, `TrackingService`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.

## Lifecycle and Continuity Semantics

Tracking lifecycle handling now supports explicit states and recovery-oriented behavior:

- States: `active`, `occluded`, `reidentified`, `lost`, `ended`
- Lost timeout checks support both frame-based and ms-based thresholds.
- Desync recovery emits `tracker_desync_recovery` events when time moves backward.

Continuity scoring is available per track through `compute_track_continuity_score(...)` and can be emitted through track metadata.
