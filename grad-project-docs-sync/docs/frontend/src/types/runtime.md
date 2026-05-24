# frontend/src/types/runtime.ts

## Purpose

Shared TypeScript contracts for `/runtime` API responses.

## Rollout Fields

- Runtime event/session payload types now include optional rollout metadata:
  - `rollout_mode`
  - `rollout_reason`

These fields allow UI-level visibility into shadow/canary/cutover routing decisions.

## Observability Extensions

- `RuntimeSummaryKpis` adds:
  - `avg_gpu_utilization`
  - `avg_memory_mb`
  - `avg_queue_depth`
- `RuntimeTimelineItem` adds:
  - timing fields (`source_timestamp_ms`, `ingest_timestamp_ms`, `frame_delta_ms`, `effective_fps`)
  - grouped fields (`buffer`, `lifecycle`, `provider`, `resources`)
- `RuntimeStudentTimelinePoint` adds:
  - `source_timestamp_ms`
  - `lifecycle_state`
