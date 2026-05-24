# Research Report: RTSP Camera Feed Fix

**Branch**: `002-fix-rtsp-camera-feed` | **Date**: 2026-03-04  
**Stack**: Django 5.1.5, Channels 4.2.2, DRF 3.15.2, PostgreSQL 16, React 19, go2rtc

**Companion documents**:
- [research-rtsp-probe.md](research-rtsp-probe.md) — Three-stage RTSP URL validation probe implementation

---

## Topic 0: go2rtc REST API for Stream Management

### 0.1 API Overview

All endpoints on **port 1984** (the `api.listen` port). Base URL: `http://go2rtc:1984` (Docker service name).

| Operation | Method | Endpoint | Notes |
|-----------|--------|----------|-------|
| Register stream | `PUT` | `/api/streams?name={name}&src={rtsp_url}` | Creates stream in memory + patches config. Returns 200 (empty body). |
| Update stream source | `PATCH` | `/api/streams?name={name}&src={new_url}` | Changes source for existing stream. |
| Delete stream | `DELETE` | `/api/streams?src={stream_name}` | Removes from memory + config. `src` = stream name, not RTSP URL. |
| List all streams | `GET` | `/api/streams` | Returns JSON object keyed by stream name. |
| Get one stream | `GET` | `/api/streams?src={stream_name}` | Returns filtered JSON. 404 if not found. |
| Snapshot probe | `GET` | `/api/frame.jpeg?src={stream_name}` | Forces RTSP connect + frame decode. Heavy but definitive health check. |

### 0.2 Stream Registration — Lazy Connection Model

**Critical**: `PUT /api/streams` does NOT connect to the RTSP source immediately. go2rtc uses lazy connections — the RTSP connection is established only when a consumer (WebRTC viewer, RTSP client) requests the stream. When all consumers disconnect, go2rtc may close the RTSP connection.

**Implication**: The `PUT` response cannot tell you if the RTSP URL is valid. You MUST validate the RTSP URL independently (see [research-rtsp-probe.md](research-rtsp-probe.md)) BEFORE calling `PUT`.

### 0.3 Stream Status Response Format

`GET /api/streams` returns:

```json
{
  "camera_abc123": {
    "producers": [
      {
        "url": "rtsp://admin:pass@192.168.1.10/stream",
        "medias": [...],
        "recv_bytes": 12345,
        "recv_packets": 100
      }
    ],
    "consumers": [
      {
        "url": "webrtc",
        "send_bytes": 9876,
        "send_packets": 80
      }
    ]
  }
}
```

| Condition | Meaning |
|-----------|---------|
| Stream name absent | Not registered (e.g., go2rtc restarted) |
| `producers: []` + `consumers: []` | Registered but idle (no viewers) |
| `producers` non-empty, `recv_bytes` increasing | RTSP source is producing video |
| `consumers` non-empty | Active WebRTC/RTSP viewers |

### 0.4 Health Check Strategy

- Poll `GET /api/streams` every 15 seconds from a Celery beat task.
- Compare registered stream names against DB cameras with `status=connected`.
- **Missing stream → re-register** (go2rtc restart recovery).
- **Stream registered but no consumers for >30s → leave it** (go2rtc will lazy-disconnect the RTSP source on its own).

### 0.5 Error Handling Matrix

| Scenario | go2rtc Response | Django Action |
|----------|-----------------|---------------|
| `PUT` with valid params | 200 (empty) | Success — stream registered |
| `PUT` with missing params | 400 | Map to internal error |
| Unreachable RTSP URL | 200 on `PUT` (no validation!) | Must pre-validate with RTSP probe |
| Bad RTSP credentials | 200 on `PUT` (lazy, no validation!) | Must pre-validate with RTSP probe |
| go2rtc service down | `ConnectionRefused` / timeout | `httpx.ConnectError` → "Streaming service unavailable" |
| go2rtc restarted | All streams lost from memory | Health-check detects missing, re-registers |
| Resource exhaustion | go2rtc crashes/OOM | Catch connection errors → "Streaming service at capacity" |

