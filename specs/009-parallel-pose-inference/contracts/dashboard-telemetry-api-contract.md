# Contract: Dashboard Telemetry REST And WebSocket APIs

**Feature**: [Parallel Pose Inference with Hybrid Telemetry](../spec.md)  
**Data Model**: [data-model.md](../data-model.md)  
**Capture Contract**: [telemetry-capture-contract.md](telemetry-capture-contract.md)  
**MCP Contract**: [mcp-telemetry-server-contract.md](mcp-telemetry-server-contract.md)

## Purpose

Application dashboards consume authenticated backend REST/WebSocket projections over normalized persisted telemetry. They never read Nsight, Triton, PresentMon, OpenVINO, or MCP directly.

## Required HTTP Interfaces

All routes require an authenticated user with a permitted instructor, admin, or release-validation role and a permitted relationship to the job.

| Endpoint | Purpose | Required Projection |
|----------|---------|---------------------|
| `GET /api/v1/video-analysis/jobs/{job_id}/telemetry/summary/` | KPI and validation verdict overview | Run/source states, key KPIs, stage timing, evidence readiness, redaction state |
| `GET /api/v1/video-analysis/jobs/{job_id}/telemetry/timeline/` | Chart series | Bounded/downsampled time series filtered by source, accelerator, model, stage, and interval |
| `GET /api/v1/video-analysis/jobs/{job_id}/telemetry/artifacts/` | Evidence availability | Manifest metadata, hashes, authorization-controlled download capability; never raw trace bytes inline |
| `GET /api/v1/video-analysis/jobs/{job_id}/runtime-assignments/` | Execution proof | Model/runtime/effective device/worker/fallback proof |
| `GET /api/v1/video-analysis/jobs/{job_id}/benchmark-comparison/` | Performance verdict | Baseline/candidate parity, deltas, thresholds, telemetry mode, instrumentation overhead |

## Summary Response

```json
{
  "job_id": "uuid",
  "run_id": "uuid",
  "validation_gate_status": "passed",
  "telemetry": {
    "nvidia_nsight_systems": "available",
    "triton_prometheus": "available",
    "intel_presentmon": "available",
    "openvino_profile": "available"
  },
  "kpis": {
    "throughput_fps": 0.0,
    "p95_processing_latency_ms": 0.0
  },
  "evidence_ready": true,
  "pii_safe": true,
  "generated_at": "ISO-8601"
}
```

## Timeline Query Bounds

| Control | Required Behavior |
|---------|-------------------|
| Time window | Require `from` and `to`, or apply a documented finite server default; reject excessive ranges. |
| Resolution | Support server-selected aggregation/downsampling for requested intervals. |
| Result size | Enforce page/sample caps with opaque cursor continuation where required. |
| Fields | Return normalized metrics and safe dimensions only; no student-identifying content. |
| Invalid input | Respond with validation error without leaking collector paths or internals. |

## WebSocket Interface

| Interface | Purpose |
|-----------|---------|
| `WS /video-analysis/jobs/{job_id}/telemetry/` | Emit authorized normalized telemetry updates for live/delayed status and offline job progress. |

WebSocket events use the same role checks, redaction policy, and bounded projection schema as REST. An event includes `job_id`, `run_id`, `event_type`, `sequence`, `generated_at`, and a PII-safe normalized payload. Connection loss or MCP availability does not alter stored processing state.

## Authorization And Privacy

- Apply the existing application authentication and job authorization boundary before querying telemetry.
- Redact or omit student-identifying fields before response serialization or WebSocket emission.
- Expose raw trace downloads only through an explicit authorized artifact download path for admin/release-validation roles.
- Record denied evidence access through the application's audit facility.

## Independence From MCP

Dashboard endpoints query normalized backend persistence directly. Required integration tests run with the MCP service disabled and with simulated MCP failure; responses and WebSocket updates must remain available if telemetry evidence is valid.

## Failure Responses

| Condition | Required API Behavior |
|-----------|-----------------------|
| Unknown or inaccessible job | Return not-found/forbidden without existence leakage according to application policy. |
| Evidence still ingesting | Return a stable pending/partial state and available completed projections. |
| Mandatory evidence invalid | Return failed/degraded verdict with actionable safe reason codes. |
| MCP unavailable | No dashboard error; MCP is not on the request path. |

## Contract Tests

- Role and job-bound access is enforced for every endpoint and WebSocket subscription.
- Summary/timeline/artifact/assignment/comparison payloads are generated from normalized persisted evidence.
- Timeline range/sample limits and artifact metadata-only responses are enforced.
- PII fields and unrestricted raw trace contents are absent from standard responses/events.
- Dashboard delivery passes while MCP is disabled, disconnected, or rejecting clients.

