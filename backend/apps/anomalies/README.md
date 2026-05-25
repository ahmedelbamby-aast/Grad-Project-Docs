# Anomalies Module

## Purpose
Detects, records, and broadcasts anomaly events during exam monitoring sessions. Provides both session-scoped and admin-global WebSocket feeds.

## Public API
- Models: `Anomaly`, `AnomalyEvent`
- Consumers: `AnomalyConsumer` (per-session), `AdminAnomalyFeedConsumer` (admin global)
- Service: `AnomalyService` (triage, status changes, broadcast)
- Serializers: `AnomalySerializer`, `AnomalyEventSerializer`
- Views: anomaly query and triage endpoints

## Dependencies
- `channels` — WebSocket consumer base class
- `sessions` app — anomalies are scoped to monitoring sessions

## Key Configuration
- Session WebSocket route: `/ws/anomalies/{session_id}/`
- Admin WebSocket route: `/ws/admin/anomalies/`

## Notes
- Both consumers support ping/pong keep-alive (respond to `{"type": "ping"}` with `{"type": "pong"}`).
- Session group: `anomalies_{session_id}` — per active session.
- Admin group: `admin_anomaly_feed` — shared across all admin clients.
# Boundary Summary

| Field | Value |
| --- | --- |
| Purpose | Anomaly triage, status, notes, and severity |
| Responsibilities | Manage alerts; publish triage records |
| Public inputs | Detection events; tracking events; triage commands |
| Public outputs | Alerts; triage records |
| Consumers | Dashboard runtime, exports, audit |
| Dependencies | Detections, sessions, accounts |
| Failure behavior | Alerts are marked degraded or rejected with validation errors |
