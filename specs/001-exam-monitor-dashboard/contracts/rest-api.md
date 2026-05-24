# API Contracts: REST Endpoints

**Feature**: 001-exam-monitor-dashboard | **Date**: 2026-02-27

All endpoints use JSON request/response bodies. Authentication is session-based (cookie).
Base URL: `/api/v1/`

---

## Authentication (`/api/v1/auth/`)

### POST `/api/v1/auth/login/`

**Purpose**: Authenticate user (instructor or admin) and create session (FR-001, FR-002)

**Request**:
```json
{
  "identifier": "instructor@university.edu",
  "password": "securePassword123"
}
```

`identifier` accepts either email or username (FR-001). The backend resolves the identifier to the correct account by checking both fields.

**Response 200**:
```json
{
  "id": "uuid",
  "username": "aelbamby",
  "email": "instructor@university.edu",
  "first_name": "Ahmed",
  "last_name": "ElBamby",
  "role": {
    "id": "uuid",
    "name": "Instructor",
    "permitted_pages": ["exam_board", "camera_feed", "predictions", "sessions", "recordings", "health", "settings"],
    "permitted_actions": ["triage_anomalies", "manage_cameras", "export_sessions", "add_comments"]
  },
  "theme_preference": "black-purple",
  "must_change_password": false
}
```

**Response 401**:
```json
{
  "error": "Invalid credentials",
  "code": "AUTH_FAILED"
}
```

**Notes**: Non-specific error message (FR-005). Sets `sessionid` cookie (HttpOnly, Secure, SameSite=Lax). Redirects: instructors → Exam Board, admins → Admin Dashboard (FR-003).

---

### POST `/api/v1/auth/logout/`

**Purpose**: Invalidate session (FR-004)

**Request**: Empty body. Session cookie required.

**Response 200**:
```json
{
  "message": "Logged out successfully"
}
```

---

### GET `/api/v1/auth/me/`

**Purpose**: Get current authenticated user profile

**Response 200**:
```json
{
  "id": "uuid",
  "username": "aelbamby",
  "email": "instructor@university.edu",
  "first_name": "Ahmed",
  "last_name": "ElBamby",
  "role": {
    "id": "uuid",
    "name": "Instructor",
    "permitted_pages": ["exam_board", "camera_feed", "predictions", "sessions", "recordings", "health", "settings"],
    "permitted_actions": ["triage_anomalies", "manage_cameras", "export_sessions", "add_comments"]
  },
  "theme_preference": "black-purple",
  "must_change_password": false,
  "last_login": "2026-02-27T10:30:00Z"
}
```

**Response 401**: `{ "error": "Authentication required", "code": "NOT_AUTHENTICATED" }`

---

### PATCH `/api/v1/auth/me/`

**Purpose**: Update user profile (theme preference)

**Request**:
```json
{
  "theme_preference": "full-black"
}
```

**Response 200**: Updated user object (same shape as GET).

---

### PUT `/api/v1/auth/me/password/`

**Purpose**: Change own password (FR-070). Requires current password verification.

**Request**:
```json
{
  "current_password": "oldPassword123",
  "new_password": "newSecurePassword456"
}
```

**Response 200**:
```json
{
  "message": "Password changed successfully"
}
```

**Response 400**: `{ "error": "Current password is incorrect", "code": "INVALID_CURRENT_PASSWORD" }`

---

## Camera Sources (`/api/v1/cameras/`)

### GET `/api/v1/cameras/`

**Purpose**: List cameras accessible to the user (FR-009). Cameras are system-wide shared resources.

**Query Parameters**: `?room_id=uuid&status=connected`

**Response 200**:
```json
{
  "count": 3,
  "results": [
    {
      "id": "uuid",
      "name": "Front Camera",
      "connection_type": "rtsp",
      "connection_url": "rtsp://192.168.1.100:554/stream",
      "status": "connected",
      "room": {
        "id": "uuid",
        "name": "Hall A"
      },
      "resolution_width": 1920,
      "resolution_height": 1080,
      "frame_rate": 30.0,
      "added_by": {
        "id": "uuid",
        "username": "aelbamby",
        "first_name": "Ahmed",
        "last_name": "ElBamby"
      },
      "created_at": "2026-02-27T09:00:00Z"
    }
  ]
}
```

---

### POST `/api/v1/cameras/`

**Purpose**: Add a new camera source (FR-006)

**Request**:
```json
{
  "name": "Front Camera",
  "connection_type": "rtsp",
  "connection_url": "rtsp://192.168.1.100:554/stream",
  "room_id": "uuid"
}
```

`room_id` is optional. Cameras without a room can be assigned to a room later by an admin.

**Response 201**: Created camera object.

**Response 400**:
```json
{
  "error": "Maximum 8 cameras per instructor",
  "code": "CAMERA_LIMIT_EXCEEDED"
}
```

**Response 400** (invalid RTSP):
```json
{
  "error": "RTSP URL does not match allowed patterns",
  "code": "INVALID_RTSP_URL"
}
```

---

### PATCH `/api/v1/cameras/{id}/`

**Purpose**: Update camera name

**Request**: `{ "name": "Updated Camera Name" }`

**Response 200**: Updated camera object.

---

### DELETE `/api/v1/cameras/{id}/`

**Purpose**: Remove a camera source (FR-009)

**Response 204**: No content.

---

### POST `/api/v1/cameras/{id}/connect/`

**Purpose**: Connect to camera and start receiving live preview frames (two-step flow: this is step 1 — preview only, no recording, no session per R-013). go2rtc registers the RTSP stream and makes it available via WHEP for WebRTC preview.

**Response 200**:
```json
{
  "id": "uuid",
  "status": "connected",
  "resolution_width": 1920,
  "resolution_height": 1080,
  "frame_rate": 30.0
}
```

