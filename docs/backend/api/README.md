# API Reference

## Purpose

Provides the upload, status, frame-fetch, visibility, and WebSocket endpoints used by the video analysis feature.

## Endpoints

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/video-analysis/jobs/` | `POST` | Upload a video and create a job. |
| `/api/v1/video-analysis/jobs/{job_id}/status/` | `GET` | Poll job status and progress. |
| `/api/v1/video-analysis/jobs/{job_id}/frames/{frame_id}/` | `GET` | Retrieve a frame and its visible overlays. |
| `/api/v1/video-analysis/jobs/{job_id}/visibility/` | `PATCH` | Toggle model overlay visibility. |
| `/api/v1/video-analysis/jobs/{job_id}/retry/` | `POST` | Requeue failed/partial jobs from the same analytics surface. |
| `/api/v1/videos/{id}/results` | `GET` | Return synchronized playback metadata, detections, `pipeline_mode`, and optional `annotated_video_url`. |
| `/ws/video-analysis/jobs/{job_id}/` | WebSocket | Subscribe to real-time job updates. |
| `/ws/video-analysis/live/{session_id}/` | WebSocket | Stream live RTSP overlay packets for camera workspaces. |

## Response Notes

- Status responses include `completed_partial` and `component_failures` when processing is only partially successful.
- Frame responses return already-filtered bounding boxes based on the cached visibility state.
- Visibility updates are persisted in Redis and applied to subsequent frame retrievals.

## Error Codes

| Code | Meaning |
|---|---|
| `JOB_NOT_FOUND` | The job ID does not exist. |
| `ACCESS_DENIED` | The authenticated user cannot access the job. |
| `VALIDATION_FAILED` | The request body or upload failed validation. |
| `JOB_NOT_READY` | Visibility was requested before the job was ready. |

## Related Documents

- [Video Analysis App](../apps/video_analysis/README.md)
- [Tracking App](../apps/tracking/README.md)
- [Frontend Components](../../frontend/src/components/README.md)
