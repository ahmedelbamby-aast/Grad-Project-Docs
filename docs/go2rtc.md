# go2rtc.yaml

**Last updated:** 2026-05-09

## Related Documents

- [source](../go2rtc.yaml)
- [system atlas](diagrams/SYSTEM_MERMAID_ATLAS.md)
- [source mirror](diagrams/SOURCE_FILE_MIRROR.md)

## Executive View

This document reflects the current repository-owned implementation surface.

## Architectural Role

Runtime configuration file that wires infrastructure, process management, or proxy behavior.

## Reflected Surface

| Symbol | Kind | Reflection |
|------|------|------------|
| `api:` | Constant | Reflected directly from the current top-level implementation surface. |
| `listen: ":1984"` | Constant | Reflected directly from the current top-level implementation surface. |
| `webrtc:` | Constant | Reflected directly from the current top-level implementation surface. |
| `listen: ":8555"` | Constant | Reflected directly from the current top-level implementation surface. |
| `candidates:` | Constant | Reflected directly from the current top-level implementation surface. |
| `- 127.0.0.1:8555` | Constant | Reflected directly from the current top-level implementation surface. |
| `streams: {}` | Constant | Reflected directly from the current top-level implementation surface. |

## Architecture Diagram

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Source["Current Module"]
        SRC["go2rtc.yaml\nThis document reflects the current repository-owned implementation surface."]
    end
    subgraph Dependencies["Project Dependencies"]
        I0["No project-local imports captured
File is locally scoped or infrastructure-only"]
    end
    subgraph Surface["Reflected Surface"]
        E1["api:\nConstant"]
        E2["listen: ':1984'\nConstant"]
        E3["webrtc:\nConstant"]
        E4["listen: ':8555'\nConstant"]
        E5["candidates:\nConstant"]
        E6["- 127.0.0.1:8555\nConstant"]
        E7["streams: {}\nConstant"]
    end
    subgraph Role["Architectural Role"]
        R1["Runtime configuration file that wires infrastructure, process management, or proxy behavior."]
    end
    SRC --> I0
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

This module sits at `go2rtc.yaml` and acts as a concrete implementation boundary inside the repository. Runtime configuration file that wires infrastructure, process management, or proxy behavior.

From a dependency perspective, the file currently reaches into no project-local imports in the captured surface. Those links were read from the real source file so the diagram reflects the actual local coupling rather than an inferred architecture.

From a surface perspective, the top-level implementation currently exposes or declares `api:`, `listen: ":1984"`, `webrtc:`, `listen: ":8555"`, `candidates:`, `- 127.0.0.1:8555`, `streams: {}`. That reflected surface is intentionally tied to the source file itself, so if the code changes the document should be regenerated with it.

From an accuracy perspective, this page focuses on project-local structure: repository imports, top-level classes/functions/constants, and the architectural role implied by the file location and concrete implementation type. External library imports are intentionally omitted from the diagram so the repository interaction map remains readable while staying faithful to the codebase.
