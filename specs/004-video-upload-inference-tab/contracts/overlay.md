# API Contract: Model Visibility Toggles

**Feature**: 004-video-upload-inference-tab  
**Date**: 2026-04-30  
**Endpoint**: `PATCH /api/v1/video-analysis/jobs/{job_id}/visibility/`  

## Purpose

Allow users to show/hide bounding boxes, labels, and overlays from different AI models in real-time during uploaded playback or RTSP live monitoring. Changes apply immediately to all subsequent frame renders.

### Tracker Context

Overlay rendering is based on the tracked IDs returned by the shared inference pipeline. The supported built-in tracker paths are `bytetrack.yaml` and `botsort.yaml`.

The same visibility payload is applied to both uploaded-video playback and live RTSP overlays.

---

## Request

### Method & URL

**Method**: PATCH  
**URL**: `/api/v1/video-analysis/jobs/{job_id}/visibility/`  
**Content-Type**: `application/json`  
**Authentication**: Required (Bearer token or session cookie)

### Request Body

```json
{
  "models": {
    "person_detection": true,
    "sitting_standing": false,
    "attention_tracking": true,
    "hand_raising": true
  }
}
```

### Field Descriptions

| Field | Type | Description |
|-------|------|-------------|
| `models` | Object | Key-value pairs of model names and visibility state |
| `models.[model_name]` | Boolean | `true` to show, `false` to hide |

### Supported Model Names

| Model Name | Description |
|------------|-------------|
| `person_detection` | YOLO person/student/teacher detection (always required) |
| `sitting_standing` | Sitting vs. standing behavior classification |
| `attention_tracking` | Head/face attention direction tracking |
| `hand_raising` | Hand-raise detection |
| `[custom_model]` | Any future behavior classification models |

### Validation Rules

- **At least one model must be enabled**: Cannot set all models to `false`. System will reject this and return HTTP 400.
- **person_detection cannot be disabled**: This is the base layer; other models depend on it. Attempts to disable will be ignored; it will remain `true`.
- **Unknown model names**: Silently ignored (no error); only known models are updated.

### Example Request

```bash
curl -X PATCH http://localhost:8000/api/v1/video-analysis/jobs/550e8400-e29b-41d4-a716-446655440000/visibility/ \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "models": {
      "person_detection": true,
      "sitting_standing": false,
      "attention_tracking": true,
      "hand_raising": false
    }
  }'
```

---

## Response

### Success (HTTP 200 OK)

The server applies the changes and returns the updated visibility state.

```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "visibility_applied_at": "2026-04-30T10:32:15Z",
  "models": {
    "person_detection": true,
    "sitting_standing": false,
    "attention_tracking": true,
    "hand_raising": false
  },
  "affected_frame_range": {
    "from_frame": 8191,
    "to_frame": 18000
  },
  "message": "Visibility settings updated successfully",
  "_links": {
    "status": "/api/v1/video-analysis/jobs/550e8400-e29b-41d4-a716-446655440000/status/",
    "frames": "/api/v1/video-analysis/jobs/550e8400-e29b-41d4-a716-446655440000/frames/"
  }
}
```

### Field Descriptions

| Field | Type | Description |
|-------|------|-------------|
| `job_id` | UUID | Job identifier |
| `visibility_applied_at` | ISO 8601 | Timestamp when changes were applied |
| `models` | Object | Final visibility state (after server-side adjustments) |
| `affected_frame_range` | Object | Frames affected by this change (cached frames invalidated) |
| `message` | String | Status message |
| `_links` | Object | HATEOAS links to related resources |

---

## Error Responses

### HTTP 400 Bad Request

**Scenario 1**: No models enabled

```json
{
  "error": "VALIDATION_FAILED",
  "message": "At least one model must be enabled",
  "details": {
    "violation": "all_disabled",
    "resolution": "Enable at least one model (usually person_detection)"
  }
}
```

**Scenario 2**: Invalid request body