**Response 503**: `{ "error": "Camera connection failed after 5 retries", "code": "CONNECTION_FAILED" }`

---

### POST `/api/v1/cameras/{id}/disconnect/`

**Purpose**: Disconnect camera without removing it (FR-009)

**Response 200**: `{ "id": "uuid", "status": "disconnected" }`

---

## Monitoring Sessions (`/api/v1/sessions/`)

### GET `/api/v1/sessions/`

**Purpose**: List current user's sessions

**Query Parameters**: `?status=active&exam_id=uuid&ordering=-started_at`

**Response 200**:
```json
{
  "count": 10,
  "next": "/api/v1/sessions/?page=2",
  "results": [
    {
      "id": "uuid",
      "status": "active",
      "exam": {
        "id": "uuid",
        "subject_name": "Data Structures",
        "subject_code": "CS201"
      },
      "user": {
        "id": "uuid",
        "username": "aelbamby",
        "first_name": "Ahmed",
        "last_name": "ElBamby"
      },
      "started_at": "2026-02-27T10:00:00Z",
      "ended_at": null,
      "camera_ids": ["uuid1", "uuid2"],
      "recording_count": 2
    }
  ]
}
```

---

### POST `/api/v1/sessions/`

**Purpose**: Start a monitoring session (begins recording per FR-026). Cameras must already be connected (two-step flow: connect → preview → start session → record per R-013).

**Request**:
```json
{
  "camera_ids": ["uuid1", "uuid2"],
  "exam_id": "uuid"
}
```

`exam_id` is optional when starting from Camera Feed page directly. When starting from Exam Board, exam_id is provided and cameras are auto-loaded from the exam's room (FR-056).

**Response 201**: Created session object. For each camera, either creates a new Recording or links to an existing active Recording (deduplicated per R-012). Detection pipeline starts for all cameras.

**Response 400**: `{ "error": "Camera {id} is not connected", "code": "CAMERA_NOT_CONNECTED" }` — cameras must be in `connected` status before starting a session.

**Response 409**: `{ "error": "Active session already exists", "code": "SESSION_ACTIVE" }`

---

### POST `/api/v1/sessions/{id}/end/`

**Purpose**: End a monitoring session

**Response 200**: `{ "id": "uuid", "status": "completed", "ended_at": "2026-02-27T12:00:00Z" }`

---

## Detections (`/api/v1/detections/`)

### GET `/api/v1/detections/frames/`

**Purpose**: Query detection frames for playback (FR-028)

**Query Parameters**: `?session_id=uuid&camera_id=uuid&from_ts=ISO8601&to_ts=ISO8601&page_size=100`

**Response 200**:
```json
{
  "count": 1500,
  "results": [
    {
      "id": 12345,
      "timestamp": "2026-02-27T10:05:30.123Z",
      "frame_number": 450,
      "detection_count": 25,
      "detections": [
        {
          "id": 67890,
          "detection_class": "student",
          "confidence": 0.92,
          "bbox": [100, 150, 250, 400],
          "tracking_id": "S-0012"
        }
      ]
    }
  ]
}
```

---

### GET `/api/v1/detections/predictions/`

**Purpose**: Query pyramid predictions for a student (FR-016, FR-030)

**Query Parameters**: `?tracking_id=S-0012&session_id=uuid&from_ts=ISO8601&to_ts=ISO8601`

**Response 200**:
```json
{
  "tracking_id": "S-0012",
  "predictions": [
    {
      "timestamp": "2026-02-27T10:05:30.123Z",
      "posture": "sitting",
      "posture_confidence": 0.89,
      "horizontal_gaze": "left",
      "horizontal_gaze_confidence": 0.75,
      "depth_gaze": "forward",
      "depth_gaze_confidence": 0.82,
      "vertical_gaze": "down",
      "vertical_gaze_confidence": 0.71,
      "constraint_violation": false
    }
  ]
}
```

---

## Anomalies (`/api/v1/anomalies/`)

### GET `/api/v1/anomalies/`

**Purpose**: List anomaly events (FR-017, FR-045, FR-068)

**Query Parameters**: `?session_id=uuid&camera_id=uuid&status=new,acknowledged&severity=high&ordering=-timestamp`

**Response 200**:
```json
{
  "count": 5,
  "results": [
    {
      "id": "uuid",
      "tracking_id": "S-0012",
      "severity": "high",
      "description": "Student S-0012 looking left with turned posture for >30 seconds",
      "behavior_started_at": "2026-02-27T10:14:25.000Z",
      "behavior_ended_at": null,
      "status": "new",
      "camera_source_id": "uuid",
      "timestamp": "2026-02-27T10:15:00Z",
      "prediction_snapshot": {
        "posture": "sitting",
        "horizontal_gaze": "left",
        "depth_gaze": "forward",
        "vertical_gaze": "down"
      },
      "notes_count": 0,
      "dismissed_reason": null,
      "status_changed_by": null,
      "status_changed_at": null
    }
  ]
}
```

`behavior_ended_at` is `null` when the behavior is ongoing (FR-068).

---

### POST `/api/v1/anomalies/{id}/acknowledge/`

**Purpose**: Acknowledge an anomaly alert (FR-041)

**Request**: Empty body.

**Response 200**:
```json
{
  "id": "uuid",
  "status": "acknowledged",
  "status_changed_by": "user-uuid",
  "status_changed_at": "2026-02-27T10:16:00Z"
}
```

**Response 400**: `{ "error": "Anomaly is already dismissed", "code": "INVALID_STATUS_TRANSITION" }`
**Response 409**: `{ "error": "Already triaged by [Instructor Name]", "code": "ALREADY_TRIAGED" }` — first-write-wins.

