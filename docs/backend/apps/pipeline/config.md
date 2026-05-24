# backend/apps/pipeline/config.py

## Related Documents

- [source](../../../../backend/apps/pipeline/config.py)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

Pipeline configuration using Pydantic v2.

## Architectural Role

Python implementation module that contributes behavior to the repository runtime.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `FrameMeta` | Class | Reflected directly from the current top-level implementation surface. |
| `PersonDetectionResult` | Class | Reflected directly from the current top-level implementation surface. |
| `CroppedStudentImage` | Class | Reflected directly from the current top-level implementation surface. |
| `BehaviorPrediction` | Class | Reflected directly from the current top-level implementation surface. |
| `MergedBehaviorRecord` | Class | Reflected directly from the current top-level implementation surface. |
| `PipelineConfig` | Class | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["backend/apps/pipeline/config.py\nPipeline configuration using Pydantic v2."]
    end
    subgraph Dependencies["Project Dependencies"]
        I0["No project-local imports captured
File is locally scoped or infrastructure-only"]
    end
    subgraph Surface["Reflected Surface"]
        E1["FrameMeta\nClass"]
        E2["PersonDetectionResult\nClass"]
        E3["CroppedStudentImage\nClass"]
        E4["BehaviorPrediction\nClass"]
        E5["MergedBehaviorRecord\nClass"]
        E6["PipelineConfig\nClass"]
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
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `backend/apps/pipeline/config.py` and acts as a concrete implementation boundary inside the repository. Python implementation module that contributes behavior to the repository runtime.

From a dependency perspective, the file currently reaches into no project-local imports in the captured surface. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `FrameMeta`, `PersonDetectionResult`, `CroppedStudentImage`, `BehaviorPrediction`, `MergedBehaviorRecord`, `PipelineConfig`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.

## Stream Hardening Settings

The pipeline config now includes explicit runtime stream controls:

- `stream_read_timeout_ms`
- `stream_frame_timeout_ms`
- `stream_buffer_max_frames`
- `stream_queue_max_events`
- `stream_max_empty_reads`

Tracking lifecycle timeout controls are also available:

- `tracking_lost_timeout_frames`
- `tracking_lost_timeout_ms`
