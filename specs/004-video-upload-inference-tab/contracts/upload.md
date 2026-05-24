# API Contract: Video Upload & Playback Results

**Feature**: 004-video-upload-inference-tab  
**Date**: 2026-04-30  
**Endpoint**: `POST /api/v1/video-analysis/jobs/`  

## Request

**Method**: POST  
**Content-Type**: `multipart/form-data`  
**Authentication**: Required (Bearer token or session cookie)

### Form Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `video_file` | File (.mp4) | Yes | Video file up to 2GB |
| `title` | String | No | Human-readable job title (default: filename) |
| `description` | String | No | Optional job description |

### Example Request (using configured YOLO model)

```bash
curl -X POST http://localhost:8000/api/v1/video-analysis/jobs/ \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -F "video_file=@classroom-session.mp4" \
  -F "title=Classroom Session - April 30" \
  -F "description=Morning class observation"
```

## Response

### Success (HTTP 201 Created)

```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "queued",
  "filename": "classroom-session.mp4",
  "title": "Classroom Session - April 30",
  "description": "Morning class observation",
  "created_at": "2026-04-30T10:30:00Z",
  "total_frames": 18000,
  "processed_frames": 0,
  "progress_percent": 0,
  "file_size_bytes": 524288000,
  "duration_seconds": 600,
  "fps": 30,
  "_links": {
    "self": "/api/v1/video-analysis/jobs/550e8400-e29b-41d4-a716-446655440000/",
    "status": "/api/v1/video-analysis/jobs/550e8400-e29b-41d4-a716-446655440000/status/",
    "frames": "/api/v1/video-analysis/jobs/550e8400-e29b-41d4-a716-446655440000/frames/",
    "tracks": "/api/v1/video-analysis/jobs/550e8400-e29b-41d4-a716-446655440000/tracks/",
    "results": "/api/v1/videos/550e8400-e29b-41d4-a716-446655440000/results"
  }
}
```

### Field Descriptions

| Field | Type | Description |
|-------|------|-------------|
| `job_id` | UUID | Unique identifier for this job |
| `status` | String | Current status: `queued` |
| `filename` | String | Original uploaded filename |
| `title` | String | Job title (if provided) |
| `description` | String | Job description (if provided) |
| `created_at` | ISO 8601 | Timestamp of job creation |
| `total_frames` | Integer | Total frames in video |
| `processed_frames` | Integer | Frames processed (starts at 0) |
| `progress_percent` | Float | 0-100 completion percentage |
| `file_size_bytes` | Integer | Size of uploaded file in bytes |
| `duration_seconds` | Float | Video duration in seconds |
| `fps` | Float | Frames per second |
| `_links` | Object | HATEOAS links to related resources |

---

## Error Responses

### HTTP 400 Bad Request

**Scenario**: File not provided or validation failed

```json
{
  "error": "VALIDATION_FAILED",
  "message": "Video file is required",
  "details": {
    "video_file": ["This field is required."]
  }
}
```

### HTTP 413 Payload Too Large

**Scenario**: File exceeds 2GB limit

```json
{
  "error": "FILE_TOO_LARGE",
  "message": "File size exceeds 2GB maximum",
  "details": {
    "max_size_bytes": 2147483648,
    "uploaded_size_bytes": 3000000000
  }
}
```

### HTTP 415 Unsupported Media Type

**Scenario**: File format not supported

```json
{
  "error": "UNSUPPORTED_FORMAT",
  "message": "Only .mp4 files are supported",
  "details": {
    "provided_format": ".avi",
    "supported_formats": [".mp4"]
  }
}
```

### HTTP 422 Unprocessable Entity

**Scenario**: File is corrupted or unreadable

```json
{
  "error": "INVALID_VIDEO_FILE",
  "message": "Video file is corrupted or not a valid video",
  "details": {
    "reason": "ffprobe could not read video stream"
  }
}
```

### HTTP 429 Too Many Requests

**Scenario**: User has too many concurrent uploads

```json
{
  "error": "RATE_LIMIT_EXCEEDED",
  "message": "Maximum 5 concurrent uploads per user",
  "details": {
    "limit": 5,
    "current": 5,
    "retry_after_seconds": 60
  }
}
```

### HTTP 500 Internal Server Error

**Scenario**: Server error during processing

```json
{
  "error": "INTERNAL_ERROR",
  "message": "An error occurred while processing the upload",
  "details": {
    "request_id": "550e8400-e29b-41d4-a716-446655440001",
    "support_contact": "support@example.com"
  }
}
```

---

## Validation Rules

### File Size
- **Minimum**: 100 KB
- **Maximum**: 2 GB
- **Rationale**: Prevents trivial uploads; 2GB supports ~1 hour of 4K video at typical bitrate.

### File Format
- **Accepted**: `.mp4` (H.264 or H.265 codec)
- **Rejected**: `.avi`, `.mov`, `.mkv`, `.flv` (other formats may be added later)
- **Rationale**: .mp4 is widely supported and ensures consistent processing pipeline.

### Metadata Extraction
- **FPS**: Extracted from video file; used for timestamp calculation.
- **Total Frames**: Calculated as `duration_seconds * fps`.
- **Duration**: Extracted using ffprobe; must be >= 1 second.

### Title & Description
- **Length Limits**: 
  - Title: 1-255 characters
  - Description: 0-1000 characters
- **Allowed Characters**: Alphanumeric, spaces, hyphens, underscores, periods.

---

## Async Processing

- Upload request returns immediately with job in `queued` status.
- Processing begins asynchronously on a Celery worker.
- Client polls or subscribes to WebSocket for progress updates.
- See [contracts/status.md](status.md) for status tracking contract.

## Playback Results

Completed uploads expose synchronized playback data from `GET /videos/{id}/results`.

### Example Response (HTTP 200 OK)

```json
{
  "video_id": "550e8400-e29b-41d4-a716-446655440000",
  "video_url": "https://localhost:8000/api/v1/video-analysis/jobs/550e8400-e29b-41d4-a716-446655440000/video/",
  "fps": 30,
  "duration_ms": 20000,
  "detections": {
    "0": [],
    "1": [
      {
        "id": "track_7",
        "parent_id": null,
        "source_model": "person_detection",
        "label": "student",
        "confidence": 0.97,
        "bbox": { "x1": 100, "y1": 200, "x2": 180, "y2": 360 }
      }
    ]
  }
}
```

### Result Rules

- `video_url` points to the original uploaded video asset or a server-provided stream URL.
- `fps` and `duration_ms` are used by the frontend to synchronize overlays with playback.
- `detections` must use full-frame coordinates and stable tracking IDs.
- The frontend overlays detections client-side and does not require a burned-in annotated export.

### Tracking Note

The backend tracking stage uses Ultralytics-supported trackers only: `bytetrack.yaml` for the default path and `botsort.yaml` when ReID-enabled tracking is required.

---

## Cross-References

- **[data-model.md](../data-model.md)**: VideoAnalysisJob entity definition.
- **[contracts/status.md](status.md)**: Job status and progress tracking.
- **[contracts/overlay.md](overlay.md)**: Model visibility controls.
- **[quickstart.md](../quickstart.md)**: Example end-to-end flow.