### 0.6 Naming Convention

Stream names follow `camera_{uuid}` where `{uuid}` is the `CameraSource.id` (UUID with hyphens). Used consistently across:
- go2rtc registration: `PUT /api/streams?name=camera_{uuid}&src={rtsp_url}`
- go2rtc deletion: `DELETE /api/streams?src=camera_{uuid}`
- WHEP signaling: `POST /api/webrtc?src=camera_{uuid}`
- nginx proxy: `/whep/{camera_id}/` → rewritten to `camera_{camera_id}`

### 0.7 Recommended HTTP Client

| Factor | `httpx` | `requests` |
|--------|---------|------------|
| Async support | Native `AsyncClient` | No |
| Connection pooling | Built-in | Built-in |
| Timeout granularity | connect, read, write, pool | Basic |
| Django Channels compat | Excellent | Poor |

**Decision**: Use `httpx` — sync client for Celery tasks, async client for potential future async views. Timeout: 5s for CRUD, 15s for snapshot probe.

## Topic 1: Encrypting RTSP URLs at Rest in PostgreSQL

### 1.1 Library Comparison

| Library | Last PyPI Release | Django 5.1 Support | Python 3.10+ | Notes |
|---|---|---|---|---|
| `django-encrypted-model-fields` | 2020 (0.6.5) | **No** — relies on `django.utils.encoding.force_text` removed in Django 4.0 | Partial | **Dead project.** Will crash on import with Django 5.x. |
| `django-fernet-fields` | 2018 (0.6) | **No** — unmaintained, uses old `from_db_value` signature | Partial | **Dead project.** Not compatible with Django 3+. |
| **Custom `EncryptedTextField`** | N/A | **Yes** — you control the field code | Yes | **Recommended.** Uses `cryptography.fernet.Fernet` directly. ~40 lines. No third-party risk. |

**Verdict**: Both third-party libraries are abandoned and incompatible with Django 5.1. A custom field using `cryptography` (already transitively installed via `channels`) is the correct approach. It's a trivial amount of code and avoids supply-chain risk.

### 1.2 Custom `EncryptedTextField` Implementation

```python
# backend/core/fields.py

import os
import base64

from cryptography.fernet import Fernet, InvalidToken
from django.db import models


def _get_fernet() -> Fernet:
    """Return a Fernet instance from the FIELD_ENCRYPTION_KEY env var.

    The key must be a URL-safe base64-encoded 32-byte key.
    Generate one with: python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
    """
    key = os.environ.get('FIELD_ENCRYPTION_KEY')
    if not key:
        raise ImproperlyConfigured(
            'FIELD_ENCRYPTION_KEY environment variable is not set. '
            'Generate one with: python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"'
        )
    return Fernet(key.encode())


class EncryptedTextField(models.TextField):
    """TextField that transparently encrypts on write and decrypts on read.

    Storage format: URL-safe base64 Fernet token (variable length, ~160+ chars
    for a typical RTSP URL). The underlying DB column is a regular TEXT column.
    """

    description = "Fernet-encrypted text field"

    def get_prep_value(self, value: str | None) -> str | None:
        """Encrypt before INSERT/UPDATE."""
        if value is None or value == '':
            return value
        f = _get_fernet()
        return f.encrypt(value.encode()).decode()  # returns base64 str

    def from_db_value(self, value: str | None, expression, connection) -> str | None:
        """Decrypt after SELECT."""
        if value is None or value == '':
            return value
        f = _get_fernet()
        try:
            return f.decrypt(value.encode()).decode()
        except InvalidToken:
            # Value is not encrypted (legacy data or corruption).
            # Log a warning and return raw value to avoid data loss.
            import logging
            logging.getLogger('exam_monitor').warning(
                'Failed to decrypt EncryptedTextField value — returning raw.'
            )
            return value

    def deconstruct(self):
        """Ensure migrations don't store the key."""
        name, path, args, kwargs = super().deconstruct()
        return name, 'core.fields.EncryptedTextField', args, kwargs
```