---

### POST `/api/v1/anomalies/{id}/dismiss/`

**Purpose**: Dismiss an anomaly with mandatory reason (FR-042)

**Request**:
```json
{
  "reason": "False positive - student was stretching, not cheating"
}
```

**Response 200**:
```json
{
  "id": "uuid",
  "status": "dismissed",
  "dismissed_reason": "False positive - student was stretching, not cheating",
  "status_changed_by": "user-uuid",
  "status_changed_at": "2026-02-27T10:17:00Z"
}
```

**Response 400**: `{ "error": "Reason must be at least 5 characters", "code": "REASON_TOO_SHORT" }`

---

### POST `/api/v1/anomalies/{id}/notes/`

**Purpose**: Add a timestamped note to an anomaly (FR-043)

**Request**:
```json
{
  "content": "Student appeared to be looking at neighbor's paper. Monitoring closely."
}
```

**Response 201**:
```json
{
  "id": "uuid",
  "anomaly_event_id": "uuid",
  "user_id": "uuid",
  "user_name": "Ahmed ElBamby",
  "content": "Student appeared to be looking at neighbor's paper. Monitoring closely.",
  "created_at": "2026-02-27T10:18:00Z"
}
```

---

### GET `/api/v1/anomalies/{id}/notes/`

**Purpose**: List notes for an anomaly

**Response 200**:
```json
{
  "results": [
    {
      "id": "uuid",
      "user_name": "Ahmed ElBamby",
      "content": "Student appeared to be looking at neighbor's paper.",
      "created_at": "2026-02-27T10:18:00Z"
    }
  ]
}
```

---

## Recordings (`/api/v1/recordings/`)

### GET `/api/v1/recordings/`

**Purpose**: List recordings browsable by session (FR-027)

**Query Parameters**: `?session_id=uuid&camera_id=uuid&ordering=-started_at`

**Response 200**:
```json
{
  "count": 2,
  "results": [
    {
      "id": "uuid",
      "session_id": "uuid",
      "camera_source_id": "uuid",
      "camera_name": "Front Camera",
      "duration_seconds": 3600,
      "file_size_bytes": 524288000,
      "status": "completed",
      "started_at": "2026-02-27T10:00:00Z",
      "ended_at": "2026-02-27T11:00:00Z",
      "anomaly_count": 3
    }
  ]
}
```

---

### GET `/api/v1/recordings/{id}/stream/`

**Purpose**: Stream video for playback (FR-028)

**Response 200**: Video stream with `Content-Type: video/mp4`, supports Range requests for seeking.

**Headers**: `Accept-Ranges: bytes`, `Content-Length`, `Content-Range`.

---

### GET `/api/v1/recordings/{id}/anomaly-markers/`

**Purpose**: Get anomaly timestamp markers for playback timeline (FR-029)

**Response 200**:
```json
{
  "recording_id": "uuid",
  "markers": [
    {
      "anomaly_id": "uuid",
      "timestamp_offset_seconds": 125.5,
      "severity": "high",
      "description": "Student S-0012 looking left >30 seconds",
      "tracking_id": "S-0012"
    }
  ]
}
```

---

## Health (`/api/v1/health/`)

### GET `/api/v1/health/`

**Purpose**: Health check endpoint for external monitoring (FR-039)

**Response 200** (no auth required):
```json
{
  "status": "healthy",
  "timestamp": "2026-02-27T10:30:00Z",
  "subsystems": {
    "api": { "status": "up", "response_time_ms": 12 },
    "websocket": { "status": "up", "active_connections": 15 },
    "detection_pipeline": { "status": "up", "cameras_active": 8, "avg_fps": 22.5 },
    "database": { "status": "up", "response_time_ms": 3 },
    "redis": { "status": "up", "response_time_ms": 1 },
    "storage": {
      "status": "ok",
      "used_bytes": 10737418240,
      "total_bytes": 53687091200,
      "used_percent": 20.0
    }
  }
}
```

**Response 503** (degraded):
```json
{
  "status": "degraded",
  "timestamp": "2026-02-27T10:30:00Z",
  "subsystems": {
    "detection_pipeline": { "status": "down", "error": "No active inference workers" }
  }
}
```

---

### GET `/api/v1/health/dashboard/`

**Purpose**: Detailed health data for the Health Dashboard page (FR-038, auth required)

**Response 200**:
```json
{
  "api_status": "up",
  "websocket_health": {
    "status": "up",
    "active_connections": 15,
    "message_rate_per_sec": 1200
  },
  "detection_fps": {
    "camera-uuid-1": 24.5,
    "camera-uuid-2": 22.1
  },
  "storage": {
    "used_bytes": 10737418240,
    "total_bytes": 53687091200,
    "used_percent": 20.0,
    "warning_level": null
  },
  "active_sessions": 3,
  "uptime_seconds": 86400
}
```

---

## Session Exports (`/api/v1/sessions/{id}/export/`)

### POST `/api/v1/sessions/{id}/export/`

**Purpose**: Initiate a full session archive export (FR-047, FR-048)

The export is an asynchronous Celery task. The endpoint returns immediately with a job ID. The client polls the job status endpoint until `status` is `completed`, then downloads the archive.

**Request**: No body required (exports the entire session).

**Response 202**:
```json
{
  "export_id": "uuid",
  "session_id": "uuid",
  "status": "pending",
  "created_at": "2026-02-27T10:30:00Z",
  "message": "Export job queued"
}
```

**Response 404**: Session not found.
**Response 409**: Export already in progress for this session.

---

### GET `/api/v1/sessions/{id}/export/{export_id}/`

**Purpose**: Check export job status (FR-047)

