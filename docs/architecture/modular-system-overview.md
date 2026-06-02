# Modular System Overview

**Last updated:** 2026-05-09

## Related Documents

- [module boundary map](module-boundary-map.md)
- [runtime scenario matrix](runtime-scenario-matrix.md)
- [compatibility contracts](compatibility-contracts.md)

## Code Structure

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
classDiagram
    class FrontendAPI
    class BackendBoundaries
    class PipelineContracts
    class TrackingContracts
    FrontendAPI --> BackendBoundaries
    BackendBoundaries --> PipelineContracts
    BackendBoundaries --> TrackingContracts
```

The class diagram shows frontend adapters consuming backend boundary records while backend workflows consume pipeline and tracking contracts.

## System Interaction

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TD
    UI["Frontend pages/stores"] --> API["Frontend API adapters"]
    API --> B["Backend boundaries"]
    B --> P["Pipeline contracts"]
    B --> T["Tracking contracts"]
    B --> H["Health/deployment boundaries"]
```

The flowchart captures the allowed runtime direction from UI through adapters into backend public contracts.

## Cross Interaction

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
sequenceDiagram
    participant Page
    participant API
    participant Backend
    participant Pipeline
    participant Tracking
    Page->>API: request workflow data
    API->>Backend: typed contract call
    Backend->>Pipeline: inference request
    Pipeline-->>Backend: normalized response
    Backend->>Tracking: detection boxes
    Tracking-->>Backend: track output contract
    Backend-->>API: public DTO
```

The sequence diagram shows cross-module communication through DTOs and contract serializers.
