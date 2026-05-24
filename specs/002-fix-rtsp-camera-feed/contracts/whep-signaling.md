# WHEP Signaling Contract: Camera Video Streaming

**Branch**: `002-fix-rtsp-camera-feed` | **Date**: 2026-03-04  
**Protocol**: WHEP (WebRTC-HTTP Egress Protocol) via go2rtc, proxied through nginx

---

## Endpoint

```
POST /whep/{camera_id}/
```

**Authentication**: nginx `auth_request` subrequest to `/api/v1/auth/me/`. Session cookie required. Unauthenticated requests receive `401 Unauthorized`.

---

## Request

```http
POST /whep/a1b2c3d4-e5f6-7890-abcd-ef1234567890/ HTTP/1.1
Host: example.com
Content-Type: application/sdp
Cookie: sessionid=abc123

v=0
o=- 0 0 IN IP4 127.0.0.1
s=-
t=0 0
m=video 9 UDP/TLS/RTP/SAVPF 96
a=rtpmap:96 H264/90000
a=recvonly
...
```

| Header | Value | Required |
|--------|-------|----------|
| `Content-Type` | `application/sdp` | Yes |
| `Cookie` | `sessionid=...` | Yes (for auth) |

**Body**: SDP offer from the browser's `RTCPeerConnection`.

---

## Response

### Success `200 OK`

```http
HTTP/1.1 200 OK
Content-Type: application/sdp

v=0
o=- 0 0 IN IP4 192.168.1.100
s=-
t=0 0
m=video 8555 UDP/TLS/RTP/SAVPF 96
a=rtpmap:96 H264/90000
a=sendonly
c=IN IP4 192.168.1.100
a=candidate:1 1 TCP 2130706431 192.168.1.100 8555 typ host
...
```

**Body**: SDP answer from go2rtc containing ICE candidates on port 8555.

### Error Responses

| Status | Condition | Frontend Action |
|--------|-----------|-----------------|
| `401` | Session cookie missing or expired | Show "Session expired — please log in again" |
| `403` | User doesn't have permission for this camera | Show "Access denied" |
| `404` | Stream not registered in go2rtc | Show "Camera not connected — please connect first" |
| `500` | go2rtc internal error | Retry (up to 3 attempts with 2s→4s→8s backoff) |
| `502` | go2rtc unreachable (nginx can't proxy) | Show "Streaming service unavailable" |

---

## nginx Proxy Configuration

```nginx
upstream go2rtc_api {
    server go2rtc:1984;
}

location ~ ^/whep/(?<camera_id>[0-9a-f-]+)/$ {
    auth_request /_auth_check;
    proxy_pass http://go2rtc_api/api/webrtc?src=camera_$camera_id;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header Content-Type $content_type;
    proxy_set_header Content-Length $content_length;
}

location = /_auth_check {
    internal;
    proxy_pass http://django_app/api/v1/auth/me/;
    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
    proxy_set_header X-Original-URI $request_uri;
    proxy_set_header Cookie $http_cookie;
}
```

**Key**: nginx rewrites `/whep/{camera_id}/` → `go2rtc:1984/api/webrtc?src=camera_{camera_id}`, inserting the `camera_` prefix.

---

## ICE Media Transport

After WHEP signaling completes, the browser establishes a direct WebRTC connection to go2rtc on port **8555** (TCP ICE candidates). This port is:

- Exposed directly in Docker Compose (`8555:8555/tcp`)
- NOT proxied through nginx (it's DTLS-encrypted media, not HTTP)
- Secured at the signaling layer (only authenticated users receive valid SDP answers)

---

## Frontend Integration

```typescript
// useWhepClient hook — simplified interface
function useWhepClient(cameraId: string): {
  videoRef: React.RefObject<HTMLVideoElement>;
  state: 'idle' | 'connecting' | 'connected' | 'reconnecting' | 'error' | 'disconnected';
  error: string | null;
  retryCount: number;
  connect: () => void;
  disconnect: () => void;
}
```

**WHEP URL construction**:
```typescript
// Same-origin, no VITE_GO2RTC_URL needed
const url = `/whep/${cameraId}/`;
```

**Retry logic** (FR-009):
- 3 attempts with exponential backoff: 2s → 4s → 8s (~14s total)
- During retries: show "Reconnecting… attempt N/3" overlay
- After exhaustion: show persistent error banner
