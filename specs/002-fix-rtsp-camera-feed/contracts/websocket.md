# WebSocket Contract: Camera Status Channel

**Branch**: `002-fix-rtsp-camera-feed` | **Date**: 2026-03-04  
**Protocol**: Django Channels (AsyncJsonWebsocketConsumer over Redis channel layer)

---

## Connection

### Per-Camera Channel (existing, extended)

```
ws://{host}/ws/cameras/{camera_id}/
```

Subscribes to status updates for a **single camera**. Used on the Camera Feed page when viewing a specific camera tile.

### Global Camera Channel (new)

```
ws://{host}/ws/cameras/
```

Subscribes to status updates for **all cameras** owned by the connected user. Used on the Camera List page and Camera Feed grid page.

**Authentication**: Django session auth via `AuthMiddlewareStack` (existing ASGI middleware). Unauthenticated connections are rejected.

---

## Server → Client Messages

### camera.status

Broadcast whenever a camera's status changes (connect, disconnect, error, reconnecting).

```json
{
  "type": "camera.status",
  "camera_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "connected",
  "error_detail": "",
  "timestamp": "2026-03-04T12:00:00.000Z"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `type` | `string` | Always `"camera.status"` |
| `camera_id` | `string` (UUID) | Camera identifier |
| `status` | `string` | One of: `connected`, `disconnected`, `reconnecting`, `error` |
| `error_detail` | `string` | Error description when `status === "error"`, empty string otherwise |
| `timestamp` | `string` (ISO 8601) | Server timestamp of the status change |

**Trigger sources**:
- `CameraService.connect()` — status → `connected` or `error`
- `CameraService.disconnect()` — status → `disconnected`
- `CameraService.mark_error()` — status → `error`
- `CameraService.mark_reconnecting()` — status → `reconnecting`
- Health-check Celery task — status changes during auto-reconnection

**Delivery guarantee**: At-least-once via Redis PUB/SUB. Clients should be idempotent — receiving the same status twice is harmless.

---

### camera.relay_health (admin only)

Broadcast to admin clients when the go2rtc relay health changes.

```json
{
  "type": "camera.relay_health",
  "status": "healthy",
  "registered_streams": 5,
  "active_producers": 3,
  "timestamp": "2026-03-04T12:00:15.000Z"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `type` | `string` | Always `"camera.relay_health"` |
| `status` | `string` | One of: `healthy`, `degraded`, `down` |
| `registered_streams` | `number` | Total streams in go2rtc |
| `active_producers` | `number` | Streams with active RTSP connections |
| `timestamp` | `string` (ISO 8601) | Health-check timestamp |

---

## Client → Server Messages

### ping

Keep-alive ping (handled by existing `useWebSocket` hook with 30s interval).

```json
{
  "type": "ping"
}
```

Server responds with:
```json
{
  "type": "pong"
}
```

---

## Channel Groups

| Group Name | Pattern | Subscribers | Messages Received |
|------------|---------|-------------|-------------------|
| `camera_{camera_id}` | Per-camera | Feed page viewers of that camera | `camera.status` for that camera only |
| `cameras_all` | Global | List page viewers, feed grid viewers | `camera.status` for all cameras |

**Broadcasting logic** (in `CameraService._broadcast_status`):
- Every status change sends to **both** `camera_{camera_id}` and `cameras_all`.
- Cost: one extra Redis PUBLISH per status change — negligible.

---

## Frontend Integration

### useCameraStatus Hook (new)

```typescript
// Subscribes to camera status updates via WebSocket
function useCameraStatus(cameraId?: string): {
  statuses: Map<string, CameraStatusUpdate>;
  isConnected: boolean;
}
```

- If `cameraId` provided: connects to `ws/cameras/{cameraId}/` (per-camera).
- If `cameraId` omitted: connects to `ws/cameras/` (global).
- Updates `cameraStore` on each `camera.status` message.
- Uses existing `useWebSocket` hook for reconnection/keep-alive.

### Store Integration

```typescript
// cameraStore additions
interface CameraStoreState {
  // ... existing ...
  setCameraStatus(cameraId: string, status: CameraStatus, errorDetail: string): void;
}
```

The `setCameraStatus` action updates the camera in the store map, triggering re-renders of any component consuming that camera's status (badge, overlay, feed tile).

---

## Sequence: Camera Connect via WebSocket

```
Instructor Browser          Django REST API              Django Channels (Redis)        Other Browsers
       │                          │                              │                           │
       │ POST /cameras/{id}/      │                              │                           │
       │  connect/                │                              │                           │
       │─────────────────────────>│                              │                           │
       │                          │ CameraService.connect()      │                           │
       │                          │  → probe RTSP                │                           │
       │                          │  → PUT go2rtc                │                           │
       │                          │  → update DB status          │                           │
       │                          │  → broadcast_status()────────>│                           │
       │                          │                              │ group_send(camera_{id})   │
       │                          │                              │ group_send(cameras_all)   │
       │                          │                              │──────────────────────────>│
       │     200 OK               │                              │    camera.status msg      │
       │<─────────────────────────│                              │                           │
       │                          │                              │                           │
```
