# Cameras Module

## Purpose
- Manages camera sources, connection lifecycle, and live-stream integration via pluggable stream providers.

## Public API
- Models: `CameraSource`, `StreamRegistration`, `ConnectionEvent`
- Service: `CameraService`, `StreamGateway`, `RtspProber`
- ViewSet: `CameraSourceViewSet` (connect/disconnect/events)
- Tasks: `stream_health_check`, `cleanup_connection_events`

## Key Configuration
- `FIELD_ENCRYPTION_KEY`
- `STREAM_PROVIDER` (`legacy_go2rtc` or `gst_mediamtx`)
- `STREAM_PIPELINE_PROFILE`
- `STREAM_RESTART_BUDGET`
- `STREAM_RESTART_COOLDOWN_SECONDS`
- `GO2RTC_API_URL`, `GO2RTC_WHEP_URL`
- `MEDIAMTX_API_URL`, `MEDIAMTX_WHEP_URL`
- `GSTREAMER_PROBE_URL`

## Notes
- All user-facing camera status transitions are broadcast via Channels.
- Connection events are retained for 90 days (cleanup task).
- `CameraStatusConsumer` supports ping/pong keep-alive (responds to `{"type": "ping"}` with `{"type": "pong"}`).
# Boundary Summary

| Field | Value |
| --- | --- |
| Purpose | Camera inventory and stream source configuration |
| Responsibilities | Manage camera records; publish stream descriptors |
| Public inputs | Camera CRUD; RTSP and WHEP metadata |
| Public outputs | Camera records; stream descriptors |
| Consumers | Sessions, live runtime, frontend live monitoring UI |
| Dependencies | Accounts, sessions, stream-provider configuration |
| Failure behavior | Invalid streams are reported as degraded camera state |
