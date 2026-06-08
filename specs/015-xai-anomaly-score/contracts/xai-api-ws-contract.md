# Contract: XAI REST And WebSocket Surface

**Contract ID**: `xai.api_ws.v1`
**Access**: Authenticated, job-scoped authorization, access audit for sensitive
artifacts and reviewer feedback.

## REST Endpoints

| Method/path | Purpose | Response |
|---|---|---|
| `GET /api/v1/video-analysis/jobs/{job}/xai/signals/` | Signal definitions and availability | paginated definitions/status |
| `GET /api/v1/video-analysis/jobs/{job}/xai/evidence/` | Filtered evidence envelopes | paginated bounded summaries |
| `GET /api/v1/video-analysis/jobs/{job}/xai/scores/` | Review-priority records | paginated scores/contributions |
| `GET /api/v1/video-analysis/jobs/{job}/xai/explanations/{id}/` | Composed explanation | explanation contract |
| `POST /api/v1/video-analysis/jobs/{job}/xai/deep-requests/` | Request eligible deep XAI | `202` request state |
| `GET /api/v1/video-analysis/jobs/{job}/xai/deep-requests/{id}/` | Deep request status/artifact | lifecycle/status |
| `GET /api/v1/video-analysis/jobs/{job}/xai/artifacts/{id}/` | Authorized artifact access | streamed artifact |
| `POST /api/v1/video-analysis/jobs/{job}/xai/reviews/` | Governed feedback | immutable feedback record |
| `GET /api/v1/video-analysis/jobs/{job}/xai/renderer-config/` | Renderer/data contract | bounded WebGL config |

## Query Rules

- Pagination and source-time range are mandatory for evidence/scores.
- Track/signal/model/truth-state filters are optional and indexed.
- Unbounded full-job arrays are never returned in one JSON response.
- Large series/matrices use chunk/tile or binary endpoints.
- Every response includes schema version and route snapshot reference.

## WebSocket Events

```json
{
  "schema_version": "xai.ws.event.v1",
  "event_id": "uuid",
  "event_type": "xai.score.appended",
  "job_id": "uuid",
  "source_time_ms": 10000,
  "sequence": 42,
  "route_snapshot_ref": "uuid",
  "payload": {},
  "truth_state": "valid"
}
```

Supported event types:

- `xai.evidence.appended`;
- `xai.score.appended`;
- `xai.explanation.updated`;
- `xai.deep_request.state_changed`;
- `xai.artifact.available`;
- `xai.reconciliation.changed`;
- `xai.renderer.data_invalidated`.

## Incremental Delivery

- Sequence is monotonic per job/channel.
- Client detects gaps and fetches a bounded REST reconciliation range.
- Out-of-order/duplicate events are idempotently ignored.
- Queue pressure may drop stale renderer updates, not authoritative evidence.
- Binary chunks include schema/version/dtype/shape/endianness/digest metadata.

## Errors

| Code | Meaning |
|---|---|
| `XAI_ROUTE_SNAPSHOT_INVALID` | Runtime truth is unavailable/incompatible |
| `XAI_SIGNAL_UNREGISTERED` | Active source lacks a governed signal definition |
| `XAI_SCORE_WITHHELD` | Score exists as withheld with reason |
| `XAI_DEEP_METHOD_INELIGIBLE` | Method unsupported for route/input |
| `XAI_DEEP_QUEUE_SATURATED` | Bounded queue rejected request |
| `XAI_ARTIFACT_UNAVAILABLE` | Artifact absent/expired/failed |
| `XAI_ACCESS_DENIED` | Job/artifact/review authorization failed |
| `XAI_SCHEMA_INCOMPATIBLE` | Client/server contract mismatch |