**Response 200** (in progress):
```json
{
  "export_id": "uuid",
  "session_id": "uuid",
  "status": "processing",
  "progress_percent": 45,
  "created_at": "2026-02-27T10:30:00Z"
}
```

**Response 200** (completed):
```json
{
  "export_id": "uuid",
  "session_id": "uuid",
  "status": "completed",
  "progress_percent": 100,
  "download_url": "/api/v1/sessions/{id}/export/{export_id}/download/",
  "file_size_bytes": 524288000,
  "created_at": "2026-02-27T10:30:00Z",
  "completed_at": "2026-02-27T10:35:00Z"
}
```

**Response 200** (failed):
```json
{
  "export_id": "uuid",
  "session_id": "uuid",
  "status": "failed",
  "error": "Insufficient disk space for export archive",
  "created_at": "2026-02-27T10:30:00Z"
}
```

---

### GET `/api/v1/sessions/{id}/export/{export_id}/download/`

**Purpose**: Download the exported session archive (FR-048)

**Response 200**: Binary ZIP file download.
- Content-Type: `application/zip`
- Content-Disposition: `attachment; filename="session_{id}_export.zip"`

**Archive contents** (non-anonymized per Clarification Round 2):
```
session_{id}_export/
├── recordings/              # Raw video MP4 files (one per camera)
│   ├── camera_{uuid}.mp4
│   └── ...
├── detections.json          # All DetectionFrame + Detection + PyramidPrediction data
├── anomalies.json           # All AnomalyEvent records with triage status + notes
├── session_metadata.json    # MonitoringSession info, camera list, instructor
└── tracking_ids.json        # Student tracking ID mapping (non-anonymized)
```

**Response 404**: Export not found or not yet completed.

---

## Dashboard (`/api/v1/dashboard/`)

### GET `/api/v1/dashboard/`

**Purpose**: Role-based dashboard data (FR-046). Response shape depends on the authenticated user's role.

**Response 200 (Instructor)**:
```json
{
  "role": "instructor",
  "active_sessions": {
    "count": 1,
    "sessions": [
      {
        "id": "uuid",
        "exam_name": "Data Structures - Midterm",
        "started_at": "2026-02-27T09:00:00Z",
        "camera_count": 4,
        "anomaly_count": 3
      }
    ]
  },
  "camera_status": {
    "connected": 6,
    "disconnected": 2,
    "total": 8
  },
  "upcoming_exams": {
    "count": 2,
    "next_exam_starts_at": "2026-02-27T14:00:00Z"
  }
}
```

**Response 200 (Admin — FR-046, FR-071, FR-073)**:
```json
{
  "role": "admin",
  "active_sessions": {
    "count": 3,
    "sessions": [
      {
        "id": "uuid",
        "exam_name": "Data Structures - Midterm",
        "instructor": {
          "id": "uuid",
          "username": "aelbamby",
          "first_name": "Ahmed",
          "last_name": "ElBamby"
        },
        "started_at": "2026-02-27T09:00:00Z",
        "camera_count": 4,
        "anomaly_count": 5
      }
    ]
  },
  "stats": {
    "total_instructors": 12,
    "total_exams": 25,
    "connected_cameras": 18,
    "storage": {
      "used_bytes": 10737418240,
      "total_bytes": 53687091200,
      "used_percent": 20.0
    }
  }
}
```

**Notes**: Admin dashboard anomaly feed and activity feed are separate paginated endpoints (below) to avoid bloating the main dashboard payload.

---

### GET `/api/v1/dashboard/admin/anomaly-feed/`

**Purpose**: Unified real-time anomaly feed for admin dashboard (FR-071). Returns recent high/medium severity anomalies across all active sessions.

**Query Parameters**: `?severity=high,medium&page_size=20`

**Response 200**:
```json
{
  "count": 15,
  "next": "/api/v1/dashboard/admin/anomaly-feed/?page=2",
  "results": [
    {
      "id": "uuid",
      "severity": "high",
      "tracking_id": "S-0012",
      "description": "Student S-0012 looking left with turned posture for >30 seconds",
      "behavior_started_at": "2026-02-27T10:14:25.000Z",
      "behavior_ended_at": null,
      "timestamp": "2026-02-27T10:15:00Z",
      "exam": {
        "id": "uuid",
        "subject_name": "Data Structures",
        "subject_code": "CS201"
      },
      "session_id": "uuid",
      "instructor": {
        "id": "uuid",
        "first_name": "Ahmed",
        "last_name": "ElBamby"
      },
      "status": "new"
    }
  ]
}
```

**Permission**: Admin only (FR-052).

---

### GET `/api/v1/dashboard/admin/activity-feed/`

**Purpose**: Recent system activity backed by audit log (FR-073)

**Query Parameters**: `?page_size=50&action_type=login,session_start`

`page_size` accepts: 10, 20, 30, 40, 50, 60, 70, 80, 90, 100, or `all` (default: 50). When `all` is selected, results are paginated with lazy loading to avoid performance degradation (FR-073).

**Response 200**:
```json
{
  "count": 200,
  "next": "/api/v1/dashboard/admin/activity-feed/?page=2",
  "results": [
    {
      "id": 12345,
      "action_type": "session_start",
      "action_label": "Session Started",
      "user": {
        "id": "uuid",
        "username": "aelbamby",
        "first_name": "Ahmed",
        "last_name": "ElBamby"
      },
      "target_summary": "Monitoring Session for Data Structures Midterm",
      "timestamp": "2026-02-27T10:00:00Z"
    }
  ]
}
```

**Permission**: Admin only (FR-052).

---

## Exam Board — Instructor (`/api/v1/exams/`)

### GET `/api/v1/exams/my/`

**Purpose**: List exams assigned to the current instructor (FR-054, FR-055)

