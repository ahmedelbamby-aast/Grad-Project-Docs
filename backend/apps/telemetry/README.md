# Telemetry Layer

**Last updated:** 2026-06-01

Single authoritative layer for collecting, computing, and persisting run metrics
across all execution modes: offline video processing, RTSP live streaming, and
test/benchmark runs.

## Package layout

```
backend/apps/telemetry/
├── apps.py          — Django AppConfig
├── models.py        — ORM models (six tables)
├── session.py       — TelemetrySession public API
├── metrics.py       — Pure statistical helpers (percentile, mean, …)
├── writer.py        — Atomic DB + JSON dual-sink writer
├── migrations/
│   ├── 0001_initial.py
│   └── 0002_telemetrylpmevent.py
└── README.md        — this file
```

JSON output directory: `backend/logs/telemetry/`

---

## Table schema

### `telemetry_sessions`

| Column | Type | Description |
|--------|------|-------------|
| `session_id` | UUID PK | Run identifier |
| `source_type` | varchar | `rtsp` / `offline` / `test` |
| `started_at` | timestamptz | Wall-clock session start |
| `ended_at` | timestamptz | Wall-clock session end |
| `wall_time_s` | float | Total elapsed seconds |
| `total_frames` | int | All frames seen |
| `dropped_frames` | int | Frames not processed |
| `drop_rate` | float | `dropped / total` |
| `mean_frame_latency_ms` | float | Mean pipeline latency per frame |
| `p50/p95/p99_frame_latency_ms` | float | Percentile pipeline latencies |
| `mean_rtt_per_model_ms` | JSON | `{"person_detector": 12.4, …}` |
| `p95_rtt_per_model_ms` | JSON | Same structure, p95 values |
| `peak_gpu_memory_mb` | float | Peak GPU memory usage |
| `mean_gpu_utilization_pct` | float | Average GPU utilisation |
| `triton_model_load_time_s` | float | Time for Triton to load all models |
| `runtime_mode` | varchar | `live` / `offline` |
| `triton_endpoint` | varchar | Active Triton URL |
| `celery_task_id` | varchar | Celery task that owns this session |
| `metadata` | JSON | Arbitrary context (git SHA, env, …) |
| `db_commit_ok` | bool | False if DB write failed (JSON fallback) |

### `telemetry_videos`

One row per video/stream within a session.  Key columns:
`session_id`, `video_id`, `source_type`, `processing_started_at`,
`processing_ended_at`, `target_fps`, `observed_fps`, `total_frames`,
`detection_count`, `model_call_count`, `mean_frame_latency_ms`, `p95_frame_latency_ms`.

### `telemetry_frames`

One row per processed frame (bulk-inserted ≤ 500 per batch).  Key columns:
`session_id`, `frame_index`, `wall_timestamp`, `stream_pts_ms`,
`preprocess_latency_ms`, `inference_latency_ms`, `postprocess_latency_ms`,
`total_pipeline_latency_ms`, `model_latencies_ms` (JSON), `detection_count`.

### `telemetry_model_calls`

One row per Triton inference call.  Key columns:
`session_id`, `model_name`, `model_version`, `request_sent_at`,
`response_received_at`, `rtt_ms`, `input_shape` (JSON), `status`, `error_detail`.

### `telemetry_lpm_events`

One row per frame where the Logical Path Matrix executes. Key columns:
`session_id`, `frame_index`, `person_count`, `eliminated_contradictions`,
`violations_c1`, `violations_c2`, `violations_c3`, `violations_c4`,
`latency_ms`.

### `telemetry_students`

One row per tracked student per session.  Key columns:
`session_id`, `student_id`, `canonical_track_id`, `frames_tracked`,
`mean_confidence`, `mean_embedding_distance`, `recognition_latency_ms`,
`first_seen_at`, `last_seen_at`.

---

## JSON file schema

Each run produces `backend/logs/telemetry/<session_id>_<source_type>_<YYYYMMDD_HHMMSS>.json`.

Top-level keys mirror the DB tables:

```json
{
  "session": { ... },
  "videos": [ ... ],
  "model_calls": [ ... ],
  "lpm_events": [ ... ],
  "students": [ ... ],
  "frame_count_in_json": 1234,
  "lpm_event_count_in_json": 1234,
  "db_commit_failed": false   // present only when DB write failed
}
```

The JSON file is **always** written before the DB transaction commits.
If the DB rolls back the file is retained with `"db_commit_failed": true`.

---

## Usage

### Context-manager (recommended)

