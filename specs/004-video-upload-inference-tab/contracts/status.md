# API Contract: Job Status & Progress

**Feature**: 004-video-upload-inference-tab  
**Date**: 2026-04-30  
**Endpoints**: 
- GET `/api/v1/video-analysis/jobs/{job_id}/status/` (polling)
- WebSocket: `/ws/video-analysis/jobs/{job_id}/` (real-time)
- WebSocket: `/ws/camera/{camera_id}` (real-time RTSP detection packets)

## Polling Endpoint

### Request

**Method**: GET  
**URL**: `/api/v1/video-analysis/jobs/{job_id}/status/`  
**Authentication**: Required (Bearer token or session cookie)

### Example Request

```bash
curl -X GET http://localhost:8000/api/v1/video-analysis/jobs/550e8400-e29b-41d4-a716-446655440000/status/ \
  -H "Authorization: Bearer YOUR_TOKEN"
```

### Response (HTTP 200 OK)

```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "processing",
  "progress_percent": 45.5,
  "processed_frames": 8190,
  "total_frames": 18000,
  "current_stage": "detection_and_tracking",
  "elapsed_seconds": 120,
  "estimated_remaining_seconds": 143,
  "message": "Processing frame 273 (10 fps)",
  "last_update": "2026-04-30T10:32:00Z",
  "error": null
}
```

### Field Descriptions

| Field | Type | Description |
|-------|------|-------------|
| `job_id` | UUID | Job identifier |
| `status` | String | One of: `queued`, `processing`, `embedding`, `completed`, `completed_partial`, `failed` |
| `progress_percent` | Float | 0-100 completion percentage |
| `processed_frames` | Integer | Frames processed so far |
| `total_frames` | Integer | Total frames in video |
| `current_stage` | String | Current processing stage (see below) |
| `elapsed_seconds` | Integer | Seconds elapsed since processing started |
| `estimated_remaining_seconds` | Integer | Estimated seconds until completion |
| `message` | String | Human-readable status message |
| `last_update` | ISO 8601 | Timestamp of last status update |
| `error` | Object or Null | Error details if status is `failed` (see below) |

### Processing Stages

| Stage | Description |
|-------|-------------|
| `queued` | Job waiting to start |
| `detection_and_tracking` | YOLO detection + ByteTrack tracking (BoT-SORT when ReID is enabled) |
| `embedding` | Embedding generation and re-ID |
| `completed` | All processing complete |
| `completed_partial` | Processing completed with some stage failures, but usable outputs are available |
| `failed` | Processing encountered an error |

### Polling Interval

**Recommended**: 2-5 seconds for real-time UI updates.

---

## WebSocket Connection

### Connection Request

**URL**: `ws://localhost:8000/ws/video-analysis/jobs/{job_id}/`  
**Authentication**: Required (Bearer token passed as URL query param or via headers)

### Example WebSocket Connection (JavaScript)

```javascript
const jobId = "550e8400-e29b-41d4-a716-446655440000";
const token = "YOUR_BEARER_TOKEN";
const ws = new WebSocket(
  `ws://localhost:8000/ws/video-analysis/jobs/${jobId}/?token=${token}`
);

ws.onopen = () => {
  console.log("Connected to job status stream");
};

ws.onmessage = (event) => {
  const message = JSON.parse(event.data);
  console.log("Status update:", message);
};

ws.onerror = (error) => {
  console.error("WebSocket error:", error);
};

