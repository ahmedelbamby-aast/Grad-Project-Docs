# backend/apps/pipeline/services.py

## Related Documents

- [source](../../../../backend/apps/pipeline/services.py)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

This document reflects the backend feature module exactly as it is implemented in the repository today.

## Architectural Role

Service layer that concentrates orchestration logic away from transport and persistence edges.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `PipelineService` | Class | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["backend/apps/pipeline/services.py\nThis document reflects the backend feature module exactly as it is implemented i"]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["apps.pipeline.cropper\nProject Module"]
        I2["apps.pipeline.detector\nProject Module"]
        I3["apps.pipeline.layers\nProject Module"]
        I4["apps.pipeline.merger\nProject Module"]
        I5["apps.pipeline.rule_engine\nProject Module"]
        I6["apps.pipeline.tracker\nProject Module"]
        I7["apps.pipeline.model_lifecycle\nProject Module"]
        I8["apps.pipeline.config\nProject Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["PipelineService\nClass"]
    end
    subgraph Role["Architectural Role"]
        R1["Service layer that concentrates orchestration logic away from transport and persistence edges."]
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
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `backend/apps/pipeline/services.py` and acts as a concrete implementation boundary inside the repository. Service layer that concentrates orchestration logic away from transport and persistence edges.

From a dependency perspective, the file currently reaches into `apps.pipeline.cropper`, `apps.pipeline.detector`, `apps.pipeline.layers`, `apps.pipeline.merger`, `apps.pipeline.rule_engine`, `apps.pipeline.tracker`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `PipelineService`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.

## Runtime Payload Additions

Runtime lifecycle payloads can include stream timing and pressure fields:

- `source_timestamp_ms`
- `ingest_timestamp_ms`
- `frame_delta_ms`
- `effective_fps`
- `buffer_overflow_count`
- `backpressure_event_count`

Detector benchmark payloads expose:

- `fps`
- `latency_ms`