**Model usage** (in `apps/cameras/models.py`):

```python
from core.fields import EncryptedTextField

class CameraSource(models.Model):
    # ...existing fields...
    connection_url = EncryptedTextField()  # was: models.TextField()
```

**Migration**: Django will generate an `AlterField` migration. Since the column type stays `TEXT`, no data transformation SQL is needed. However, **existing plaintext rows must be encrypted** via a data migration:

```python
# migration RunPython forward function
from cryptography.fernet import Fernet
import os

def encrypt_existing_urls(apps, schema_editor):
    CameraSource = apps.get_model('cameras', 'CameraSource')
    f = Fernet(os.environ['FIELD_ENCRYPTION_KEY'].encode())
    for cam in CameraSource.objects.all():
        if cam.connection_url and not cam.connection_url.startswith('gAAAAA'):
            cam.connection_url = f.encrypt(cam.connection_url.encode()).decode()
            cam.save(update_fields=['connection_url'])
```

> **Note**: Fernet tokens always start with `gAAAAA` (base64 of version byte + timestamp). This is a safe heuristic to detect already-encrypted values.

**Env var setup** (`.env`):

```bash
# Generate with: python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
FIELD_ENCRYPTION_KEY=your-base64-encoded-32-byte-key-here
```

### 1.3 Key Rotation

Fernet supports **multi-key decryption** via `MultiFernet`:

```python
from cryptography.fernet import Fernet, MultiFernet

def _get_fernet() -> Fernet | MultiFernet:
    """Support key rotation. Keys are comma-separated, newest first."""
    keys_raw = os.environ.get('FIELD_ENCRYPTION_KEY', '')
    keys = [k.strip() for k in keys_raw.split(',') if k.strip()]
    if not keys:
        raise ImproperlyConfigured('FIELD_ENCRYPTION_KEY is not set.')
    if len(keys) == 1:
        return Fernet(keys[0].encode())
    # MultiFernet: encrypts with keys[0], decrypts trying all keys in order
    return MultiFernet([Fernet(k.encode()) for k in keys])
```

**Rotation procedure**:
1. Generate new key: `python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"`
2. Prepend to env var: `FIELD_ENCRYPTION_KEY=new_key,old_key`
3. Deploy — new writes use `new_key`, reads try both keys.
4. Run a management command to re-encrypt all rows with the new key (`MultiFernet.rotate()`).
5. After all rows are re-encrypted, remove `old_key` from the env var.

```python
# management command: python manage.py rotate_encryption_keys
from cryptography.fernet import Fernet, MultiFernet

multi = MultiFernet([Fernet(k) for k in keys])
for cam in CameraSource.objects.all():
    raw = cam.connection_url.encode()
    cam.connection_url = multi.rotate(raw).decode()  # re-encrypts with keys[0]
    cam.save(update_fields=['connection_url'])
```

### 1.4 Querying Encrypted Fields

**You cannot filter/search on Fernet-encrypted fields** — the ciphertext is non-deterministic (each encryption produces a different token due to the embedded timestamp).

**If you need lookups** (e.g., FR-019 uniqueness check: "does this instructor already have this URL?"), use a **blind index** — a keyed hash stored alongside the ciphertext:

```python
import hashlib, hmac, os

def _url_hash(url: str) -> str:
    """Deterministic HMAC-SHA256 hash for blind-index lookups."""
    key = os.environ['FIELD_ENCRYPTION_KEY'].split(',')[0].encode()
    return hmac.new(key, url.encode(), hashlib.sha256).hexdigest()

class CameraSource(models.Model):
    connection_url = EncryptedTextField()        # encrypted, non-queryable
    connection_url_hash = models.CharField(       # blind index, queryable
        max_length=64, db_index=True, blank=True
    )

    def save(self, *args, **kwargs):
        if self.connection_url:
            self.connection_url_hash = _url_hash(self.connection_url)
        super().save(*args, **kwargs)
```