**Response 200**:
```json
{
  "count": 3,
  "results": [
    {
      "id": "uuid",
      "subject_name": "Data Structures",
      "subject_type": "midterm",
      "subject_code": "CS201",
      "credit_hours": 3,
      "expected_student_count": 45,
      "scheduled_start": "2026-02-27T14:00:00Z",
      "scheduled_end": "2026-02-27T16:00:00Z",
      "duration_minutes": 120,
      "room": {
        "id": "uuid",
        "name": "Hall A",
        "camera_count": 4
      },
      "status": "scheduled",
      "active_session_id": null,
      "students": [
        {
          "id": "uuid",
          "student_name": "Mohamed Ali",
          "university_id": "20210001",
          "seat_number": "A1"
        }
      ]
    }
  ]
}
```

**Notes**: `active_session_id` is non-null when the exam has an active monitoring session (FR-072). `status` is one of: `scheduled`, `active`, `completed`.

---

### POST `/api/v1/exams/{id}/start-session/`

**Purpose**: Start a monitoring session from the Exam Board (FR-055, FR-056). Auto-loads cameras from the exam's assigned room.

**Request**: Empty body. The system auto-detects cameras registered to the exam's room.

**Response 201**:
```json
{
  "session_id": "uuid",
  "exam_id": "uuid",
  "cameras_loaded": 4,
  "camera_ids": ["uuid1", "uuid2", "uuid3", "uuid4"],
  "status": "active",
  "started_at": "2026-02-27T14:00:00Z"
}
```

**Response 400**: `{ "error": "No cameras registered for this room", "code": "NO_ROOM_CAMERAS" }` (FR-056)
**Response 409**: `{ "error": "Active session already exists for this exam", "code": "SESSION_ACTIVE" }`

---

## Instructor Comments (`/api/v1/sessions/{id}/comments/`)

### GET `/api/v1/sessions/{id}/comments/`

**Purpose**: List comments for a session (FR-058, FR-059)

**Query Parameters**: `?camera_id=uuid&ordering=created_at`

**Response 200**:
```json
{
  "results": [
    {
      "id": "uuid",
      "user": {
        "id": "uuid",
        "username": "aelbamby",
        "first_name": "Ahmed",
        "last_name": "ElBamby",
        "role_name": "Instructor"
      },
      "camera_source_id": "uuid",
      "content": "Student in row 3 appears nervous, fidgeting frequently",
      "created_at": "2026-02-27T10:25:00Z"
    }
  ]
}
```

---

### POST `/api/v1/sessions/{id}/comments/`

**Purpose**: Add a timestamped comment to a session (FR-058)

**Request**:
```json
{
  "content": "Student in row 3 appears nervous, fidgeting frequently",
  "camera_source_id": "uuid"
}
```

`camera_source_id` is optional — if omitted, the comment is a general session observation.

**Response 201**: Created comment object (same shape as list item).

**Response 403**: `{ "error": "Not assigned to this exam", "code": "NOT_ASSIGNED" }` — only assigned instructors and admins can comment.

---

## Merged Session View (`/api/v1/exams/{id}/sessions/merged/`)

### GET `/api/v1/exams/{id}/sessions/merged/`

**Purpose**: Read-time aggregation of all instructor sessions for a completed exam (FR-R-017). Shows unified history with full attribution.

**Query Parameters**: `?include=comments,anomalies,triage_history`

**Response 200**:
```json
{
  "exam_id": "uuid",
  "exam_name": "Data Structures - Midterm",
  "sessions": [
    {
      "id": "uuid",
      "instructor": {
        "id": "uuid",
        "first_name": "Ahmed",
        "last_name": "ElBamby"
      },
      "started_at": "2026-02-27T14:00:00Z",
      "ended_at": "2026-02-27T16:05:00Z",
      "camera_count": 4
    }
  ],
  "merged_timeline": {
    "anomalies": [
      {
        "id": "uuid",
        "tracking_id": "S-0012",
        "severity": "high",
        "description": "Student S-0012 looking left >30 seconds",
        "timestamp": "2026-02-27T14:35:00Z",
        "status": "acknowledged",
        "triaged_by": {
          "id": "uuid",
          "first_name": "Ahmed",
          "last_name": "ElBamby"
        }
      }
    ],
    "comments": [
      {
        "id": "uuid",
        "user": {
          "id": "uuid",
          "first_name": "Ahmed",
          "last_name": "ElBamby",
          "role_name": "Instructor"
        },
        "content": "Row 3 seems restless",
        "timestamp": "2026-02-27T14:20:00Z"
      }
    ],
    "triage_actions": [
      {
        "anomaly_id": "uuid",
        "action": "acknowledged",
        "performed_by": {
          "id": "uuid",
          "first_name": "Ahmed",
          "last_name": "ElBamby"
        },
        "timestamp": "2026-02-27T14:36:00Z"
      }
    ]
  }
}
```

---

## Admin User Management (`/api/v1/admin/users/`)

**Permission**: All endpoints in this section require Admin role (FR-052, FR-060).

### GET `/api/v1/admin/users/`

**Purpose**: List all users (FR-060)

**Query Parameters**: `?role_id=uuid&is_active=true&search=ahmed&ordering=last_name`

**Response 200**:
```json
{
  "count": 12,
  "results": [
    {
      "id": "uuid",
      "username": "aelbamby",
      "email": "instructor@university.edu",
      "first_name": "Ahmed",
      "last_name": "ElBamby",
      "role": {
        "id": "uuid",
        "name": "Instructor"
      },
      "is_active": true,
      "must_change_password": false,
      "last_login": "2026-02-27T10:30:00Z",
      "created_at": "2026-01-15T08:00:00Z"
    }
  ]
}
```

