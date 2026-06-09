# Contract: XAI REST And WebSocket Surface

**Contract ID**: `xai.api_ws.v1`
**Access**: Authenticated. Job-scoped authorization and access audit are
required for sensitive artifacts and reviewer feedback. The
review-priority aggregate preview endpoint is a non-persistent governed preview
surface under `/api/v1/anomalies/`; it requires a real `job_id`, is limited to
teacher/admin access, is authenticated and audited, but it does not persist a
score record.

## REST Endpoints

| Method/path | Purpose | Response |
|---|---|---|
| `GET /api/v1/video-analysis/jobs/{job}/xai/signals/` | Signal definitions and availability | paginated definitions/status |
| `GET /api/v1/video-analysis/jobs/{job}/xai/evidence/` | Filtered evidence envelopes | paginated bounded summaries |
| `GET /api/v1/video-analysis/jobs/{job}/xai/pattern-profiles/` | Compatible observed-pattern profiles and knowledge limits | paginated bounded summaries |
| `GET /api/v1/video-analysis/jobs/{job}/xai/pattern-windows/` | Per-student signal-pattern windows | chunked/paginated bounded summaries |
| `GET /api/v1/video-analysis/jobs/{job}/xai/scores/` | Review-priority records | paginated scores/contributions |
| `GET /api/v1/video-analysis/jobs/{job}/xai/explanations/{id}/` | Composed explanation | explanation contract |
| `POST /api/v1/video-analysis/jobs/{job}/xai/deep-requests/` | Request eligible deep XAI | `202` request state |
| `GET /api/v1/video-analysis/jobs/{job}/xai/deep-requests/{id}/` | Deep request status/artifact | lifecycle/status |
| `GET /api/v1/video-analysis/jobs/{job}/xai/artifacts/{id}/` | Authorized artifact access | streamed artifact |
| `POST /api/v1/video-analysis/jobs/{job}/xai/reviews/` | Governed non-ground-truth reviewer assessment | immutable assessment record |
| `GET /api/v1/video-analysis/jobs/{job}/xai/renderer-config/` | Renderer/data contract | bounded WebGL config |
| `GET /api/v1/video-analysis/jobs/{job}/xai/interaction-graph/` | Bounded deterministic student-interaction graph frames (identity-gated nodes/edges and per-student graph features) | chunked/paginated bounded graph frames |
| `GET /api/v1/video-analysis/jobs/{job}/xai/baselines/` | Compatible general/context baseline snapshots and tiers (assumed-normal, never ground truth) | paginated bounded baseline summaries |
| `GET /api/v1/video-analysis/jobs/{job}/xai/parameters/` | Parameter provenance (`learned`/`configured`) for the values used | paginated provenance records |
| `GET /api/v1/video-analysis/jobs/{job}/xai/model-promotions/` | Model promotion records and current `promotion_status` (probe -> mandatory) | paginated promotion records |
| `POST /api/v1/anomalies/review-priority/aggregate-preview/` | Preview bounded student-local scores plus session/classroom aggregate using governed review-label semantics | bounded session summary only; no persistence |

## Query Rules

- Pagination and source-time range are mandatory for evidence/scores.
- Track/signal/model/truth-state filters are optional and indexed.
- Unbounded full-job arrays are never returned in one JSON response.
- Large series/matrices use chunk/tile or binary endpoints.
- Every persisted/runtime-bound response includes schema version and route
  snapshot reference. The non-persistent aggregate-preview response includes
  `schema_version` and `route_snapshot_ref=null`.
- Pattern/score responses include `ground_truth_status`,
  `pattern_state`, profile/window references, and knowledge limits.
- The aggregate-preview endpoint is authenticated, accepts only bounded
  student summaries, uses a governed session aggregation policy, and applies
  only governed confirmed-observation review labels to the separate
  session/classroom lift term. Caller-supplied
  `approved_deviation_weight` overrides are rejected.
- The aggregate-preview request must include a valid `job_id` scope reference.
- Interaction-graph responses are bounded by configured node/edge caps, carry
  per-edge `identity_confidence` and `truth_state`, mark ambiguous-identity edges
  `unresolved`, and never imply collusion or cheating. Large adjacency matrices
  use the binary/tile path; learned-graph (`PROBE_ONLY`) outputs are never
  returned as a production score or `pattern_state`.
- Score responses expose `deviation_vs_self` and `deviation_vs_population` plus
  `parameter_provenance_refs`; baseline responses are marked assumed-normal and
  never ground truth; parameter responses carry a `learned`/`configured`
  provenance with the baseline reference or fingerprinted `.env`/config key.
- Model-promotion responses expose `promotion_status` and the evidence packet
  (benchmark, serving metrics, model card, approver, rollback) for each
  transition; only `MANDATORY` models contribute production signals, and no
  promotion field reports behavioral accuracy/AUROC.

## Aggregate Preview Schema

Request:

```json
{
  "job_id": "uuid",
  "students": [
    {
      "student_ref": "student-a",
      "review_priority_score": 25.0,
      "pattern_state": "within_observed_pattern",
      "truth_state": "valid",
      "review_label": "confirmed-observation"
    }
  ]
}
```

Response:

```json
{
  "schema_version": "anomaly.review_priority.preview.v1",
  "route_snapshot_ref": null,
  "student_count": 2,
  "approved_deviation_count": 1,
  "base_mean_score": 17.5,
  "approved_deviation_boost": 15.0,
  "session_review_priority_score": 32.5,
  "students": [
    {
      "student_ref": "student-a",
      "review_priority_score": 25.0,
      "pattern_state": "pattern_deviation",
      "review_approved_deviation": true,
      "truth_state": "valid"
    },
    {
      "student_ref": "student-b",
      "review_priority_score": 10.0,
      "pattern_state": "within_observed_pattern",
      "review_approved_deviation": false,
      "truth_state": "valid"
    }
  ]
}
```

Rules:

- The caller must already hold job-scoped access for `job_id`; elevated role
  membership alone is insufficient.
- Student-local scores are echoed unchanged; the endpoint computes the
  session/classroom lift as a separate aggregate term only.
- Caller-supplied `approved_deviation_weight` overrides are rejected.
- `pattern_deviation` input is accepted only when the student's own valid
  evidence supports that state or when a governed approved review label
  supplies the deviation trigger.
- Non-valid truth states cannot yield `within_observed_pattern` or
  `pattern_deviation` in the preview output. `unknown` and `unavailable`
  normalize to `insufficient_context`; `degraded`, `suppressed`, and
  `invalidated` normalize to `withheld`.
- The endpoint is a preview surface only: no score, contribution, or profile
  record is persisted.

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
- `xai.pattern_profile.changed`;
- `xai.pattern_window.appended`;
- `xai.explanation.updated`;
- `xai.deep_request.state_changed`;
- `xai.artifact.available`;
- `xai.reconciliation.changed`;
- `xai.interaction_graph.appended`;
- `xai.baseline.changed`;
- `xai.model_promotion.changed`;
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
| `XAI_PATTERN_CONTEXT_INSUFFICIENT` | No compatible valid pattern comparison exists |
| `XAI_PATTERN_PROFILE_QUARANTINED` | Profile cannot support scoring |
| `XAI_DEEP_METHOD_INELIGIBLE` | Method unsupported for route/input |
| `XAI_DEEP_QUEUE_SATURATED` | Bounded queue rejected request |
| `XAI_ARTIFACT_UNAVAILABLE` | Artifact absent/expired/failed |
| `XAI_ACCESS_DENIED` | Job/artifact/review authorization failed |
| `XAI_SCHEMA_INCOMPATIBLE` | Client/server contract mismatch |
