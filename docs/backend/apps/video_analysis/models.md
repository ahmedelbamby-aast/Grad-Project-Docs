# backend/apps/video_analysis/models.py

## Related Documents

- [source](../../../../backend/apps/video_analysis/models.py)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

Video analysis models for upload, processing, and overlay playback.

## Architectural Role

Persistence boundary that defines the database entities and relationships used by the surrounding feature.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `COLOR_HEX_VALIDATOR` | Constant | Reflected directly from the current top-level implementation surface. |
| `MODEL_NAME_CHOICES` | Constant | Reflected directly from the current top-level implementation surface. |
| `PIPELINE_MODE_CHOICES` | Constant | Reflected directly from the current top-level implementation surface. |
| `VIDEO_CLASS_CHOICES` | Constant | Reflected directly from the current top-level implementation surface. |
| `_validate_xyxy` | Function | Reflected directly from the current top-level implementation surface. |
| `_validate_embedding_vector` | Function | Reflected directly from the current top-level implementation surface. |
| `VideoAnalysisJob` | Class | Reflected directly from the current top-level implementation surface. |
| `Frame` | Class | Reflected directly from the current top-level implementation surface. |
| `Detection` | Class | Reflected directly from the current top-level implementation surface. |
| `StudentTrack` | Class | Reflected directly from the current top-level implementation surface. |
| `FrameEmbedding` | Class | Reflected directly from the current top-level implementation surface. |
| `BoundingBox` | Class | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["backend/apps/video_analysis/models.py\nVideo analysis models for upload, processing, and overlay playback."]
    end
    subgraph Dependencies["Project Dependencies"]
        I0["No project-local imports captured
File is locally scoped or infrastructure-only"]
    end
    subgraph Surface["Reflected Surface"]
        E1["COLOR_HEX_VALIDATOR\nConstant"]
        E2["MODEL_NAME_CHOICES\nConstant"]
        E3["PIPELINE_MODE_CHOICES\nConstant"]
        E4["VIDEO_CLASS_CHOICES\nConstant"]
        E5["_validate_xyxy\nFunction"]
        E6["_validate_embedding_vector\nFunction"]
        E7["VideoAnalysisJob\nClass"]
        E8["Frame\nClass"]
        E9["Detection\nClass"]
        E10["StudentTrack\nClass"]
        E11["FrameEmbedding\nClass"]
        E12["BoundingBox\nClass"]
    end
    subgraph Role["Architectural Role"]
        R1["Persistence boundary that defines the database entities and relationships used by the surrounding feature."]
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
    SRC --> E11
    SRC --> E12
    SRC --> R1
```

This diagram uses the same visual language as the root architecture view: one subgraph for the current module, one for concrete repository dependencies, one for the reflected implementation surface, and one for the architectural role that the file currently occupies.

## Detailed Reflection

This module sits at `backend/apps/video_analysis/models.py` and acts as a concrete implementation boundary inside the repository. Persistence boundary that defines the database entities and relationships used by the surrounding feature.

From a dependency perspective, the file currently reaches into no project-local imports in the captured surface. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `COLOR_HEX_VALIDATOR`, `MODEL_NAME_CHOICES`, `PIPELINE_MODE_CHOICES`, `VIDEO_CLASS_CHOICES`, `_validate_xyxy`, `_validate_embedding_vector`, `VideoAnalysisJob`, `Frame`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
