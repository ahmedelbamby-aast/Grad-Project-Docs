# REST API Contracts: Student Behavior Detection Pyramid

**Branch**: `003-student-behavior-pyramid` | **Date**: 2026-04-24 | **Spec**: [../spec.md](../spec.md) | **Plan**: [../plan.md](../plan.md)

## Base URL

All endpoints are relative to the Django backend API root: `/api/v1/`

---

## Endpoints

### 1. List Detection Frames

**`GET /api/v1/detections/frames/`**

Returns paginated list of detection frames, including nested detection records.

**Authentication**: Session cookie (required)
**Permission**: `IsAuthenticated`

**Query Parameters**:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `session_id` | int | No | Filter frames by monitoring session |
| `camera_source_id` | int | No | Filter frames by camera source |
| `page` | int | No | Page number (default: 1) |
| `page_size` | int | No | Results per page (default: 20, max: 100) |

**Response `200 OK`**:

```json
{
  "count": 150,
  "next": "/api/v1/detections/frames/?page=2",
  "previous": null,
  "results": [
    {
      "id": 1,
      "camera_source": 3,
      "session": 1,
      "timestamp": "2026-04-24T10:30:00Z",
      "frame_number": 42,
      "raw_frame_ref": "/media/frames/session-1/frame-042.jpg",
      "detection_count": 21,
      "detections": [
        {
          "id": 101,
          "frame": 1,
          "detection_class": "student",
          "confidence": 0.95,
          "bbox_x1": 120,
          "bbox_y1": 200,
          "bbox_x2": 280,
          "bbox_y2": 500,
          "tracking_id": "S-001"
        },
        {
          "id": 102,
          "frame": 1,
          "detection_class": "teacher",
          "confidence": 0.92,
          "bbox_x1": 500,
          "bbox_y1": 100,
          "bbox_x2": 650,
          "bbox_y2": 480,
          "tracking_id": null
        }
      ]
    }
  ]
}
```

---

### 2. Retrieve Detection Frame

**`GET /api/v1/detections/frames/{frame_id}/`**

Returns a single detection frame with all nested detections.

**Authentication**: Session cookie (required)
**Permission**: `IsAuthenticated`

**Response `200 OK`**: Same structure as a single item in the list response.

**Response `404 Not Found`**:

```json
{
  "detail": "Not found."
}
```

---

### 3. List Playback Predictions

**`GET /api/v1/detections/predictions/playback/`**

Returns pyramid predictions for playback/review, optionally filtered by session.

**Authentication**: Session cookie (required)
**Permission**: `IsAuthenticated`

**Query Parameters**:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `session_id` | int | No | Filter predictions by monitoring session |

**Response `200 OK`**:

```json
{
  "count": 15,
  "results": [
    {
      "id": 201,
      "detection": 101,
      "tracking_id": "S-001",
      "posture": "sitting",
      "posture_confidence": 0.93,
      "horizontal_gaze": "looking_left",
      "horizontal_gaze_confidence": 0.87,
      "depth_gaze": "looking_forward",
      "depth_gaze_confidence": 0.91,
      "vertical_gaze": "looking_down",
      "vertical_gaze_confidence": 0.85,
      "constraint_violation": false,
      "timestamp": "2026-04-24T10:30:01Z"
    },
    {
      "id": 202,
      "detection": 103,
      "tracking_id": "S-002",
      "posture": "standing",
      "posture_confidence": 0.88,
      "horizontal_gaze": "",
      "horizontal_gaze_confidence": 0.0,
      "depth_gaze": "looking_backward",
      "depth_gaze_confidence": 0.76,
      "vertical_gaze": "looking_up",
      "vertical_gaze_confidence": 0.82,
      "constraint_violation": true,
      "timestamp": "2026-04-24T10:30:01Z"
    }
  ]
}
```

**Note**: Empty string values for behavior fields (e.g., `"horizontal_gaze": ""`) indicate that the corresponding model was unavailable or failed during inference (FR-014). The `constraint_violation` field indicates whether the rule engine flagged the combined behavior as anomalous.

---

### 4. Scan Model Inventory

**`POST /api/v1/pipeline/model-inventory/scan/`**

Scans root `models/` directory, detects artifacts by format, validates completeness (including OpenVINO pair checks), and persists inventory records.

**Authentication**: Session cookie (required)
**Permission**: `IsAdminUser`

**Request Body**:

```json
{
  "models_root": "models"
}
```

**Response `200 OK`**:

```json
{
  "status": "completed",
  "scanned_count": 6,
  "artifacts": [
    {
      "id": 1,
      "name": "yolo12l.pt",
      "format": "pt",
      "model_family": "yolo12l",
      "is_ready": true
    },
    {
      "id": 2,
      "name": "yolo12l.onnx",
      "format": "onnx",
      "model_family": "yolo12l",
      "is_ready": true
    }
  ],
  "missing_by_family": {
    "yolo12l": ["tensorrt", "openvino_ir"]
  }
}
```

---

### 5. Queue Export Jobs for Missing Formats

**`POST /api/v1/pipeline/model-exports/queue/`**

Queues sequential background export jobs for missing formats when `.pt` source is available.

**Authentication**: Session cookie (required)
**Permission**: `IsAdminUser`

**Request Body**:

