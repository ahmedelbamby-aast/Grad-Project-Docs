# backend/apps/tracking/reid.py

## Related Documents

- [source](../../../../backend/apps/tracking/reid.py)
- [system atlas](../../../diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](../../../diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

Re-identification helpers for student re-entry handling.

## Architectural Role

Python implementation module that contributes behavior to the repository runtime.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `ReIdCandidate` | Class | Reflected directly from the current top-level implementation surface. |
| `cosine_similarity` | Function | Reflected directly from the current top-level implementation surface. |
| `is_reid_match` | Function | Reflected directly from the current top-level implementation surface. |
| `find_best_reid_match` | Function | Reflected directly from the current top-level implementation surface. |
| `prune_stale_reid_candidates` | Function | Reflected directly from the current top-level implementation surface. |
| `clear_stale_embedding_cache` | Function | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["backend/apps/tracking/reid.py\nRe-identification helpers for student re-entry handling."]
    end
    subgraph Dependencies["Project Dependencies"]
        I1["apps.tracking.embeddings\nProject Module"]
    end
    subgraph Surface["Reflected Surface"]
        E1["ReIdCandidate\nClass"]
        E2["cosine_similarity\nFunction"]
        E3["is_reid_match\nFunction"]
        E4["find_best_reid_match\nFunction"]
        E5["prune_stale_reid_candidates\nFunction"]
        E6["clear_stale_embedding_cache\nFunction"]
    end
    subgraph Role["Architectural Role"]
        R1["Python implementation module that contributes behavior to the repository runtime."]
    end
    SRC --> I1
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

This module sits at `backend/apps/tracking/reid.py` and acts as a concrete implementation boundary inside the repository. Python implementation module that contributes behavior to the repository runtime.

From a dependency perspective, the file currently reaches into `apps.tracking.embeddings`. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `ReIdCandidate`, `cosine_similarity`, `is_reid_match`, `find_best_reid_match`, `prune_stale_reid_candidates`, `clear_stale_embedding_cache`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
