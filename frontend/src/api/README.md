# API Module

## Purpose
Centralised HTTP client and per-resource API functions for communicating with the backend REST API. All modules share a single Axios instance configured in `client.ts`.

## Key Files
- `client.ts` — Axios instance (base URL, CSRF, error normalisation)
- `auth.ts` — Authentication endpoints (login, logout, me)
- `cameras.ts` — Camera CRUD, connect/disconnect, events
- `sessions.ts` — Monitoring session endpoints
- `detections.ts` — Detection frame queries
- `anomalies.ts` — Anomaly queries and triage
- `dashboard.ts` — Dashboard aggregation
- `recordings.ts` — Recording management
- `health.ts` — System health checks
- `exports.ts` — Data export endpoints
- `runtime.ts` — Runtime dashboard and telemetry projection endpoints

## Key Configuration
- `VITE_API_BASE_URL` — Override base URL (default: `/api/v1`, relative — goes through Vite proxy or nginx)

## Notes
- `client.ts` uses a relative `baseURL` (`/api/v1`) so all requests go through the same-origin proxy, avoiding CORS entirely.
- CSRF token is auto-attached via request interceptor from the `csrftoken` cookie.
- All HTTP errors are normalised to a uniform `ApiError` shape by `toApiError()`.

## Runtime And Telemetry Contracts

`runtime.ts` consumes normalized backend runtime/telemetry projections only:

- `/video-analysis/runtime/sessions/`
- `/video-analysis/runtime/frame-events/`
- `/video-analysis/runtime/model-events/`
- `/video-analysis/runtime/summary/`
- `/video-analysis/runtime/issues/`
- `/video-analysis/runtime/filters/`
- `/video-analysis/runtime/timeline/`
- `/video-analysis/runtime/model-matrix/`
- `/video-analysis/runtime/student-timeline/`
- `/video-analysis/runtime/profiles/`

Job-scoped telemetry APIs remain under `/video-analysis/jobs/{job_id}/telemetry/*` and are expected to be PII-safe, bounded, and role-gated by backend policy.

## Maturity Closure Notes
- Runtime mode and evidence contracts are enforced for wave-driven closure.

