# backend/apps/cameras/constants.py

## Related Documents

- [source](../../../../backend/apps/cameras/constants.py)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

This document reflects the backend feature module exactly as it is implemented in the repository today.

## Architectural Role

Python implementation module that contributes behavior to the repository runtime.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `RTSP_PROBE_TIMEOUT` | Constant | Reflected directly from the current top-level implementation surface. |
| `HEALTH_CHECK_INTERVAL` | Constant | Reflected directly from the current top-level implementation surface. |
| `MAX_RECONNECT_RETRIES` | Constant | Reflected directly from the current top-level implementation surface. |
| `RECONNECT_BASE_DELAY` | Constant | Reflected directly from the current top-level implementation surface. |
| `CONNECTION_EVENT_RETENTION_DAYS` | Constant | Reflected directly from the current top-level implementation surface. |
| `MAX_CAMERAS_PER_INSTRUCTOR` | Constant | Reflected directly from the current top-level implementation surface. |
| `BULK_FAILURE_WINDOW` | Constant | Reflected directly from the current top-level implementation surface. |
| `BULK_FAILURE_THRESHOLD` | Constant | Reflected directly from the current top-level implementation surface. |
| `ERROR_MESSAGES` | Constant | Reflected directly from the current top-level implementation surface. |
| `CameraRuntimeConfig` | Class | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["backend/apps/cameras/constants.py\nThis document reflects the backend feature module exactly as it is implemented i"]
    end
    subgraph Dependencies["Project Dependencies"]
        I0["No project-local imports captured
File is locally scoped or infrastructure-only"]
    end
    subgraph Surface["Reflected Surface"]
        E1["RTSP_PROBE_TIMEOUT\nConstant"]
        E2["HEALTH_CHECK_INTERVAL\nConstant"]
        E3["MAX_RECONNECT_RETRIES\nConstant"]
        E4["RECONNECT_BASE_DELAY\nConstant"]
        E5["CONNECTION_EVENT_RETENTION_DAYS\nConstant"]
        E6["MAX_CAMERAS_PER_INSTRUCTOR\nConstant"]
        E7["BULK_FAILURE_WINDOW\nConstant"]
        E8["BULK_FAILURE_THRESHOLD\nConstant"]
        E9["ERROR_MESSAGES\nConstant"]
        E10["CameraRuntimeConfig\nClass"]
    end
    subgraph Role["Architectural Role"]
        R1["Python implementation module that contributes behavior to the repository runtime."]
    end
    SRC --> I0
    SRC --> E1
    SRC --> E2
    SRC --> E3
    SRC --> E4
    SRC --> E5
    SRC --> E6
    SRC --> E7
    SRC --> E8
    SRC --> E9
    SRC --> E10
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `backend/apps/cameras/constants.py` and acts as a concrete implementation boundary inside the repository. Python implementation module that contributes behavior to the repository runtime.

From a dependency perspective, the file currently reaches into no project-local imports in the captured surface. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `RTSP_PROBE_TIMEOUT`, `HEALTH_CHECK_INTERVAL`, `MAX_RECONNECT_RETRIES`, `RECONNECT_BASE_DELAY`, `CONNECTION_EVENT_RETENTION_DAYS`, `MAX_CAMERAS_PER_INSTRUCTOR`, `BULK_FAILURE_WINDOW`, `BULK_FAILURE_THRESHOLD`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