```json
{
  "error": "VALIDATION_FAILED",
  "message": "Invalid request format",
  "details": {
    "models": ["Must be an object with boolean values"]
  }
}
```

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

### HTTP 409 Conflict

**Scenario**: Job is still processing (not yet ready for visibility control)

```json
{
  "error": "JOB_NOT_READY",
  "message": "Cannot change visibility while job is still processing",
  "details": {
    "job_status": "processing",
    "progress_percent": 45,
    "resolution": "Wait for job to complete or switch to polling mode"
  }
}
```

### HTTP 403 Forbidden

**Scenario**: User does not have permission to modify this job

```json
{
  "error": "ACCESS_DENIED",
  "message": "You do not have permission to modify this job"
}
```

---

## Real-Time Updates

### Immediate Effect (Playback Impact)

- Changes are applied **immediately** to the rendering pipeline.
- If a frame is currently being rendered, the new visibility settings apply.
- Subsequent calls to `GET /api/v1/video-analysis/jobs/{job_id}/frames/{frame_number}/` will reflect the new state.

### Latency Target

- **Visibility change → visual update**: < 500ms (99th percentile)
- Frames are cached in Redis; visibility changes invalidate the cache for affected frames.

### WebSocket Notification (Optional)

If the job has an active WebSocket connection, the server broadcasts a visibility change event:

```json
{
  "type": "visibility_changed",
  "timestamp": "2026-04-30T10:32:15Z",
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "models": {
    "person_detection": true,
    "sitting_standing": false,
    "attention_tracking": true,
    "hand_raising": false
  }
}
```

---

## Data Persistence

### Storage

- Visibility state is stored in Redis (fast access during playback).
- On job completion, visibility settings are persisted to PostgreSQL for audit trail.

### Scope

- **Session-scoped**: Visibility settings are unique to the current playback session.
- **Per-job**: Each job has its own independent visibility state.
- **Reset on new playback**: If user closes playback and reopens, defaults to all models enabled.

---

## Use Cases

### Case 1: Isolate Person Detection

```json
{
  "models": {
    "person_detection": true,
    "sitting_standing": false,
    "attention_tracking": false,
    "hand_raising": false
  }
}
```

**Why**: Verify that the base detection layer correctly identifies all students before analyzing behavior.

### Case 2: Compare Multiple Behaviors

```json
{
  "models": {
    "person_detection": true,
    "sitting_standing": true,
    "attention_tracking": true,
    "hand_raising": false
  }
}
```

**Why**: Observe how students are classified across multiple behavior dimensions simultaneously.

### Case 3: Disable All Non-Critical Overlays

```json
{
  "models": {
    "person_detection": true,
    "sitting_standing": false,
    "attention_tracking": false,
    "hand_raising": false
  }
}
```

**Why**: Reduce visual clutter; focus on raw video with minimal annotation.

---

## Batch Visibility Changes

If calling this endpoint multiple times in quick succession (e.g., toggling multiple models), consider:

1. **Debouncing** (frontend): Wait 200-500ms before sending requests; batch changes.
2. **Or**: Send a single PATCH with all model changes at once (recommended).

### Example: Debounced Toggle (React)

```typescript
const [debounceTimer, setDebounceTimer] = useState<NodeJS.Timeout | null>(null);

const toggleModel = (modelName: string, enabled: boolean) => {
  if (debounceTimer) clearTimeout(debounceTimer);
  
  const timer = setTimeout(() => {
    sendVisibilityPatch({
      models: { ...currentVisibility, [modelName]: enabled }
    });
  }, 300);
  
  setDebounceTimer(timer);
};
```

---

## Cross-References

- **[data-model.md](../data-model.md)**: BoundingBox visibility field and storage.
- **[contracts/upload.md](upload.md)**: Job creation (default visibility: all enabled).
- **[contracts/status.md](status.md)**: Job status during playback (visibility changes are independent of status).
- **[quickstart.md](../quickstart.md)**: Example integration with playback UI.