ws.onclose = () => {
  console.log("Disconnected from status stream");
};
```

### WebSocket Messages

The server broadcasts messages to all connected clients as status changes.

#### Message Type: `status_update`

Sent whenever job status changes or progress is made.

```json
{
  "type": "status_update",
  "timestamp": "2026-04-30T10:32:00Z",
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "processing",
  "progress_percent": 45.5,
  "processed_frames": 8190,
  "total_frames": 18000,
  "current_stage": "detection_and_tracking",
  "message": "Processing frame 273 (10 fps)"
}
```

#### Message Type: `completion`

Sent when job processing completes successfully.

```json
{
  "type": "completion",
  "timestamp": "2026-04-30T10:35:00Z",
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "completed",
  "progress_percent": 100,
  "processed_frames": 18000,
  "total_frames": 18000,
  "total_students_tracked": 28,
  "message": "Processing complete. Ready for playback."
}
```

#### Message Type: `partial_completion`

Sent when job processing finishes with partial results available.

```json
{
  "type": "partial_completion",
  "timestamp": "2026-04-30T10:35:00Z",
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "completed_partial",
  "progress_percent": 100,
  "processed_frames": 17400,
  "total_frames": 18000,
  "component_failures": {
    "embedding": "timeout after 5 minutes",
    "reid": null,
    "rendering": null
  },
  "message": "Processing completed with partial results."
}
```

#### Message Type: `error`

Sent when an error occurs during processing.

```json
{
  "type": "error",
  "timestamp": "2026-04-30T10:34:30Z",
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "failed",
  "error_code": "ENCODING_ERROR",
  "error_message": "Video file contains unsupported codec",
  "details": {
    "codec": "VP9",
    "supported_codecs": ["h264", "h265"]
  }
}
```

#### Message Type: `keep_alive`

Sent every 30 seconds to keep connection alive (no status change).

```json
{
  "type": "keep_alive",
  "timestamp": "2026-04-30T10:32:30Z"
}
```

---

## RTSP Live Detection Stream

Live camera feeds publish detection packets on `/ws/camera/{camera_id}` using the shared detection packet contract.

### Connection Request

**URL**: `ws://localhost:8000/ws/camera/{camera_id}`
**Authentication**: Required (Bearer token passed as URL query param or via headers)

### Example Live Packet

```json
{
  "frame_index": 3721,
  "timestamp_ms": 123456,
  "camera_id": "cam_1",
  "detections": [
    {
      "id": "track_7",
      "parent_id": null,
      "source_model": "person_detection",
      "label": "student",
      "confidence": 0.97,
      "bbox": {
        "x1": 100,
        "y1": 200,
        "x2": 180,
        "y2": 360
      }
    }
  ]
}
```

### Live Stream Rules

- `timestamp_ms` is required for overlay synchronization.
- `frame_index` should increase monotonically for the stream session.
- `id` must remain stable across frames while the subject remains tracked.
- Clients should clear stale detections when no new packet arrives within the configured timeout.

---

## Error Responses

### HTTP 404 Not Found

**Scenario**: Job does not exist

```json
{
  "error": "JOB_NOT_FOUND",
  "message": "Video analysis job not found",
  "details": {
    "job_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

### HTTP 403 Forbidden

**Scenario**: User does not have permission to access this job

```json
{
  "error": "ACCESS_DENIED",
  "message": "You do not have permission to access this job",
  "details": {
    "job_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

### WebSocket: Connection Rejected

**Scenario**: Invalid token or authentication fails

```json
{
  "type": "error",
  "error_code": "AUTHENTICATION_FAILED",
  "message": "Invalid or expired token"
}
```

---

## Failed Job Error Details

When `status` is `failed`, the `error` field contains:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {
      "stage": "detection_and_tracking",
      "frame_number": 1234,
      "log_reference": "logs/job-550e8400.log"
    }
  }
}
```

When `status` is `completed_partial`, the response should also include `component_failures` so the UI can show which pipeline stages succeeded and which need operator attention.

### Common Error Codes

| Code | Message | Resolution |
|------|---------|-----------|
| `INVALID_VIDEO_FILE` | Video file corrupted | Re-upload video |
| `ENCODING_ERROR` | Unsupported codec | Convert video to H.264 or H.265 |
| `INSUFFICIENT_DISK_SPACE` | Server disk full | Contact administrator |
| `TIMEOUT` | Processing exceeded timeout | Large video; try again or split video |
| `OUT_OF_MEMORY` | Insufficient system memory | Contact administrator |
| `INTERNAL_ERROR` | Unexpected server error | Retry or contact support |

---

## Retry Strategy

**For Polling Clients**:
- Start with 2-second interval.
- If status hasn't changed after 10 polls, increase to 5-second interval.
- Maximum interval: 30 seconds.

**For WebSocket Clients**:
- If connection drops, wait 2 seconds before attempting reconnection.
- Exponential backoff: 2s, 4s, 8s (max 30s).
- Max 5 reconnection attempts; then fall back to polling.

---

## Cross-References

- **[data-model.md](../data-model.md)**: VideoAnalysisJob status values and state machine.
- **[contracts/upload.md](upload.md)**: Job creation and initial status.
- **[contracts/overlay.md](overlay.md)**: Toggling model visibility (independent of status).
- **[quickstart.md](../quickstart.md)**: Example polling and WebSocket usage.