**Uniqueness check** (FR-019):
```python
# In CameraService or serializer validation
exists = CameraSource.objects.filter(
    added_by=instructor,
    connection_url_hash=_url_hash(submitted_url),
).exclude(pk=camera_pk).exists()
if exists:
    raise ValidationError("You already have a camera with this URL.")
```

### 1.5 URL Masking for Display

Pattern for masking credentials in RTSP URLs (FR-015):

```python
# backend/apps/cameras/utils.py
import re
from urllib.parse import urlparse, urlunparse

_RTSP_CRED_RE = re.compile(r'^(rtsps?)://([^@]+)@(.+)$', re.IGNORECASE)

def mask_rtsp_url(url: str) -> str:
    """Mask credentials in an RTSP URL for display/logging.

    rtsp://admin:pass@192.168.1.10/stream → rtsp://***:***@192.168.1.10/stream
    rtsp://192.168.1.10/stream             → rtsp://192.168.1.10/stream (no creds)
    """
    match = _RTSP_CRED_RE.match(url)
    if not match:
        return url
    scheme, _creds, host_and_path = match.groups()
    return f'{scheme}://***:***@{host_and_path}'
```

**Usage in serializers** (read-only display):
```python
class CameraSourceSerializer(serializers.ModelSerializer):
    masked_url = serializers.SerializerMethodField()

    def get_masked_url(self, obj):
        return mask_rtsp_url(obj.connection_url)
```

**Usage in logging** (auto-redact via `core/logger.py` or directly):
```python
logger.info("Connecting camera %s to %s", camera.id, mask_rtsp_url(camera.connection_url))
```

---

## Topic 2: Broadcasting Camera Status via Django Channels

### 2.1 Sending to a Channels Group from Synchronous Code

The existing `CameraStatusConsumer` in [backend/apps/cameras/consumers.py](backend/apps/cameras/consumers.py) handles the `camera_status` event type and sends the payload to connected WebSocket clients. To **trigger** this from the service layer (sync context — a DRF view, a Celery task, or a model save), use:

```python
from channels.layers import get_channel_layer
from asgiref.sync import async_to_sync

channel_layer = get_channel_layer()

def broadcast_camera_status(camera_id: str, payload: dict) -> None:
    """Send a status update to all WebSocket clients watching this camera.

    Works from synchronous Django code (views, services, Celery tasks).
    """
    async_to_sync(channel_layer.group_send)(
        f'camera_{camera_id}',
        {
            'type': 'camera_status',   # maps to consumer method camera_status()
            'payload': payload,
        }
    )
```

**How it works**:
- `get_channel_layer()` returns the Redis-backed channel layer configured in `CHANNEL_LAYERS` settings.
- `async_to_sync` wraps the async `group_send` coroutine so it can be called from sync code.
- The `type` field `'camera_status'` is looked up by Channels as a method name on the consumer (`camera_status` → `async def camera_status(self, event)`).
- `payload` is the dict that the consumer forwards to the WebSocket client.

**From async code** (e.g., inside another consumer or async view):
```python
await channel_layer.group_send(
    f'camera_{camera_id}',
    {'type': 'camera_status', 'payload': payload}
)
```

### 2.2 Message Payload Format

