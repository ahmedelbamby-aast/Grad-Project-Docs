# API Contract: Camera REST Endpoints

**Branch**: `002-fix-rtsp-camera-feed` | **Date**: 2026-03-04  
**Base URL**: `/api/v1/cameras/`

---

## Authentication

All endpoints require `SessionAuthentication`. The session cookie (`sessionid`) must be present. CSRF token required for mutating requests (`X-CSRFToken` header from `csrftoken` cookie).

---

## Endpoints

### 1. List Cameras

```
GET /api/v1/cameras/
```

**Query Parameters**:
| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `status` | string | No | Filter by status: `connected`, `disconnected`, `reconnecting`, `error` |
| `page` | int | No | Pagination page number |

**Response** `200 OK`:
```json
{
  "count": 3,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "name": "Room 101 - Front",
      "connection_type": "rtsp",
      "connection_url": "rtsp://***:***@192.168.1.10/stream",
      "status": "connected",
      "error_detail": "",
      "resolution_width": 1920,
      "resolution_height": 1080,
      "frame_rate": 30.0,
      "room": { "id": "...", "name": "Room 101" },
      "added_by": { "id": "...", "username": "instructor1", "first_name": "Ahmed", "last_name": "Smith" },
      "created_at": "2026-03-01T10:00:00Z",
      "updated_at": "2026-03-04T12:00:00Z"
    }
  ]
}
```

**Notes**:
- `connection_url` is always masked for display (credentials replaced with `***:***`).
- Instructors see only their own cameras. Admins see all cameras.

---

### 2. Create Camera

```
POST /api/v1/cameras/
```

**Request Body**:
```json
{
  "name": "Room 101 - Front",
  "connection_type": "rtsp",
  "connection_url": "rtsp://admin:pass@192.168.1.10:554/stream",
  "room_id": "uuid-of-room-or-null"
}
```

**Validation**:
- `connection_url` must match `^rtsps?://` (FR-012).
- Per-instructor max 8 cameras (FR-017).
- Unique `connection_url` per instructor — checked via blind index (FR-019).

**Response** `201 Created`: Same shape as list item.

**Error Responses**:
- `400`: `{"connection_url": ["URL must start with rtsp:// or rtsps://"]}`
- `400`: `{"connection_url": ["You already have a camera with this URL."]}`
- `400`: `{"non_field_errors": ["Maximum number of cameras reached (8). Please disconnect an existing camera first."]}`

---

### 3. Update Camera

```
PATCH /api/v1/cameras/{id}/
```

**Request Body** (partial update):
```json
{
  "name": "Room 101 - Back Camera",
  "connection_url": "rtsp://admin:newpass@192.168.1.10:554/stream2",
  "room_id": "new-room-uuid"
}
```

**Validation**:
- If `connection_url` is changed and camera is `connected`, system auto-disconnects first (FR-020).
- New URL goes through same validation as create.

**Response** `200 OK`: Updated camera object.

---

### 4. Delete Camera

```
DELETE /api/v1/cameras/{id}/
```

**Behavior**:
- If camera is `connected`, system auto-disconnects (unregisters from go2rtc) before deleting (FR-022).
- All session data, detections, predictions, anomalies, and recordings are preserved.
- `ConnectionEvent` rows referencing this camera are preserved with `camera_id` set to null.

**Response** `204 No Content`.

---

### 5. Connect Camera

```
POST /api/v1/cameras/{id}/connect/
```

**Request Body**: None.

**Flow** (FR-001, FR-002, FR-004):
1. Validate RTSP URL (three-stage probe: TCP → DESCRIBE → frame decode).
2. Register stream with go2rtc (`PUT /api/streams`).
3. Create `StreamRegistration`.
4. Update camera status to `connected`.
5. Broadcast status via WebSocket.
6. Log `connect_success` event.

**Response** `200 OK`:
```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "name": "Room 101 - Front",
  "status": "connected",
  "error_detail": "",
  "connection_url": "rtsp://***:***@192.168.1.10/stream",
  ...
}
```

