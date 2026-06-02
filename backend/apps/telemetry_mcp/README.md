# Telemetry MCP Module

**Last updated:** 2026-05-23

Read-only telemetry MCP surface over normalized persisted runtime evidence.

## Purpose

- Provide governed AI/operator access to telemetry summaries and event records.
- Enforce read-only access boundaries independent from dashboard and ingestion paths.

## Exposed HTTP Interfaces

- `GET /api/v1/telemetry-mcp/resources/`
- `GET /api/v1/telemetry-mcp/resources/{resource_name}/?limit={n}&offset={n}`
- `POST /api/v1/telemetry-mcp/tools/query/`

`tools/query` only permits `action = "query_resource"` and denies mutating or arbitrary actions.

## Resource Contract

Current read-only resources:

- `session_summaries`
- `frame_events`
- `model_calls`

Response envelope:

- `resource`
- `read_only` (always `true`)
- `total`
- `limit`
- `offset`
- `results[]`

## Security Contract

- Authentication required.
- RBAC required: `teacher`/`admin` roles (staff/superuser allowed).
- Bounds enforced with capped pagination:
  - default `limit=50`
  - max `limit=200`
  - `offset>=0`
- Payload redaction applies recursively for sensitive keys (for example name, email, token, password, cookie, IP).

## Audit Contract

- Every access, denial, and failure is audited through telemetry MCP audit events.
- Audited outcomes include:
  - success (`log_mcp_access`)
  - authorization/prohibited-request denial (`log_mcp_denied`)
  - backend/unexpected failure (`log_mcp_failure`)

## Failure Semantics

- Unknown resource -> validation error.
- Prohibited tool action -> permission denied.
- Backend database faults -> `503` temporary-unavailable response.
- MCP failures must not mutate run state or block dashboard telemetry paths.
