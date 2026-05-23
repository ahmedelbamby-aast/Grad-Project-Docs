# Video Analysis Module

## Boundary Summary

| Field | Value |
| --- | --- |
| Purpose | Offline upload, jobs, batch processing, and preview orchestration |
| Responsibilities | Manage offline jobs; publish processed video results |
| Public inputs | Raw video; job commands |
| Public outputs | Job status; stored results; annotated media |
| Consumers | Frontend offline video UI, exports, recordings |
| Dependencies | Pipeline, tracking, storage |
| Failure behavior | Job failures persist retry-safe error status |

## Offline Prediction Flow

```mermaid
flowchart TD
    V["Raw video"] --> J["Video analysis job"]
    J --> P["Pipeline inference contract"]
    P --> T["Tracking output contract"]
    T --> R["Annotated media and stored results"]
```

The flow keeps offline orchestration behind job and pipeline contracts.

## Telemetry REST Contract

Job-scoped telemetry endpoints project normalized persisted evidence:

- `GET /api/v1/video-analysis/jobs/{job_id}/telemetry/summary/`
- `GET /api/v1/video-analysis/jobs/{job_id}/telemetry/timeline/`
- `GET /api/v1/video-analysis/jobs/{job_id}/telemetry/artifacts/`
- `GET /api/v1/video-analysis/jobs/{job_id}/runtime-assignments/`
- `GET /api/v1/video-analysis/jobs/{job_id}/benchmark-comparison/`

Runtime dashboard endpoints are also served here (`/api/v1/video-analysis/runtime/*`) for sessions, frame/model events, summary, issues, filters, timeline, matrix, student timeline, and profiles.

## Telemetry WebSocket Contract

- Job transport route: `WS /ws/video-analysis/jobs/{job_id}/`
- Telemetry message types carried on this route include:
  - `telemetry.ready`
  - `telemetry.subscribed`
  - `telemetry.update`
- `telemetry.update` is also emitted for terminal job states (`completed`, `failed`) with `event_type` in payload.

## Authorization and Projection Rules

- Telemetry responses/events are authenticated and role-gated before projection.
- Timeline and list-style telemetry responses are bounded/paged by server limits.
- Serialized telemetry must remain PII-safe and aligned with normalized backend evidence.
- Dashboard telemetry delivery is isolated from MCP availability/failure.
