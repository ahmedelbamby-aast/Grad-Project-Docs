# frontend/src/components/camera/CameraFeed.tsx

## Related Documents

- [source](../../../../../frontend/src/components/camera/CameraFeed.tsx)
- [system atlas](../../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

CameraFeed — T055 Single camera feed tile combining: • <video> (z-index 0) — WebRTC stream via useWhepClient / go2rtc WHEP • BoundingBoxCanvas overlay (z-index 1) — live detection boxes • Connection status badge (top-left) • "Connection Lost — Reconnecting..." banner on disconnect Accessibility: role="img" with aria-label containing camera name.

## Architectural Role

Reusable UI layer that renders a focused interaction or presentation concern inside the frontend.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `CameraFeed` | Function | Reflected directly from the current top-level implementation surface. |
| `CameraFeedProps` | Interface | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["frontend/src/components/camera/CameraFeed.tsx\nCameraFeed — T055 Single camera feed tile combining: • <video> (z-index 0) — Web"]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["@/hooks/useWhepClient\nFrontend Module"]
        I2["@/stores/detectionStore\nFrontend Module"]
        I3["@/stores/cameraStore\nFrontend Module"]
        I4["./BoundingBoxCanvas\nFrontend Module"]
        I5["./ErrorBanner\nFrontend Module"]
        I6["@/components/ui\nFrontend Module"]
        I7["@/types/api\nFrontend Module"]
        I8["./CameraFeed.module.css\nFrontend Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["CameraFeed\nFunction"]
        E2["CameraFeedProps\nInterface"]
    end
    subgraph Role["Architectural Role"]
        R1["Reusable UI layer that renders a focused interaction or presentation concern inside the frontend."]
    end
    SRC --> I1
    SRC --> I2
    SRC --> I3
    SRC --> I4
    SRC --> I5
    SRC --> I6
    SRC --> I7
    SRC --> I8
    SRC --> E1
    SRC --> E2
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `frontend/src/components/camera/CameraFeed.tsx` and acts as a concrete implementation boundary inside the repository. Reusable UI layer that renders a focused interaction or presentation concern inside the frontend.

From a dependency perspective, the file currently reaches into `@/hooks/useWhepClient`, `@/stores/detectionStore`, `@/stores/cameraStore`, `./BoundingBoxCanvas`, `./ErrorBanner`, `@/components/ui`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `CameraFeed`, `CameraFeedProps`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.

## Runtime Notes

- Live object density indicators are derived from `useDetectionStore().getCameraMetrics(camera.id)` to avoid per-render reduction over full frame arrays.
- Latest overlay boxes continue to come from `latestFrame(camera.id)` for freshness-first rendering.