```python
from apps.telemetry.session import TelemetrySession, FrameMeta, ModelCallMeta

with TelemetrySession(
    source_type="offline",
    metadata={"job_id": str(job.job_id)},
    celery_task_id=self.request.id,
) as ts:
    for frame, idx, pts_ms in pipeline:
        t0 = time.monotonic()
        # ... inference ...
        t1 = time.monotonic()
        ts.record_frame(FrameMeta(
            frame_index=idx,
            wall_timestamp=timezone.now(),
            stream_pts_ms=pts_ms,
            total_pipeline_latency_ms=(t1 - t0) * 1000,
            detection_count=len(detections),
        ))
        ts.record_model_call(ModelCallMeta(
            model_name="person_detector",
            request_sent_at=t0,
            response_received_at=t1,
            status="ok",
        ))
```

### Explicit form

```python
ts = TelemetrySession(source_type="rtsp")
session_id = ts.start()
# ... process frames ...
summary = ts.end()
print(summary.p95_frame_latency_ms)
```

### Test harness

```python
from apps.telemetry.session import TelemetrySession

def test_inference_benchmark():
    with TelemetrySession(source_type="test", metadata={"test": "benchmark_x"}) as ts:
        # run inference ...
        ts.record_frame(FrameMeta(...))
```

---

## Common queries

### p95 inference latency for the last 10 sessions

```sql
SELECT session_id, source_type, p95_frame_latency_ms, wall_time_s
FROM telemetry_sessions
ORDER BY started_at DESC
LIMIT 10;
```

### Mean RTT per model across all offline runs

```sql
SELECT model_name, ROUND(AVG(rtt_ms)::numeric, 2) AS mean_rtt_ms,
       ROUND(PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY rtt_ms)::numeric, 2) AS p95_rtt_ms,
       COUNT(*) AS calls
FROM telemetry_model_calls mc
JOIN telemetry_sessions s USING (session_id)
WHERE s.source_type = 'offline'
GROUP BY model_name
ORDER BY mean_rtt_ms DESC;
```

### Drop rate trend over the last 24 hours

```sql
SELECT DATE_TRUNC('hour', started_at) AS hour,
       ROUND(AVG(drop_rate)::numeric, 4) AS mean_drop_rate,
       SUM(total_frames) AS total_frames
FROM telemetry_sessions
WHERE started_at > NOW() - INTERVAL '24 hours'
GROUP BY 1
ORDER BY 1;
```

### Per-student recognition latency

```sql
SELECT student_id, frames_tracked,
       ROUND(mean_confidence::numeric, 3) AS mean_conf,
       ROUND(recognition_latency_ms::numeric, 2) AS recog_ms
FROM telemetry_students
WHERE session_id = '<uuid>'
ORDER BY frames_tracked DESC;
```

---

## Integration checklist

| Integration point | Status | File |
|---|---|---|
| `TelemetrySession` API (session.py) | Done | `apps/telemetry/session.py` |
| DB models + migration (0001_initial.py) | Done | `apps/telemetry/models.py`, `migrations/0001_initial.py` |
| LPM event model + migration (0002) | Staged | `apps/telemetry/models.py`, `migrations/0002_telemetrylpmevent.py` |
| JSON dual-sink writer (writer.py) | Done | `apps/telemetry/writer.py` |
| Statistical helpers (metrics.py) | Done | `apps/telemetry/metrics.py` |
| ContextVar propagation | Done | `apps/telemetry/context.py` |
| Celery task lifecycle (auto start/end) | Done | `apps/telemetry/celery_integration.py` → `apps.py:ready()` |
| Offline video task frame recording | Done | `apps/video_analysis/tasks.py:_tel_record_frame()` in `_on_frame_complete` + `_on_triton_frame_inferred` |
| Offline LPM event recording | Staged | `apps/video_analysis/tasks.py:_tel_record_lpm_event()` when `LPM_ENABLED=1` |
| Live stream task frame recording | Done | `apps/video_analysis/tasks.py:_tel_record_frame()` in live `_on_frame_complete` |
| Triton client RTT hook | Done | `apps/pipeline/services/triton_client.py:_try_record_telemetry()` |
| Test / benchmark harness | Done | Use `TelemetrySession(source_type="test")` context manager; Celery signals also cover `ingest_runtime_event_task` |

All integration points are wired. Run `python manage.py migrate` on prod to apply `0001_initial` and `0002_telemetrylpmevent`.
Update `agents.md` Telemetry section status to **IMPLEMENTED** once the migration is applied and confirmed in production.
