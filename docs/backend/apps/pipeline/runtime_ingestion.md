# backend/apps/pipeline/runtime_ingestion.py

## Source
- [backend/apps/pipeline/runtime_ingestion.py](../../../../../backend/apps/pipeline/runtime_ingestion.py)

## Purpose

Persists runtime telemetry as immutable raw events plus hourly rollups, with optional async ingestion through Celery.

## Main structures and functions

- `RuntimeIngestPayload`: typed ingest contract with `to_dict/from_dict`.
- `ingest_runtime_event(...)`: writes `RuntimeRawTruthEvent` with ordered `ingest_sequence`.
- `update_runtime_rollup(...)`: upserts `RuntimeSessionRollup` and updates counters/latency.
- `process_runtime_ingestion(...)`: synchronous ingest pipeline.
- `enqueue_runtime_ingestion(...)`: async-first with inline fallback.

## Data path

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart LR
    Producer["pipeline/video_analysis runtime producer"] --> Payload["RuntimeIngestPayload"]
    Payload --> Enqueue["enqueue_runtime_ingestion()"]
    Enqueue -->|async enabled| Task["apps.pipeline.tasks.ingest_runtime_event_task"]
    Enqueue -->|fallback| Inline["process_runtime_ingestion()"]
    Task --> Inline
    Inline --> Raw["RuntimeRawTruthEvent (immutable)"]
    Inline --> Rollup["RuntimeSessionRollup (hour bucket)"]
```

## Cross-links

- [tasks.md](tasks.md)
- [services/runtime_comparison.md](services/runtime_comparison.md)
- [../../architecture/data-flow.md](../../architecture/data-flow.md)

