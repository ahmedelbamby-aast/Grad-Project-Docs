# backend/apps/pipeline/runtime_ingestion.py

## Source
- [backend/apps/pipeline/runtime_ingestion.py](../../../../../backend/apps/pipeline/runtime_ingestion.py)

## Purpose
Ingests normalized runtime events, maintains rollups, persists buffering telemetry sidecars, and projects governed dashboard telemetry views.

## Main structures and functions
- `RuntimeIngestPayload`: typed runtime ingest contract with serialization helpers.
- `BoundedEventBuffer`: bounded append-only in-memory event buffer (drop-oldest).
- `StreamClock`: monotonic timestamp helper for frame delta and effective FPS.
- `BufferTelemetryEvent`: queue/spill/overflow telemetry envelope.
- `write_buffer_pressure_event(...)`: JSONL buffer telemetry writer with optional normalized ingest.
- `ingest_runtime_event(...)`: immutable raw event persistence with ordered ingest sequence.
- `update_runtime_rollup(...)`: hourly rollup upsert with worker health/backlog counters.
- `enqueue_runtime_ingestion(...)`: async-first ingestion with inline fallback.
- Projection APIs:
  - `project_dashboard_telemetry_summary(...)`
  - `project_dashboard_telemetry_timeline(...)`
  - `project_dashboard_telemetry_artifacts(...)`
  - `project_dashboard_runtime_assignments(...)`
  - `project_dashboard_worker_isolation_rollups(...)`
  - `project_dashboard_benchmark_comparison(...)`

## Feature alignment
- Mirrors queue/spill/overflow telemetry persistence required for buffering behavior.
- Mirrors per-model worker health/backlog/timeout/error rollups.
- Mirrors normalized dashboard telemetry projections used by REST and MCP layers.

## Cross-links
- [buffering.md](buffering.md)
- [failure_injection.md](failure_injection.md)
- [reconciliation.md](reconciliation.md)
- [../video_analysis/views.md](../video_analysis/views.md)
