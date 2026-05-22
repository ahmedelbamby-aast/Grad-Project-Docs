# frontend/src/pages/CameraFeedPage.tsx

## Related Documents

- [source](../../../../frontend/src/pages/CameraFeedPage.tsx)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

CameraFeedPage — T057 Full camera monitoring page with: • Session management (start / end session) • Camera list sidebar (add, remove, connect, disconnect) • CameraGrid main area (live WebRTC feeds + bbox overlays) • Det

## Architectural Role

Route-level UI composition layer that binds stores, hooks, API calls, and visual components into a user workflow.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `CAMERA_FEED_PAGE_INTERACTION_CONTRACT` | Export | Reflected directly from the current top-level implementation surface. |
| `CameraFeedPage` | Function | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["frontend/src/pages/CameraFeedPage.tsx\nCameraFeedPage — T057 Full camera monitoring page with: • Session management (st"]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["@/stores/cameraStore\nFrontend Module"]
        I2["@/hooks/useDetectionSocket\nFrontend Module"]
        I3["@/hooks/useCameraStatus\nFrontend Module"]
        I4["@/stores/visibilityStore\nFrontend Module"]
        I5["@/api/cameras\nFrontend Module"]
        I6["@/api/sessions\nFrontend Module"]
        I7["@/components/ui\nFrontend Module"]
        I8["@/components/camera/CameraGrid\nFrontend Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["CAMERA_FEED_PAGE_INTERACTION_CONTRACT\nExport"]
        E2["CameraFeedPage\nFunction"]
    end
    subgraph Role["Architectural Role"]
        R1["Route-level UI composition layer that binds stores, hooks, API calls, and visual components into a user workflow."]
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

This module sits at `frontend/src/pages/CameraFeedPage.tsx` and acts as a concrete implementation boundary inside the repository. Route-level UI composition layer that binds stores, hooks, API calls, and visual components into a user workflow.

From a dependency perspective, the file currently reaches into `@/stores/cameraStore`, `@/hooks/useDetectionSocket`, `@/hooks/useCameraStatus`, `@/stores/visibilityStore`, `@/api/cameras`, `@/api/sessions`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `CAMERA_FEED_PAGE_INTERACTION_CONTRACT`, `CameraFeedPage`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