---

### POST `/api/v1/admin/users/`

**Purpose**: Create a new user account (FR-060)

**Request**:
```json
{
  "username": "mali",
  "email": "mali@university.edu",
  "first_name": "Mohamed",
  "last_name": "Ali",
  "password": "initialPassword123",
  "role_id": "uuid"
}
```

**Response 201**: Created user object.
**Response 400**: `{ "error": "Username already exists", "code": "DUPLICATE_USERNAME" }`
**Response 400**: `{ "error": "Email already exists", "code": "DUPLICATE_EMAIL" }`

---

### GET `/api/v1/admin/users/{id}/`

**Purpose**: Get user details

**Response 200**: Full user object with role details.

---

### PATCH `/api/v1/admin/users/{id}/`

**Purpose**: Edit user account (FR-060). Supports partial updates.

**Request**:
```json
{
  "first_name": "Mohamed",
  "role_id": "uuid"
}
```

**Response 200**: Updated user object. Role changes are logged in the audit trail (FR-062).

---

### DELETE `/api/v1/admin/users/{id}/`

**Purpose**: Deactivate a user account (FR-060). Soft-delete — sets `is_active=false`.

**Response 200**:
```json
{
  "id": "uuid",
  "is_active": false,
  "message": "Account deactivated"
}
```

**Notes**: If the user has an active session, it is gracefully terminated (edge case: "Admin deactivates instructor with active session").

---

### POST `/api/v1/admin/users/{id}/reset-password/`

**Purpose**: Admin-initiated password reset (FR-060)

**Request**:
```json
{
  "new_password": "tempPassword456"
}
```

**Response 200**:
```json
{
  "message": "Password reset successfully",
  "must_change_password": true
}
```

**Notes**: Sets `must_change_password = true` on the user record (FR-060). On the user's next login, `ForcePasswordChangeMiddleware` redirects them to the password-change page. The flag is cleared after a successful password change.

---

## Admin Role Management (`/api/v1/admin/roles/`)

**Permission**: Admin only (FR-052, FR-061).

### GET `/api/v1/admin/roles/`

**Purpose**: List all roles (FR-061)

**Response 200**:
```json
{
  "count": 4,
  "results": [
    {
      "id": "uuid",
      "name": "Admin",
      "is_builtin": true,
      "permitted_pages": ["*"],
      "permitted_actions": ["*"],
      "user_count": 2,
      "created_at": "2026-01-01T00:00:00Z"
    },
    {
      "id": "uuid",
      "name": "Instructor",
      "is_builtin": true,
      "permitted_pages": ["exam_board", "camera_feed", "predictions", "sessions", "recordings", "health", "settings"],
      "permitted_actions": ["triage_anomalies", "manage_cameras", "export_sessions", "add_comments"],
      "user_count": 10,
      "created_at": "2026-01-01T00:00:00Z"
    },
    {
      "id": "uuid",
      "name": "Proctor",
      "is_builtin": false,
      "permitted_pages": ["exam_board", "camera_feed"],
      "permitted_actions": ["triage_anomalies", "add_comments"],
      "user_count": 3,
      "created_at": "2026-02-15T09:00:00Z"
    }
  ]
}
```

---

### POST `/api/v1/admin/roles/`

**Purpose**: Create a custom role (FR-061)

**Request**:
```json
{
  "name": "Proctor",
  "permitted_pages": ["exam_board", "camera_feed"],
  "permitted_actions": ["triage_anomalies", "add_comments"]
}
```

**Response 201**: Created role object.
**Response 400**: `{ "error": "Role name already exists", "code": "DUPLICATE_ROLE_NAME" }`

---

### GET `/api/v1/admin/roles/{id}/`

**Purpose**: Get role details with assigned user list

**Response 200**: Role object with `users` array.

---

### PATCH `/api/v1/admin/roles/{id}/`

**Purpose**: Edit role permissions (FR-061)

**Request**:
```json
{
  "permitted_pages": ["exam_board", "camera_feed", "recordings"],
  "permitted_actions": ["triage_anomalies", "add_comments", "export_sessions"]
}
```

**Response 200**: Updated role object.
**Response 400**: `{ "error": "Cannot modify built-in role", "code": "BUILTIN_ROLE" }` — built-in Admin and Instructor roles cannot be edited.

---

### DELETE `/api/v1/admin/roles/{id}/`

**Purpose**: Delete a custom role

**Response 204**: No content.
**Response 400**: `{ "error": "Cannot delete built-in role", "code": "BUILTIN_ROLE" }`
**Response 409**: `{ "error": "Role has assigned users. Reassign them first.", "code": "ROLE_IN_USE" }`

---

## Admin Room Management (`/api/v1/admin/rooms/`)

**Permission**: Admin only (FR-052, FR-063).

### GET `/api/v1/admin/rooms/`

**Purpose**: List all rooms (FR-063)

**Query Parameters**: `?search=hall&ordering=name`

**Response 200**:
```json
{
  "count": 5,
  "results": [
    {
      "id": "uuid",
      "name": "Hall A",
      "building": "Engineering Building",
      "capacity": 120,
      "camera_count": 4,
      "cameras": [
        {
          "id": "uuid",
          "name": "Front Camera",
          "status": "connected"
        }
      ],
      "created_at": "2026-01-10T08:00:00Z"
    }
  ]
}
```

---

### POST `/api/v1/admin/rooms/`

**Purpose**: Create a room (FR-063)

**Request**:
```json
{
  "name": "Hall A",
  "building": "Engineering Building",
  "capacity": 120
}
```

**Response 201**: Created room object.
**Response 400**: `{ "error": "Room name already exists", "code": "DUPLICATE_ROOM_NAME" }`

