# backend/apps/tracking/tracker.py

## Related Documents

- [source](../../../../backend/apps/tracking/tracker.py)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

Tracker wrappers for YOLO-based student tracking.

## Architectural Role

Python implementation module that contributes behavior to the repository runtime.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `_TRACKER_YAML_DIR` | Constant | Reflected directly from the current top-level implementation surface. |
| `_BOTSORT_REID_YAML` | Constant | Reflected directly from the current top-level implementation surface. |
| `_BYTETRACK_LOW_COMPUTE_YAML` | Constant | Reflected directly from the current top-level implementation surface. |
| `_ensure_custom_yamls` | Function | Reflected directly from the current top-level implementation surface. |
| `TrackerResult` | Class | Reflected directly from the current top-level implementation surface. |
| `TrackerConfiguration` | Class | Reflected directly from the current top-level implementation surface. |
| `ByteSortTracker` | Class | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["backend/apps/tracking/tracker.py\nTracker wrappers for YOLO-based student tracking."]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["apps.pipeline.openvino_compat\nProject Module"]
        I2["apps.pipeline.config\nProject Module"]
        I3["apps.pipeline\nProject Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["_TRACKER_YAML_DIR\nConstant"]
        E2["_BOTSORT_REID_YAML\nConstant"]
        E3["_BYTETRACK_LOW_COMPUTE_YAML\nConstant"]
        E4["_ensure_custom_yamls\nFunction"]
        E5["TrackerResult\nClass"]
        E6["TrackerConfiguration\nClass"]
        E7["ByteSortTracker\nClass"]
    end
    subgraph Role["Architectural Role"]
        R1["Python implementation module that contributes behavior to the repository runtime."]
    end
    SRC --> I1
    SRC --> I2
    SRC --> I3
    SRC --> E1
    SRC --> E2
    SRC --> E3
    SRC --> E4
    SRC --> E5
    SRC --> E6
    SRC --> E7
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `backend/apps/tracking/tracker.py` and acts as a concrete implementation boundary inside the repository. Python implementation module that contributes behavior to the repository runtime.

From a dependency perspective, the file currently reaches into `apps.pipeline.openvino_compat`, `apps.pipeline.config`, `apps.pipeline`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `_TRACKER_YAML_DIR`, `_BOTSORT_REID_YAML`, `_BYTETRACK_LOW_COMPUTE_YAML`, `_ensure_custom_yamls`, `TrackerResult`, `TrackerConfiguration`, `ByteSortTracker`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