```json
{
  "model_family": "yolo12l",
  "target_formats": ["onnx", "tensorrt", "openvino_ir"]
}
```

**Response `202 Accepted`**:

```json
{
  "status": "queued",
  "queued_jobs": [
    {"job_id": 11, "target_format": "onnx"},
    {"job_id": 12, "target_format": "tensorrt"},
    {"job_id": 13, "target_format": "openvino_ir"}
  ]
}
```

---

### 6. List Export Jobs

**`GET /api/v1/pipeline/model-exports/`**

Returns export job lifecycle states and logs.

**Authentication**: Session cookie (required)
**Permission**: `IsAuthenticated`

**Response `200 OK`**:

```json
{
  "count": 3,
  "results": [
    {
      "id": 11,
      "model_family": "yolo12l",
      "target_format": "onnx",
      "status": "completed",
      "started_at": "2026-04-29T12:01:00Z",
      "finished_at": "2026-04-29T12:01:21Z",
      "error_message": ""
    }
  ]
}
```

---

### 7. Start Benchmark Batch

**`POST /api/v1/pipeline/benchmarks/run/`**

Schedules benchmark runs for available model-format-target tuples using replay streams from root `Raw Data/`.

**Authentication**: Session cookie (required)
**Permission**: `IsAdminUser`

**Request Body**:

```json
{
  "models_root": "models",
  "replay_root": "Raw Data",
  "targets": ["cpu", "nvidia_gpu", "intel_runtime", "triton"],
  "explainability": true
}
```

**Response `202 Accepted`**:

```json
{
  "batch_id": "bench-20260429-001",
  "status": "queued",
  "scheduled_runs": 8
}
```

---

### 8. List Benchmark Runs

**`GET /api/v1/pipeline/benchmarks/`**

Returns benchmark run statuses and metric summaries.

**Authentication**: Session cookie (required)
**Permission**: `IsAuthenticated`

**Query Parameters**:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `batch_id` | string | No | Filter by benchmark batch |
| `model_family` | string | No | Filter by model family |

**Response `200 OK`**:

```json
{
  "count": 2,
  "results": [
    {
      "id": 101,
      "batch_id": "bench-20260429-001",
      "model_family": "yolo12l",
      "format": "onnx",
      "target_platform": "cpu",
      "status": "completed",
      "latency_ms": 48.2,
      "throughput_fps": 20.7,
      "warmup_ms": 1250.0,
      "memory_mb": 812.3,
      "vram_mb": 0.0,
      "error_rate": 0.0
    }
  ]
}
```

---

### 9. Get Deployment Matrix

**`GET /api/v1/pipeline/deployment-matrix/`**

Returns ranked deployment recommendations by model family and target platform.

**Authentication**: Session cookie (required)
**Permission**: `IsAuthenticated`

**Response `200 OK`**:

```json
{
  "generated_at": "2026-04-29T12:35:00Z",
  "entries": [
    {
      "model_family": "yolo12l",
      "platform": "cpu",
      "format": "onnx",
      "score": 0.91,
      "is_selected": true,
      "decision_reason": "Best latency/throughput score with stable error rate"
    },
    {
      "model_family": "yolo12l",
      "platform": "nvidia_gpu",
      "format": "tensorrt",
      "score": 0.96,
      "is_selected": true,
      "decision_reason": "Highest throughput on validated CUDA/TensorRT runtime"
    }
  ]
}
```

---

### 10. List Benchmark Figures and Explainability Artifacts

**`GET /api/v1/pipeline/benchmarks/artifacts/`**

Returns generated figure assets and explainability outputs for reporting workflows.

**Authentication**: Session cookie (required)
**Permission**: `IsAuthenticated`

**Response `200 OK`**:

```json
{
  "batch_id": "bench-20260429-001",
  "figures": [
    {"type": "comparison_bar", "path": "/media/benchmarks/bench-20260429-001/comparison-bar.png"},
    {"type": "trend_line", "path": "/media/benchmarks/bench-20260429-001/trend-line.png"},
    {"type": "matrix_heatmap", "path": "/media/benchmarks/bench-20260429-001/matrix-heatmap.png"}
  ],
  "explainability": [
    {"algorithm": "grad_cam", "path": "/media/benchmarks/bench-20260429-001/gradcam-yolo12l-onnx-001.png"}
  ]
}
```

---

## Error Responses

All endpoints return standard DRF error responses:

| Status | Description |
|--------|-------------|
| `401 Unauthorized` | Missing or invalid session cookie |
| `403 Forbidden` | User lacks required permissions |
| `404 Not Found` | Resource does not exist |
| `409 Conflict` | Requested export/benchmark operation invalid for current model/runtime state |
| `422 Unprocessable Entity` | Validation error (missing source model, invalid format mapping, etc.) |
| `500 Internal Server Error` | Unexpected server error |

```json
{
  "detail": "Authentication credentials were not provided."
}
```

## Related Documents

- [../spec.md](../spec.md) — Feature specification
- [../plan.md](../plan.md) — Implementation plan
- [../data-model.md](../data-model.md) — Data model definitions
- [websocket-api.md](websocket-api.md) — WebSocket message contracts
- [../../../.specify/memory/constitution.md](../../../.specify/memory/constitution.md) — Project governance
