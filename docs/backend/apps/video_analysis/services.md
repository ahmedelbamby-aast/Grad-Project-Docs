# Video Analysis Runtime Services

Runtime API extensions:

- `GET /video-analysis/runtime/timeline/` includes:
  - Timing: `source_timestamp_ms`, `ingest_timestamp_ms`, `frame_delta_ms`, `effective_fps`.
  - Observability blocks: `buffer`, `lifecycle`, `provider`, `resources`.
- `GET /video-analysis/runtime/student-timeline/` timeline points include:
  - `source_timestamp_ms`, `lifecycle_state`.
- `GET /video-analysis/runtime/summary/` KPI block includes:
  - `avg_gpu_utilization`, `avg_memory_mb`, `avg_queue_depth`.

## Temporal Buffer Health
- Runtime timeline payloads include temporal freshness/buffer surfaces from persisted payload metadata (uffer, reshness, lifecycle timestamps) for API consumers.