The existing consumer in [consumers.py](backend/apps/cameras/consumers.py#L15) does:

```python
async def camera_status(self, event):
    await self.send_json({'type': 'camera.status', **event['payload']})
```

So the **payload** dict is spread into the outbound JSON. The recommended payload shape:

```python
payload = {
    'camera_id': str(camera.id),            # UUID as string
    'status': camera.status,                 # 'connected' | 'disconnected' | 'reconnecting' | 'error'
    'error_detail': camera.error_detail,     # str | None — human-readable error message
    'timestamp': now().isoformat(),          # ISO 8601
}
```

The client receives:
```json
{
  "type": "camera.status",
  "camera_id": "a1b2c3d4-...",
  "status": "connected",
  "error_detail": null,
  "timestamp": "2026-03-04T12:00:00Z"
}
```

### 2.3 Group Strategy: Per-Camera vs. Global

| Approach | Group Name | Use Case | Recommendation |
|---|---|---|---|
| **Per-camera** (existing) | `camera_{camera_id}` | Camera feed page — client subscribes to one camera | **Keep.** Already implemented. Used for the `CameraFeed` tile. |
| **Global all-cameras** | `cameras_all` | Camera list page — client needs status of all cameras | **Add.** Without this, the list page must open N WebSocket connections (one per camera) which is wasteful. |

**Implementation**: Modify `CameraStatusConsumer` to support both modes via a URL route parameter:

```python
# routing.py
websocket_urlpatterns = [
    re_path(r'^ws/cameras/(?P<camera_id>[0-9a-f-]+)/$', CameraStatusConsumer.as_asgi()),
    re_path(r'^ws/cameras/$', CameraStatusConsumer.as_asgi()),  # no camera_id → global group
]
```

```python
# consumers.py
class CameraStatusConsumer(AsyncJsonWebsocketConsumer):
    async def connect(self):
        self.camera_id = self.scope['url_route']['kwargs'].get('camera_id')
        if self.camera_id:
            self.group_name = f'camera_{self.camera_id}'
        else:
            self.group_name = 'cameras_all'
        await self.channel_layer.group_add(self.group_name, self.channel_name)
        await self.accept()

    async def disconnect(self, code):
        await self.channel_layer.group_discard(self.group_name, self.channel_name)

    async def camera_status(self, event):
        await self.send_json({'type': 'camera.status', **event['payload']})
```

**Broadcasting**: The service layer sends to **both** groups:

```python
def broadcast_camera_status(camera_id: str, payload: dict) -> None:
    group_send = async_to_sync(channel_layer.group_send)
    msg = {'type': 'camera_status', 'payload': payload}

    # Per-camera group (for feed page viewers)
    group_send(f'camera_{camera_id}', msg)

    # Global group (for list page viewers)
    group_send('cameras_all', msg)
```

This costs one extra Redis `PUBLISH` per status change — negligible overhead.

### 2.4 Where to Put Broadcast Logic

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| **Service layer** (in `CameraService` methods) | Explicit, testable, co-located with business logic | Caller must remember to call it | **Recommended.** |
| `post_save` signal on `CameraSource` | Automatic on any save | Fires on every save (even non-status changes), hard to test, implicit coupling | Not recommended. |
| In the DRF view | Simple | Duplicated if multiple views change status (admin, health-check task, API) | Not recommended. |

**Recommended pattern** — broadcast is called inside `CameraService` after each state transition:

```python
# backend/apps/cameras/services.py

from channels.layers import get_channel_layer
from asgiref.sync import async_to_sync
from django.utils.timezone import now

channel_layer = get_channel_layer()


class CameraService:
    @staticmethod
    def _broadcast_status(camera: CameraSource) -> None:
        """Broadcast current camera status to all WebSocket subscribers."""
        payload = {
            'camera_id': str(camera.id),
            'status': camera.status,
            'error_detail': getattr(camera, 'error_detail', None),
            'timestamp': now().isoformat(),
        }
        msg = {'type': 'camera_status', 'payload': payload}
        send = async_to_sync(channel_layer.group_send)
        send(f'camera_{camera.id}', msg)
        send('cameras_all', msg)

    @staticmethod
    def connect(camera: CameraSource) -> CameraSource:
        # ... validation, go2rtc registration ...
        camera.status = CameraSource.Status.CONNECTED
        camera.error_detail = ''
        camera.save(update_fields=['status', 'error_detail', 'updated_at'])
        CameraService._broadcast_status(camera)
        return camera

    @staticmethod
    def disconnect(camera: CameraSource) -> CameraSource:
        # ... go2rtc unregistration ...
        camera.status = CameraSource.Status.DISCONNECTED
        camera.save(update_fields=['status', 'updated_at'])
        CameraService._broadcast_status(camera)
        return camera

    @staticmethod
    def mark_error(camera: CameraSource, error_detail: str) -> CameraSource:
        camera.status = CameraSource.Status.ERROR
        camera.error_detail = error_detail
        camera.save(update_fields=['status', 'error_detail', 'updated_at'])
        CameraService._broadcast_status(camera)
        return camera
```

This ensures every status change — from views, Celery tasks, or admin actions — triggers a broadcast, because they all go through `CameraService`.

---

## Topic 3: Frontend WHEP Client Through Authenticated Proxy

### 3.1 nginx Proxy Configuration — Current Problem

The current [nginx.conf](nginx.conf) has:

```nginx
upstream go2rtc_app {
    server go2rtc:8555;  # ← WRONG: 8555 is WebRTC ICE/TCP, not the HTTP API
}

location ~ ^/whep/(?<camera_id>[^/]+)/$ {
    auth_request /_auth_check;
    proxy_pass http://go2rtc_app/whep/$camera_id/;  # ← Goes to port 8555 (WebRTC), not 1984 (HTTP API)
}
```

From [go2rtc.yaml](go2rtc.yaml):
- Port **1984** = HTTP API (serves `/api/webrtc`, `/api/streams`, etc.)
- Port **8555** = WebRTC ICE/TCP transport (for media packets, not HTTP)

The WHEP signaling endpoint is `POST /api/webrtc?src={stream_name}` on port **1984**. The nginx config must proxy to port 1984, not 8555.

**Corrected nginx.conf**:

```nginx
events {}

http {
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    upstream django_app {
        server backend:8000;
    }

    upstream go2rtc_api {
        server go2rtc:1984;    # HTTP API (WHEP signaling, stream management)
    }

    server {
        listen 80;
        server_name _;

        # ── Django REST API ────────────────────────────────
        location /api/v1/ {
            proxy_pass http://django_app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # ── WHEP signaling (auth-gated) ───────────────────
        # Frontend POSTs SDP offer here; nginx checks Django session auth
        # then proxies to go2rtc's /api/webrtc endpoint on port 1984.
        location ~ ^/whep/(?<camera_id>[0-9a-f-]+)/$ {
            auth_request /_auth_check;

            # Rewrite to go2rtc's actual API endpoint
            proxy_pass http://go2rtc_api/api/webrtc?src=camera_$camera_id;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;

            # Required: forward the request body (SDP offer)
            proxy_set_header Content-Type $content_type;
            proxy_set_header Content-Length $content_length;

            # go2rtc returns SDP answer as text/plain or application/sdp
            # Pass it through unchanged
        }

        # ── WebSocket (Django Channels) ────────────────────
        location /ws/ {
            proxy_pass http://django_app;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            proxy_set_header Host $host;
        }

        # ── Internal auth check subrequest ─────────────────
        location = /_auth_check {
            internal;
            proxy_pass http://django_app/api/v1/auth/me/;
            proxy_pass_request_body off;
            proxy_set_header Content-Length "";
            proxy_set_header X-Original-URI $request_uri;
            proxy_set_header Cookie $http_cookie;
        }
    }
}
```

**Key changes**:
1. Upstream renamed to `go2rtc_api` pointing to **port 1984** (not 8555).
2. `proxy_pass` rewrites to `/api/webrtc?src=camera_$camera_id` — the actual go2rtc WHEP signaling endpoint.
3. `Content-Type` and `Content-Length` headers are forwarded (nginx strips them by default after `auth_request`).

### 3.2 Frontend WHEP Client Changes

Currently, [useWhepClient.ts](frontend/src/hooks/useWhepClient.ts) POSTs directly to `http://localhost:1984/api/webrtc?src={streamName}`. It must instead POST to the nginx proxy route.

**Before** (current):
```typescript
const GO2RTC_URL = (import.meta.env.VITE_GO2RTC_URL as string | undefined)?.replace(/\/$/, '')
  ?? 'http://localhost:1984';

// ...
const url = `${GO2RTC_URL}/api/webrtc?src=${encodeURIComponent(streamName)}`;
const res = await fetch(url, {
  method: 'POST',
  headers: { 'Content-Type': 'application/sdp' },
  body: offer.sdp,
});
```

**After** (proxied through nginx):
```typescript
// The WHEP signaling now goes through nginx on the same origin.
// No VITE_GO2RTC_URL needed — same-origin relative path.
// streamName is "camera_{uuid}", extract camera_id from it.
const cameraId = streamName.replace(/^camera_/, '');
const url = `/whep/${cameraId}/`;

const res = await fetch(url, {
  method: 'POST',
  headers: { 'Content-Type': 'application/sdp' },
  body: offer.sdp,
  credentials: 'same-origin',  // sends session cookie for auth_request
});
```

**Summary of changes in `useWhepClient.ts`**:
1. Remove `VITE_GO2RTC_URL` env var usage entirely.
2. Build URL as `/whep/${cameraId}/` (relative, same-origin).
3. Add `credentials: 'same-origin'` to the fetch call so the session cookie is sent.
4. Handle 401/403 responses (auth_request rejection) with a specific error message.

### 3.3 CORS Implications

| Scenario | CORS Issue? | Explanation |
|---|---|---|
| **Current** (direct to `localhost:1984`) | **Yes** — cross-origin request from `:80` to `:1984`. Currently works only because go2rtc defaults to permissive CORS (`*`). | go2rtc sets `Access-Control-Allow-Origin: *` by default, but this prevents sending cookies (`withCredentials` + wildcard origin is forbidden by browsers). |
| **After** (proxied through nginx on same origin) | **No** — request goes to `/whep/{id}/` on the same origin (`:80`). | Same-origin request. No CORS preflight. Session cookie is sent automatically. |

**After the change, CORS is a non-issue.** The WHEP signaling request is same-origin (both the React app and the `/whep/` route are served on port 80 via nginx). The browser sends the session cookie automatically. No `Access-Control-*` headers are needed.

### 3.4 ICE Candidate Transport — Port 8555

**Question**: After WHEP signaling completes through nginx, the browser and go2rtc negotiate a direct WebRTC media connection. Port 8555 is the ICE/TCP candidate port in go2rtc. Does this need nginx proxying?

**Answer**: **No — port 8555 does NOT go through nginx.** Here's why:

1. **WHEP signaling** (SDP offer/answer exchange) goes through nginx → this is the auth-gated HTTP POST we just configured.

2. **ICE connectivity** happens after signaling, directly between the browser and go2rtc. The SDP answer from go2rtc includes ICE candidates with go2rtc's IP and port 8555. The browser connects directly to this endpoint for media transport.

3. **Port 8555 must be exposed to the browser**, but it does NOT need nginx:
   - In `docker-compose.dev.yml`, it's already exposed: `"8555:8555/tcp"` ✓
   - go2rtc's WebRTC config in `go2rtc.yaml` lists `listen: ":8555"` ✓
   - The browser's ICE agent connects directly to `<host>:8555` for the WebRTC media stream

4. **Security consideration**: Port 8555 handles only WebRTC DTLS-encrypted media transport. It does not serve HTTP. An attacker cannot access camera streams through this port without completing a legitimate WHEP signaling exchange first (which is auth-gated through nginx). The WebRTC session is protected by the DTLS handshake and the SDP offer/answer credentials exchanged during signaling.

**Production note**: In production behind NAT/firewall:
- Port 8555 must be reachable from the client browser (open in firewall/security group).
- Alternatively, configure go2rtc to use a TURN server for NAT traversal.
- The `ICE_SERVERS` config in the frontend should include a TURN server for production:

```typescript
const ICE_SERVERS: RTCIceServer[] = [
  { urls: 'stun:stun.l.google.com:19302' },
  // Production: add TURN server for NAT traversal
  // { urls: 'turn:turn.example.com:3478', username: '...', credential: '...' },
];
```

### 3.5 Full Data Flow (After Fix)

```
Browser                   nginx (:80)              Django (:8000)         go2rtc (:1984)        go2rtc (:8555)
  │                          │                         │                      │                      │
  │ POST /whep/{id}/         │                         │                      │                      │
  │ Cookie: sessionid=xxx    │                         │                      │                      │
  │ Body: SDP offer          │                         │                      │                      │
  │─────────────────────────>│                         │                      │                      │
  │                          │ GET /api/v1/auth/me/    │                      │                      │
  │                          │ Cookie: sessionid=xxx   │                      │                      │
  │                          │────────────────────────>│                      │                      │
  │                          │     200 OK (authed)     │                      │                      │
  │                          │<────────────────────────│                      │                      │
  │                          │                         │                      │                      │
  │                          │ POST /api/webrtc?src=camera_{id}              │                      │
  │                          │ Body: SDP offer                               │                      │
  │                          │──────────────────────────────────────────────>│                      │
  │                          │          200 OK — SDP answer                  │                      │
  │                          │<──────────────────────────────────────────────│                      │
  │     200 OK               │                         │                      │                      │
  │     Body: SDP answer     │                         │                      │                      │
  │<─────────────────────────│                         │                      │                      │
  │                          │                         │                      │                      │
  │ ICE: DTLS + SRTP media traffic (direct, not through nginx)              │                      │
  │────────────────────────────────────────────────────────────────────────────────────────────────>│
  │<────────────────────────────────────────────────────────────────────────────────────────────────│
  │ (live video stream)      │                         │                      │                      │
```

---

## Summary of Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Encryption library | Custom `EncryptedTextField` using `cryptography.fernet.Fernet` | Third-party libraries are dead. Custom field is ~40 lines. |
| Encryption key storage | `FIELD_ENCRYPTION_KEY` env var | Security workflow §2 compliance. |
| Key rotation | `MultiFernet` with comma-separated keys | Supported natively, zero-downtime rotation. |
| Encrypted field querying | HMAC-SHA256 blind index (`connection_url_hash`) | Fernet is non-deterministic; blind index enables uniqueness checks (FR-019). |
| URL masking | Regex-based `mask_rtsp_url()` utility | Used in serializers, logs, error messages (FR-015). |
| WS broadcast source | Service layer (`CameraService._broadcast_status`) | Explicit, testable, covers all code paths (views, tasks, admin). |
| WS group strategy | Per-camera `camera_{id}` + global `cameras_all` | Per-camera for feed page, global for list page. |
| WS message format | `{type: 'camera.status', camera_id, status, error_detail, timestamp}` | Matches existing consumer pattern. |
| nginx WHEP proxy target | `go2rtc:1984` (HTTP API) not `go2rtc:8555` (WebRTC transport) | 8555 is ICE/TCP, not HTTP. WHEP signaling is HTTP. |
| Frontend WHEP URL | `/whep/${cameraId}/` (same-origin relative) | Eliminates CORS, sends session cookie automatically. |
| ICE/media port 8555 | Exposed directly, not through nginx | WebRTC media is DTLS-encrypted; auth is enforced at signaling layer. |
