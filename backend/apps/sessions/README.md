# Sessions Module

## Purpose
Manages exam monitoring sessions and provides real-time comment streaming for instructor collaboration.

## Public API
- Models: `MonitoringSession`, `InstructorComment`
- Consumer: `SessionCommentsConsumer` (WebSocket per session)
- Service: `SessionService`
- ViewSet: `MonitoringSessionViewSet`, `InstructorCommentViewSet`

## Dependencies
- `channels` — WebSocket consumer base class
- `cameras` app — sessions reference camera sources

## Key Configuration
- WebSocket route: `/ws/sessions/{session_id}/comments/`

## Notes
- `SessionCommentsConsumer` supports ping/pong keep-alive (responds to `{"type": "ping"}` with `{"type": "pong"}`).
- Channel group: `comments_{session_id}` — one group per active session.
# Boundary Summary

| Field | Value |
| --- | --- |
| Purpose | Live monitoring session lifecycle |
| Responsibilities | Start monitoring sessions; stop sessions and publish state |
| Public inputs | Start commands; stop commands; session events |
| Public outputs | Session state; channel events |
| Consumers | Cameras, detections, frontend live monitoring UI |
| Dependencies | Cameras, Channels, database |
| Failure behavior | Session reports stopped or degraded state |