---

### PATCH `/api/v1/admin/rooms/{id}/`

**Purpose**: Edit room details

**Request**: Partial update (name, building, capacity).

**Response 200**: Updated room object.

---

### DELETE `/api/v1/admin/rooms/{id}/`

**Purpose**: Delete a room. Cameras in this room have their `room_id` set to null.

**Response 204**: No content.
**Response 409**: `{ "error": "Room has active exams. Complete or reassign them first.", "code": "ROOM_IN_USE" }`

---

### POST `/api/v1/admin/rooms/{id}/cameras/`

**Purpose**: Assign a camera to a room

**Request**:
```json
{
  "camera_id": "uuid"
}
```

**Response 200**: Updated camera object with room assignment.

---

### DELETE `/api/v1/admin/rooms/{id}/cameras/{camera_id}/`

**Purpose**: Remove a camera from a room (sets `room_id` to null)

**Response 204**: No content.

---

## Admin Exam Management (`/api/v1/admin/exams/`)

**Permission**: Admin only (FR-052, FR-057, FR-063).

### GET `/api/v1/admin/exams/`

**Purpose**: List all exams (FR-063)

**Query Parameters**: `?status=scheduled&room_id=uuid&instructor_id=uuid&ordering=-scheduled_start`

**Response 200**:
```json
{
  "count": 25,
  "results": [
    {
      "id": "uuid",
      "subject_name": "Data Structures",
      "subject_type": "midterm",
      "subject_code": "CS201",
      "credit_hours": 3,
      "expected_student_count": 45,
      "scheduled_start": "2026-02-27T14:00:00Z",
      "scheduled_end": "2026-02-27T16:00:00Z",
      "duration_minutes": 120,
      "room": {
        "id": "uuid",
        "name": "Hall A"
      },
      "assigned_instructors": [
        {
          "id": "uuid",
          "username": "aelbamby",
          "first_name": "Ahmed",
          "last_name": "ElBamby"
        }
      ],
      "student_count": 45,
      "status": "scheduled",
      "created_at": "2026-02-20T10:00:00Z"
    }
  ]
}
```

---

### POST `/api/v1/admin/exams/`

**Purpose**: Create a new exam (FR-063)

**Request**:
```json
{
  "subject_name": "Data Structures",
  "subject_type": "midterm",
  "subject_code": "CS201",
  "credit_hours": 3,
  "expected_student_count": 45,
  "scheduled_start": "2026-02-27T14:00:00Z",
  "duration_minutes": 120,
  "room_id": "uuid",
  "assigned_instructor_ids": ["uuid1", "uuid2"]
}
```

**Response 201**: Created exam object.

---

### GET `/api/v1/admin/exams/{id}/`

**Purpose**: Get exam details including student roster

**Response 200**: Full exam object with `students` array.

---

### PATCH `/api/v1/admin/exams/{id}/`

**Purpose**: Edit exam details (FR-063)

**Response 200**: Updated exam object.

---

### DELETE `/api/v1/admin/exams/{id}/`

**Purpose**: Delete a scheduled exam

**Response 204**: No content.
**Response 409**: `{ "error": "Cannot delete an active or completed exam", "code": "EXAM_NOT_DELETABLE" }`

---

### GET `/api/v1/admin/exams/{id}/students/`

**Purpose**: List student roster for an exam (FR-057)

**Response 200**:
```json
{
  "count": 45,
  "results": [
    {
      "id": "uuid",
      "student_name": "Mohamed Ali",
      "university_id": "20210001",
      "seat_number": "A1"
    }
  ]
}
```

---

### POST `/api/v1/admin/exams/{id}/students/`

**Purpose**: Add a student to the exam roster

**Request**:
```json
{
  "student_name": "Mohamed Ali",
  "university_id": "20210001",
  "seat_number": "A1"
}
```

**Response 201**: Created exam student object.
**Response 400**: `{ "error": "University ID already exists in this exam", "code": "DUPLICATE_STUDENT" }`
**Response 400**: `{ "error": "Seat number already assigned in this exam", "code": "DUPLICATE_SEAT" }`

---

### DELETE `/api/v1/admin/exams/{id}/students/{student_id}/`

**Purpose**: Remove a student from the exam roster

**Response 204**: No content.

---

## Admin Recording Review (`/api/v1/admin/sessions/`)

**Permission**: Admin only (FR-052, FR-064–FR-066).

### GET `/api/v1/admin/sessions/`

**Purpose**: Browse all sessions from all instructors (FR-064)

**Query Parameters**: `?instructor_id=uuid&exam_id=uuid&from_date=2026-02-01&to_date=2026-02-28&min_anomalies=1&ordering=-started_at`

**Response 200**:
```json
{
  "count": 50,
  "results": [
    {
      "id": "uuid",
      "instructor": {
        "id": "uuid",
        "first_name": "Ahmed",
        "last_name": "ElBamby"
      },
      "exam": {
        "id": "uuid",
        "subject_name": "Data Structures",
        "subject_code": "CS201"
      },
      "started_at": "2026-02-27T10:00:00Z",
      "ended_at": "2026-02-27T12:00:00Z",
      "status": "completed",
      "camera_count": 4,
      "anomaly_count": 8,
      "triage_summary": {
        "acknowledged": 5,
        "dismissed": 2,
        "new": 1
      }
    }
  ]
}
```

---

### GET `/api/v1/admin/sessions/{id}/review/`

**Purpose**: Full session review data (FR-065)