**Error Responses**:
- `502`: Network unreachable — `{"detail": "Camera Unreachable — Could not connect to the camera at the provided URL. Please verify the IP address and that the camera is powered on.", "error_code": "RTSP_NETWORK_ERROR", "stage": "tcp_connect"}`
- `502`: Auth failure — `{"detail": "Authentication Failed — The camera rejected the provided credentials.", "error_code": "RTSP_AUTH_ERROR", "stage": "rtsp_describe"}`
- `502`: No video — `{"detail": "No Video Data — The camera responded but is not producing video.", "error_code": "RTSP_NO_VIDEO", "stage": "frame_decode"}`
- `503`: go2rtc down — `{"detail": "Streaming Service Unavailable — The video streaming service is temporarily down.", "error_code": "RELAY_UNAVAILABLE", "stage": "stream_registration"}`
- `503`: go2rtc at capacity — `{"detail": "Streaming Service at Capacity — Please try again later.", "error_code": "RELAY_CAPACITY", "stage": "stream_registration"}`
- `409`: Already connected — `{"detail": "Camera is already connected.", "error_code": "ALREADY_CONNECTED"}`

**Error Response Shape**:
```json
{
  "detail": "Human-readable error message",
  "error_code": "MACHINE_READABLE_CODE",
  "stage": "tcp_connect | rtsp_describe | frame_decode | stream_registration"
}
```

---

### 6. Disconnect Camera

```
POST /api/v1/cameras/{id}/disconnect/
```

**Request Body**: None.

**Flow**:
1. Unregister stream from go2rtc (`DELETE /api/streams`).
2. Delete `StreamRegistration`.
3. Update camera status to `disconnected`.
4. Broadcast status via WebSocket.
5. Log `disconnect` event.

**Response** `200 OK`: Updated camera object with `status: "disconnected"`.

**Error Responses**:
- `409`: `{"detail": "Camera is already disconnected.", "error_code": "ALREADY_DISCONNECTED"}`

---

### 7. Connection Events (History)

```
GET /api/v1/cameras/{id}/events/
```

**Query Parameters**:
| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `page` | int | No | Pagination page |

**Response** `200 OK`:
```json
{
  "count": 42,
  "next": "...",
  "previous": null,
  "results": [
    {
      "id": 1,
      "camera_id": "a1b2c3d4-...",
      "event_type": "connect_success",
      "error_detail": "",
      "triggered_by": { "id": "...", "username": "instructor1", ... },
      "session_id": null,
      "created_at": "2026-03-04T12:00:00Z"
    }
  ]
}
```

**Access Control**:
- Instructors: see last 10 events for their own cameras only.
- Admins: see all events with full pagination and filtering.

---

## Admin Endpoints

### 8. Admin Camera Dashboard

```
GET /api/v1/admin/cameras/
```

**Permission**: Admin only.

**Query Parameters**:
| Param | Type | Description |
|-------|------|-------------|
| `status` | string | Filter by camera status |
| `instructor` | uuid | Filter by instructor (added_by) |
| `room` | uuid | Filter by room |
| `search` | string | Search by camera name |

**Response** `200 OK`: Paginated list of all cameras across all instructors (same shape as camera list, but unfiltered by ownership).

---

### 9. Admin Force-Disconnect

```
POST /api/v1/admin/cameras/{id}/force-disconnect/
```

**Permission**: Admin only.

**Behavior**: Same as disconnect but:
- Works on any instructor's camera.
- Logs `force_disconnect` event with admin as `triggered_by`.

**Response** `200 OK`: Updated camera object.

---

### 10. Admin Connect on Behalf

```
POST /api/v1/admin/cameras/{id}/connect/
```

**Permission**: Admin only.

**Behavior**: Same as regular connect but:
- Works on any instructor's camera.
- Logs `admin_connect` event with admin as `triggered_by`.

**Response** `200 OK`: Updated camera object.

---

### 11. Admin Relay Health

```
GET /api/v1/admin/cameras/relay-health/
```

**Permission**: Admin only.

**Response** `200 OK`:
```json
{
  "status": "healthy",
  "registered_streams": 5,
  "active_producers": 3,
  "active_consumers": 2,
  "last_check_at": "2026-03-04T12:00:15Z"
}
```

**`status` values**: `healthy` (go2rtc responsive), `degraded` (responsive but some streams unhealthy), `down` (go2rtc unreachable).
