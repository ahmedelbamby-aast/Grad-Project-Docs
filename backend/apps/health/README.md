# Health Module

## Purpose
Provides system health monitoring and broadcasts infrastructure status (service health, storage warnings) to connected dashboard clients in real time.

## Public API
- Consumer: `HealthConsumer` (WebSocket, shared group)
- Service: `HealthService` (health status aggregation)
- Views: health check REST endpoints

## Dependencies
- `channels` — WebSocket consumer base class
- Celery beat — periodic health-check tasks push updates

## Key Configuration
- WebSocket route: `/ws/health/`

## Notes
- `HealthConsumer` supports ping/pong keep-alive (responds to `{"type": "ping"}` with `{"type": "pong"}`).
- Channel group: `health_dashboard` — single shared group for all dashboard clients.
- Events: `health.update`, `storage.warning`.
# Boundary Summary

| Field | Value |
| --- | --- |
| Purpose | Service, model, stream, and dependency health |
| Responsibilities | Report service health; expose degraded dependency states |
| Public inputs | Health probes; dependency checks |
| Public outputs | Health reports; degraded states |
| Consumers | Frontend health/settings UI, production deployment, operators |
| Dependencies | Pipeline health, Redis, database, Triton health |
| Failure behavior | Read-only health degradation report |