**Response 200**:
```json
{
  "session": {
    "id": "uuid",
    "instructor": { "id": "uuid", "first_name": "Ahmed", "last_name": "ElBamby" },
    "exam": { "id": "uuid", "subject_name": "Data Structures" },
    "started_at": "2026-02-27T10:00:00Z",
    "ended_at": "2026-02-27T12:00:00Z"
  },
  "cameras": [
    {
      "id": "uuid",
      "name": "Front Camera",
      "recording_id": "uuid"
    }
  ],
  "anomalies": [
    {
      "id": "uuid",
      "tracking_id": "S-0012",
      "severity": "high",
      "description": "Looking left >30s",
      "behavior_started_at": "2026-02-27T10:14:25.000Z",
      "behavior_ended_at": "2026-02-27T10:15:30.000Z",
      "status": "dismissed",
      "dismissed_reason": "False positive",
      "triaged_by": { "id": "uuid", "first_name": "Ahmed", "last_name": "ElBamby" },
      "triaged_at": "2026-02-27T10:16:00Z",
      "notes": [
        { "id": "uuid", "user_name": "Ahmed ElBamby", "content": "Monitoring closely", "created_at": "..." }
      ]
    }
  ],
  "comments": [
    {
      "id": "uuid",
      "user": { "id": "uuid", "first_name": "Ahmed", "last_name": "ElBamby", "role_name": "Instructor" },
      "content": "Student in row 3 appears nervous",
      "camera_source_id": "uuid",
      "created_at": "2026-02-27T10:25:00Z"
    }
  ]
}
```

---

### POST `/api/v1/admin/anomalies/{id}/revert/`

**Purpose**: Revert a triage action back to `new` status (FR-066)

**Request**:
```json
{
  "reason": "Need to re-evaluate this anomaly based on new evidence"
}
```

**Response 200**:
```json
{
  "id": "uuid",
  "status": "new",
  "reverted_by": {
    "id": "uuid",
    "first_name": "Admin",
    "last_name": "User"
  },
  "revert_reason": "Need to re-evaluate this anomaly based on new evidence",
  "reverted_at": "2026-02-27T14:00:00Z"
}
```

**Response 400**: `{ "error": "Anomaly is already in 'new' status", "code": "ALREADY_NEW" }`
**Response 400**: `{ "error": "Reason must be at least 5 characters", "code": "REASON_TOO_SHORT" }`

**Notes**: Triggers WebSocket broadcast to all connected viewers (FR-067). Logged in audit trail (FR-066, FR-069).

---

## Admin Recording Management (`/api/v1/admin/recordings/`)

**Permission**: Admin only (FR-031, FR-052).

### DELETE `/api/v1/admin/recordings/{id}/`

**Purpose**: Admin-only manual deletion of a completed recording (FR-031)

**Response 204**: No content (recording deleted successfully).

**Response 400**:
```json
{
  "error": "Cannot delete an active recording",
  "code": "RECORDING_ACTIVE"
}
```

**Response 404**: `{ "error": "Recording not found", "code": "NOT_FOUND" }`

**Notes**: Only completed recordings can be deleted; active recordings are rejected with 400. Deletion removes the video file from storage. Logged as `recording_access` in the audit trail (FR-069). Automatic retention cleanup (Celery periodic task, default 30 days) operates independently of manual deletion.

---

## Admin Audit Log (`/api/v1/admin/audit-log/`)

**Permission**: Admin only (FR-052, FR-069).

### GET `/api/v1/admin/audit-log/`

**Purpose**: Query full audit log (FR-069, FR-073)

**Query Parameters**: `?action_type=login,logout&user_id=uuid&from_date=2026-02-01&to_date=2026-02-28&page_size=50&ordering=-timestamp`

`page_size` accepts: 10, 20, 30, 40, 50, 60, 70, 80, 90, 100, or `all` (default: 50).

**Response 200**:
```json
{
  "count": 500,
  "next": "/api/v1/admin/audit-log/?page=2",
  "results": [
    {
      "id": 12345,
      "action_type": "triage_dismiss",
      "action_label": "Anomaly Dismissed",
      "user": {
        "id": "uuid",
        "username": "aelbamby",
        "first_name": "Ahmed",
        "last_name": "ElBamby"
      },
      "target_entity_type": "AnomalyEvent",
      "target_entity_id": "uuid",
      "details": {
        "reason": "False positive - student was stretching",
        "anomaly_tracking_id": "S-0012"
      },
      "ip_address": "192.168.1.50",
      "timestamp": "2026-02-27T10:17:00Z"
    }
  ]
}
```

**Action Types**: `login`, `logout`, `login_failed`, `account_create`, `account_edit`, `account_deactivate`, `role_create`, `role_edit`, `role_assign`, `exam_create`, `exam_edit`, `session_start`, `session_end`, `triage_acknowledge`, `triage_dismiss`, `triage_revert`, `recording_access`, `recording_download`, `config_change`, `anomaly_annotate`, `camera_add`, `camera_remove`, `export_session`

---

## Common Response Patterns

### Pagination

All list endpoints use cursor/offset pagination:
```json
{
  "count": 100,
  "next": "/api/v1/resource/?page=2",
  "previous": null,
  "results": []
}
```

### Error Format

All errors follow a consistent format:
```json
{
  "error": "Human-readable actionable message",
  "code": "MACHINE_READABLE_CODE",
  "details": {}
}
```

No raw stack traces, error codes, or technical jargon in user-facing errors (FR-036).

### HTTP Status Codes

| Code | Usage |
|------|-------|
| 200 | Successful read/update |
| 201 | Successful create |
| 202 | Accepted (async job queued, e.g., export) |
| 204 | Successful delete (no body) |
| 400 | Validation error |
| 401 | Not authenticated |
| 403 | Not authorized (insufficient role/permissions per FR-052) |
| 404 | Resource not found |
| 409 | Conflict (e.g., active session exists, first-write-wins) |
| 503 | Service unavailable (health check) |
