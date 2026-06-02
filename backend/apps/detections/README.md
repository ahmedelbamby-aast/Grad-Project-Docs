# Detections Module

**Last updated:** 2026-03-05

## Purpose
Real-time detection frame streaming and pyramid prediction delivery for monitoring sessions.

## Public API
- Models: `DetectionFrame`, `BoundingBox`, `PyramidPrediction`
- Consumer: `DetectionConsumer` (WebSocket per session)
- Serializers: `DetectionFrameSerializer`, `BoundingBoxSerializer`
- Views: detection query endpoints

## Dependencies
- `channels` — WebSocket consumer base class
- `pipeline` app — produces detection frames via `PipelineService`

## Key Configuration
- WebSocket route: `/ws/detections/{session_id}/`

## Notes
- `DetectionConsumer` supports ping/pong keep-alive (responds to `{"type": "ping"}` with `{"type": "pong"}`).
- Channel group: `detections_{session_id}` — one group per active session.
