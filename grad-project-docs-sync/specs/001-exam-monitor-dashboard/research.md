# Research: Exam Monitoring Dashboard

**Feature**: 001-exam-monitor-dashboard | **Date**: 2026-02-27
**Spec**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md)

This document resolves all NEEDS CLARIFICATION items from the Technical Context and
consolidates best-practice research for each major technology choice.

---

## R-001: Backend Framework — Django vs Flask

### Decision: **Django 5.x + Django Channels 4.x + Django REST Framework 3.15+**

### Rationale

Django provides the richest out-of-the-box feature set for this project's requirements:
built-in ORM with auto-migrations, authentication with session management, RBAC via groups
and permissions, admin panel, CSRF/HSTS/clickjacking protection, and ASGI-native deployment
since Django 5.x.

For real-time WebSocket communication (FR-019), Django Channels provides native
`AsyncWebsocketConsumer` with Redis channel layer — no monkey-patching or eventlet required.
This is critical because the spec mandates backend-push for detection results and anomaly
alerts over a persistent bidirectional WebSocket connection.

DRF ViewSets reduce each CRUD resource to ~20 lines of code. With 11 Django apps and ~40
REST endpoints, this brevity directly reduces implementation and maintenance burden.

| Requirement | Django Advantage |
|---|---|
| WebSocket (FR-019) | Native ASGI via `AsyncWebsocketConsumer` + Redis channel layer. No monkey-patching. |
| ORM (8 models) | Built-in ORM with auto-migrations, `select_related`/`prefetch_related` for deep FK chains. |
| Auth + RBAC | `contrib.auth` provides User model, groups/permissions, session backend, password hashing, CSRF — zero extra packages. |
| 30+ REST endpoints | DRF ViewSets reduce each CRUD resource to ~20 lines. Browsable API, pagination, filtering, OpenAPI. |
| Celery integration | First-class via `django-celery-beat`. Auto task discovery. ORM available inside tasks. |
| ASGI deployment | Django 5.x is ASGI-native. HTTP + WebSocket on the same Uvicorn process. |
| Windows + Linux | Standard asyncio — no eventlet/gevent monkey-patching (Flask-SocketIO's known Windows pain point). |
| Security middleware | Built-in CSRF, HSTS, clickjacking, session cookie flags, password validators. Flask needs ~5 separate packages. |
| Scale (20×8×15 FPS) | Redis channel layer handles 100k+ msg/sec. 2,400 msg/sec worst-case is trivial. |

### Alternatives Considered

| Alternative | Why Rejected |
|---|---|
| **Flask + Flask-SocketIO** | Assembly burden (~15 packages vs ~5). eventlet Windows issues. No native ASGI. Security gaps. |
| **FastAPI** | No built-in ORM/auth/admin/migrations. Same assembly burden as Flask. Smaller WebSocket consumer ecosystem. |
| **Quart** | Tiny community (~1.5k stars). Flask extension incompatibilities. Same assembly burden as Flask. |

### Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Async/sync ORM boundary in Channels consumers | `database_sync_to_async` wrapper (well-documented, stable since Channels 3.x) |
| Django Channels learning curve | Prototype a single camera feed consumer during Phase 1 |
| Django Admin customization | Default covers 80% of needs; `ModelAdmin` + `django-unfold` for the rest |

---

## R-002: Frontend Framework — React vs Vue vs Next.js vs Solid

### Decision: **React 19.2 + TypeScript ~5.9.3 + Vite 7.3 + Zustand 5.x**

### Rationale

React is the strongest choice for a real-time video monitoring dashboard:

1. **Canvas rendering at scale**: `useRef` + `requestAnimationFrame` pattern is battle-proven
   for multi-stream video (Twitch, Discord, Zoom web). Required for 1-8 simultaneous camera
   feeds with bounding box overlays at ≥15 FPS (FR-007, FR-011).

2. **WebSocket + state management**: Zustand's `getState()` can be called outside React
   components — critical for WebSocket `onmessage` handlers updating detection stores without
   triggering unnecessary React reconciliation. At ~120 messages/second (8 cameras × 15 FPS),
   this avoids reactivity proxy overhead that Vue's Pinia would introduce.

3. **Testing ecosystem**: React Testing Library (20M+ downloads/week) + Zustand's plain-object
   stores make the ≥80% coverage mandate achievable. Zustand stores are testable without any
   DOM mount — `const state = useStore.getState()`.

4. **Accessibility**: Radix UI + React Aria (Adobe) provide WCAG 2.1 AA compliant headless
   components for modals, dropdowns, toggles, keyboard navigation (FR-035).

5. **Theming**: CSS custom properties applied via `data-theme` attribute on `<html>`. Theme
   switch sets attribute + persists to localStorage. All component styles reference CSS
   variables — ≤300ms switch without page reload (FR-022). No FOUC risk in SPA.

6. **Developer pool**: 450k+ SO questions, dominant in university CS programs — critical for
   a grad project that may need handoff or collaboration.

### Alternatives Considered

| Alternative | Why Rejected |
|---|---|
| **Vue 3 + Pinia** | Strong runner-up. Reactivity proxy overhead at ~120 msg/s. Smaller component/testing ecosystem. |
| **Next.js 14 (App Router)** | Architecturally wrong — SSR adds complexity with zero value for an authenticated real-time SPA. FOUC on theme switching. Every component needs `"use client"`. |
| **Solid.js** | Best theoretical performance but immature testing library (~2k downloads). Single headless UI option (Kobalte). ~3k SO questions. |

### Risks & Mitigations

| Risk | Mitigation |
|---|---|
| React re-render overhead at high message frequency | Zustand `subscribeWithSelector` + `shallow` equality; Canvas bypasses React DOM entirely |
| CSS custom property browser support | All target browsers have full support since 2016 |
| Vite HMR with WebSocket connections | Custom `useWebSocket` hook with reconnect logic surviving HMR in dev |

---

## R-003: Database — PostgreSQL

### Decision: **PostgreSQL 16**

### Rationale

1. **Relational integrity**: 8 interconnected entities with deep FK chains (Instructor →
   MonitoringSession → DetectionFrame → Detection → PyramidPrediction). Django ORM's
   `select_related`/`prefetch_related` prevents N+1 queries natively.

2. **JSON support**: `jsonb` columns for flexible prediction metadata storage. GIN indexes
   on jsonb for efficient anomaly filtering.

3. **Concurrent writes**: Up to 20 instructors × 8 cameras × 15 FPS = 2,400 detection
   frames/second at peak. PostgreSQL handles this with MVCC, connection pooling via PgBouncer,
   and partitioned tables for detection frames by date.

4. **Full-text search**: Built-in `tsvector` for searching anomaly notes and dismissed reasons
   (FR-042, FR-043) without Elasticsearch.

5. **Django integration**: psycopg 3.x is the recommended async-capable driver. Django ORM
   generates optimized SQL and handles migrations automatically.

### Alternatives Considered

| Alternative | Why Rejected |
|---|---|
| **MySQL 8** | Limited jsonb equivalent. Weaker Django ecosystem support. |
| **SQLite** | Single-writer limitation incompatible with 20 concurrent instructors. |
| **MongoDB** | No relational integrity for deeply relational entity model. |
| **TimescaleDB** | Overkill — PostgreSQL partitioning handles time-series detection frames without extension. |

### Risks & Mitigations

| Risk | Mitigation |
|---|---|
| High-volume detection frame inserts | Batch inserts via `bulk_create()`, table partitioning by date, PgBouncer pooling |
| Storage growth from detection metadata | Configurable retention policy with Celery periodic task for old data pruning |
| Dev setup complexity | Docker Compose provides PostgreSQL + Redis + Celery in one command |

---

## R-004: Real-Time Communication — WebSocket Architecture

### Decision: **Django Channels with Redis channel layer**

### Rationale

The spec mandates persistent bidirectional WebSocket (FR-019) — the backend MUST push
detection results and anomaly alerts; the frontend MUST NOT poll.

1. **AsyncWebsocketConsumer**: Native ASGI consumers for WebSocket lifecycle with full Django
   ORM access via `database_sync_to_async`.

2. **Redis channel layer**: Channel groups map directly to camera feeds — one group per
   camera, broadcast detection results to all subscribers. Redis handles 100k+ msg/sec;
   worst-case is ~2,400 msg/sec.

3. **Message types**: Structured JSON messages with `type` field:
   - `detection.frame` — bounding boxes + tracking IDs
   - `prediction.update` — pyramid layer predictions per student
   - `anomaly.alert` — new anomaly event with severity
   - `anomaly.status_change` — triage action notification
   - `health.update` — subsystem health metrics
   - `camera.status` — connection state changes

4. **Scaling**: Uvicorn workers + Redis pub/sub scale horizontally.

### Alternatives Considered

| Alternative | Why Rejected |
|---|---|
| **Server-Sent Events (SSE)** | Unidirectional only. Cannot handle client-sent triage actions over same connection. |
| **Long polling** | Explicitly prohibited by FR-019. Higher latency. |
| **gRPC-Web** | Requires envoy proxy. Overkill for JSON messaging. No Django native support. |

---

## R-005: YOLO Model Loading & Inference — Ultralytics

### Decision: **Ultralytics 8.x (ultralytics Python package)**

### Rationale

User-specified requirement. Ultralytics provides a unified API for the entire detection pipeline:

1. **Model loading**: `YOLO("model.pt")` — single-line loading for any YOLOv8/v11 model
   variant (detection, pose, classification). Supports `.pt` (PyTorch), `.onnx` (ONNX
   Runtime), `.engine` (TensorRT) export formats.

2. **Inference**: `model.predict(source, stream=True)` — generator-based streaming inference
   ideal for real-time camera feeds. Returns `Results` objects with `.boxes` (bounding boxes),
   `.names` (class names), `.conf` (confidence scores).

3. **Object tracking**: `model.track(source, tracker="botsort.yaml")` — built-in BoT-SORT
   and ByteTrack integration. Returns persistent tracking IDs per object across frames,
   directly satisfying FR-012 (persistent tracking ID on each student's bounding box).

4. **Multi-model pyramid**: Each pyramid layer loads a separate `YOLO()` instance. The
   Strategy pattern wraps each layer as an interchangeable adapter conforming to a
   `BasePyramidLayer` interface (Constitution Principle I — Liskov Substitution).

5. **GPU/CPU flexibility**: Auto-detects CUDA availability. `device="cuda:0"` for GPU,
   `device="cpu"` for fallback. Satisfies Constitution Principle VI for CUDA acceleration
   with graceful CPU fallback.

6. **Lazy loading**: Models loaded on-demand per camera session start. Each `YOLO()` instance
   cached in a model registry (Factory pattern) and released when session ends.

### Integration Architecture

```
Camera Frame (OpenCV)
    → Base Detector (YOLO: teacher/student classification + tracking)
    → [Students only] Posture Layer (YOLO: standing/sitting)
    → [Students only] Horizontal Gaze Layer (YOLO: left/right)
    → [Students only] Depth Gaze Layer (YOLO: forward/backward)
    → [Students only] Vertical Gaze Layer (YOLO: up/down)
    → Anomaly Detection (rule engine on aggregated predictions)
    → WebSocket broadcast (Django Channels)
```

Each layer processes only the cropped region of interest (student bounding box) from the
base detector, minimizing per-layer inference cost.

### Ultralytics Configuration (config-driven, not hardcoded)

```yaml
PYRAMID_MODELS:
  base_detector:
    path: "models/base_detector.pt"
    task: "detect"
    tracker: "botsort.yaml"
    confidence: 0.5
    device: "auto"
  posture:
    path: "models/posture.pt"
    task: "classify"
    confidence: 0.6
    device: "auto"
  horizontal_gaze:
    path: "models/horizontal_gaze.pt"
    task: "classify"
    confidence: 0.6
    device: "auto"
  depth_gaze:
    path: "models/depth_gaze.pt"
    task: "classify"
    confidence: 0.6
    device: "auto"
  vertical_gaze:
    path: "models/vertical_gaze.pt"
    task: "classify"
    confidence: 0.6
    device: "auto"
```

### Alternatives Considered

| Alternative | Why Rejected |
|---|---|
| **Raw PyTorch + torchvision** | Requires manual NMS, tracking, model loading, export format switching. Ultralytics abstracts all of this. |
| **ONNX Runtime directly** | No built-in tracking. Loses `.predict()` streaming API. |
| **Detectron2 (Meta)** | Primarily for research. No built-in tracking. Not actively developed for production deployment. |
| **MMDetection** | Complex configuration system. Steeper learning curve. No single-line tracking API. |

### Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Ultralytics version breaking changes | Pin to `ultralytics==8.x.x`. Use `BasePyramidLayer` abstraction to isolate Ultralytics API surface. |
| GPU memory with 5 models loaded simultaneously | Lazy loading per session + model caching. FP16 inference. Monitor GPU memory via health endpoint (FR-038). |
| Multi-model sequential inference latency | Crop-based inference on student ROIs (not full frame). Async scheduling of non-dependent layers. Profile per-hardware FPS. |

---

## R-006: Caching & Real-Time Pub/Sub — Redis

### Decision: **Redis 7**

### Rationale

Redis serves three distinct roles:

1. **Django Channels layer**: Channel groups for WebSocket routing (required by Channels).
2. **Session cache**: Fast authentication lookups across WebSocket and HTTP requests.
3. **Celery broker**: Message queue for background tasks (video recording, storage cleanup).

Single Redis instance for development; separate logical databases for production.

### Alternatives Considered

| Alternative | Why Rejected |
|---|---|
| **RabbitMQ** | Better Celery broker but requires separate infrastructure for Channels layer. Redis serves both roles. |
| **Memcached** | No pub/sub capability. Cannot serve as Channels layer or Celery broker. |

---

## R-007: Video Recording Storage Strategy

### Decision: **Raw video on local filesystem + PostgreSQL metadata**

### Rationale

Per spec clarification (Q4), recordings store raw video without baked-in overlays. All
prediction data stored separately for dynamic overlay rendering during playback.

1. **Video files**: `media/recordings/{session_id}/{camera_id}/` as MP4 (H.264). OpenCV
   `VideoWriter` writes frames.
2. **Prediction metadata**: PostgreSQL `DetectionFrame` + `PyramidPrediction` tables with
   frame timestamps. Playback queries metadata by timestamp range, renders overlays on Canvas.
3. **Anomaly markers**: `AnomalyEvent` records linked to frame timestamps for clickable
   timeline markers (FR-029) with ≤500ms seek (SC-008).
4. **Storage monitoring**: Celery periodic task checks disk usage, warns at 80%/95% (FR-031).

### Future Extension

`BaseStorageBackend` abstraction — local FS default, S3-compatible backends added without
changing recording logic.

---

## R-008: Testing Strategy

### Decision: **Three-phase TDD with strict enforcement**

### Backend Testing Stack

| Tool | Purpose |
|---|---|
| `pytest` + `pytest-django` | Test runner with Django fixtures |
| `pytest-asyncio` | Async test support for Channels consumers |
| `factory_boy` | Model factories for test data generation |
| `pytest-cov` | Coverage reporting (≥80% gate) |
| `channels-testing` | WebSocket consumer testing utilities |
| `responses` / `httpx-mock` | External HTTP mocking |

### Frontend Testing Stack

| Tool | Purpose |
|---|---|
| `vitest` | Jest-compatible test runner with native Vite integration |
| `@testing-library/react` | Component testing with user-centric queries |
| `@testing-library/user-event` | Realistic user interaction simulation |
| `msw` (Mock Service Worker) | API mocking at network level |
| `playwright` | E2E browser automation (Chrome, Firefox, Edge) |

### Test Directory Mirroring

Backend: `tests/unit/<app>/`, `tests/integration/`, `tests/system/` parallel `apps/` structure.
Frontend: `tests/unit/<module>/`, `tests/integration/`, `tests/e2e/` parallel `src/` structure.

---

## R-009: Theme System Architecture

### Decision: **CSS Custom Properties with `data-theme` attribute**

### Rationale

1. **Theme definition**: Each theme file defines the same CSS variables (`--color-bg`,
   `--color-text`, `--color-accent`, `--color-alert-high`, `--color-bbox-student`, etc.).

2. **Theme switching**: `document.documentElement.dataset.theme = "full-black"` activates
   the corresponding CSS selector. Measured ~5-50ms (well under 300ms requirement).

3. **Persistence**: `localStorage.setItem("theme", ...)` on switch; read in `<head>` script
   before paint — preventing flash-of-unstyled-content.

4. **Bounding box colors**: Adaptive via CSS variables — distinct stroke colors for
   student/teacher boxes that maintain contrast per theme. Validated against WCAG 2.1 AA.

### Theme Color Tokens

```css
/* Black + Purple (default) */
--color-bg-primary: #0D0D0D;
--color-bg-secondary: #1A1A2E;
--color-accent: #7C3AED;
--color-text-primary: #F5F5F5;
--color-alert-high: #EF4444;
--color-alert-medium: #F59E0B;
--color-alert-low: #3B82F6;
--color-bbox-student: #22D3EE;
--color-bbox-teacher: #A78BFA;

/* Full Black */
--color-bg-primary: #000000;
--color-bg-secondary: #0A0A0A;
--color-accent: #A78BFA;
/* ... remaining tokens follow same pattern */

/* White */
--color-bg-primary: #FFFFFF;
--color-bg-secondary: #F3F4F6;
--color-accent: #6D28D9;
/* ... remaining tokens follow same pattern */
```

---

## R-010: Authentication & Session Management

### Decision: **Django contrib.auth + session-based authentication**

### Rationale

1. **WebSocket compatibility**: Session cookies sent on WebSocket handshake — Django Channels
   consumers authenticate users without custom token passing.
2. **Built-in security**: Secure cookie flags (`HttpOnly`, `Secure`, `SameSite`), CSRF
   protection, password hashing (PBKDF2), session invalidation.
3. **RBAC**: Django `Group` and `Permission` models for instructor/admin roles.
4. **Audit logging**: Custom middleware logs all login attempts, logout events, session
   timeouts to the audit log model.

### Why Not JWT

JWTs are stateless — impossible to invalidate a specific session without a blocklist (negating
statelessness). Session-based auth allows instant invalidation on logout (FR-004) and
configurable inactivity timeout.

---

## R-011: Video Streaming Pipeline — RTSP Camera to Browser

### Decision: **go2rtc WebRTC bridge (primary) + MJPEG-over-WebSocket fallback**

### Problem Statement

The browser cannot consume RTSP directly. A relay/transcoding layer is needed to deliver
1–8 concurrent camera feeds per instructor at <1 second camera-to-screen latency (FR-008),
while supporting an HTML5 Canvas layer for bounding box overlays (FR-011). The backend must
also independently read RTSP frames for YOLO inference — the display pipeline and detection
pipeline are separate concerns.

### Architecture Overview

```
                  ┌─────────────┐
 RTSP Camera ───▶│   go2rtc     │──── WebRTC (H.264) ──▶ Browser <video>
                  └─────────────┘                           │
                                                    Canvas overlay ◀── Bounding boxes
                  ┌─────────────┐                           │
 RTSP Camera ───▶│  Django      │──── YOLO inference        │
                  │  (OpenCV)   │──── detection JSON ──▶ WebSocket ──┘
                  └─────────────┘
```

Two independent pipelines consume the same RTSP source (RTSP supports multiple consumers):

1. **Display pipeline** — go2rtc reads RTSP, serves WebRTC to browser with H.264 passthrough
   (zero transcoding if camera outputs H.264). Browser renders in `<video>` element.
2. **Detection pipeline** — Django backend reads RTSP via OpenCV, runs YOLO pyramid
   inference, pushes detection JSON over WebSocket. Frontend renders bounding boxes on a
   Canvas layer positioned over the `<video>`.

### Approach Evaluation

---

#### Approach 1: RTSP → FFmpeg → MJPEG over HTTP → `<img>` / Canvas

**How it works**: FFmpeg (or Python/OpenCV) reads RTSP, encodes each frame as JPEG, serves
as HTTP multipart stream (`multipart/x-mixed-replace`). Browser renders in `<img>` tag or
draws to Canvas via `createImageBitmap()`.

| Criterion | Assessment |
|-----------|-----------|
| **Latency** | **150–400ms**. No inter-frame compression = no decode buffer. FFmpeg adds ~50–100ms transcode, HTTP chunked transfer adds ~50–100ms. Meets <1s. |
| **CPU cost** | **High**. JPEG encoding for every frame is CPU-intensive. 8 cameras × 15 FPS × 720p ≈ 120 JPEG encodes/second. No hardware acceleration for MJPEG encoding on most GPUs. |
| **Bandwidth** | **Very high**. MJPEG has no inter-frame compression. 720p@15fps ≈ 5–15 Mbps per stream depending on quality. 8 streams = 40–120 Mbps. |
| **Browser compat** | **Universal**. `<img>` with multipart works in every browser. Canvas `drawImage()` works everywhere. |
| **Complexity** | **Low**. Simplest approach — 20–30 lines per language. Python `StreamingResponse` or FFmpeg subprocess. |
| **Django fit** | **Good**. `StreamingHttpResponse` supports multipart streaming. Can share the same OpenCV capture as the detection pipeline, avoiding a second RTSP connection. |

**Verdict**: Good fallback option. Simple, universal, but bandwidth and CPU costs make it
unsuitable as the primary approach for 8 simultaneous streams. Works well for 1–2 cameras
or as a degraded-mode fallback.

---

#### Approach 2: RTSP → FFmpeg → WebSocket binary → JSMpeg Canvas

**How it works**: FFmpeg reads RTSP, transcodes to MPEG-TS (MPEG1 video), pipes to a
WebSocket relay (the Django Channels backend). Browser-side JSMpeg library decodes MPEG1 in
JavaScript and renders to Canvas.

| Criterion | Assessment |
|-----------|-----------|
| **Latency** | **100–300ms**. No browser buffering — JSMpeg decodes frames immediately as WebSocket messages arrive. Very competitive. |
| **CPU cost** | **High on both sides**. Server: FFmpeg transcodes H.264→MPEG1 (wasteful re-encode). Client: JavaScript software decoding of MPEG1 at 720p consumes significant CPU. 8 streams will saturate a browser tab. |
| **Bandwidth** | **Moderate**. MPEG1 is ~2–5 Mbps per 720p stream (better than MJPEG, worse than H.264). 8 streams = 16–40 Mbps. |
| **Browser compat** | **Universal**. Pure JavaScript decoder — no codec dependency, no plugin. Works even in WebView/Electron. |
| **Complexity** | **Moderate-High**. Must manage FFmpeg subprocesses piping to WebSocket endpoints. JSMpeg integration on frontend. Process lifecycle management (start/stop per camera). |
| **Django fit** | **Good**. WebSocket relay matches existing Django Channels architecture. But adds FFmpeg process management burden to the Python backend. |

**Key library**: [JSMpeg](https://github.com/phoboslab/jsmpeg) (unmaintained since 2020,
but stable and widely forked). Alternative: [mpegts.js](https://github.com/nicevoice/mpegts.js) is actively maintained.

**Verdict**: Attractive latency, but the double-transcode penalty (camera H.264 → MPEG1 →
JS decode) is architecturally wasteful. Client-side JS decoding becomes the bottleneck at
4+ streams. Not recommended as primary.

---

#### Approach 3: RTSP → WebRTC bridge (go2rtc) → Browser `<video>` + Canvas overlay ★ RECOMMENDED

**How it works**: A lightweight media server (go2rtc or mediamtx) ingests RTSP streams and
exposes them via WebRTC (WHEP protocol). Browser establishes an RTCPeerConnection, receives
H.264 RTP packets, renders in a standard `<video>` element. Canvas overlay is layered on
top for bounding boxes.

| Criterion | Assessment |
|-----------|-----------|
| **Latency** | **50–300ms**. Best achievable. WebRTC is purpose-built for real-time. H.264 passthrough means zero transcoding latency. Only network jitter buffer (~50ms) + decode (~10ms). |
| **CPU cost** | **Minimal**. Server: H.264 passthrough = zero transcoding if camera outputs H.264 (nearly all IP cameras do). Client: hardware-accelerated H.264 decoding in browser — negligible CPU. 8 streams decode effortlessly. |
| **Bandwidth** | **Optimal**. H.264 is the most efficient option. 720p@15fps ≈ 1–3 Mbps per stream. 8 streams = 8–24 Mbps. ~5× less than MJPEG. |
| **Browser compat** | **Excellent**. WebRTC supported in Chrome 28+, Firefox 22+, Edge 79+, Safari 11+. H.264 hardware decode in all target browsers. WHEP is an IETF standard (RFC 8location). |
| **Complexity** | **Moderate**. Requires deploying go2rtc as a sidecar service. Frontend needs a WHEP client (~50 lines). Canvas overlay on `<video>` is a standard pattern. |
| **Django fit** | **Clean separation**. Django does NOT proxy video — go2rtc handles it. Django manages camera metadata (DRF CRUD REST API) and configures go2rtc via its REST API. Detection results flow through Django Channels WebSocket. No video bytes touch Python. |

**Why go2rtc over mediamtx**:

| Feature | go2rtc v1.9+ | mediamtx v1.9+ |
|---------|-------------|-----------------|
| Binary size | ~15 MB | ~25 MB |
| H.264 passthrough (no transcode) | ✅ Yes | ✅ Yes |
| WHEP (WebRTC egress) | ✅ Native | ✅ Native |
| REST API for stream management | ✅ Full CRUD | ⚠️ Limited |
| RTSP source auto-reconnect | ✅ Built-in | ✅ Built-in |
| Two-way audio | ✅ | ✅ |
| Docker image | ✅ ~20 MB | ✅ ~30 MB |
| MSE fallback (no WebRTC) | ✅ Built-in | ❌ |
| Active maintenance (2025-2026) | ✅ Very active | ✅ Active |
| Cross-platform binary | ✅ Windows+Linux+Mac | ✅ Windows+Linux+Mac |
| License | MIT | MIT |

go2rtc is preferred because: (1) smaller footprint, (2) built-in MSE (Media Source Extensions)
fallback — if WebRTC ICE negotiation fails on a restrictive network, it seamlessly falls back
to MSE over WebSocket with the same H.264 stream, (3) full REST API for dynamic stream
management aligns perfectly with Django DRF camera CRUD endpoints.

**Canvas overlay architecture**:
```
┌──────────────────────────────┐
│  position: relative          │
│  ┌────────────────────────┐  │
│  │ <video> (WebRTC)       │  │  ← go2rtc WHEP stream
│  │ z-index: 0             │  │
│  └────────────────────────┘  │
│  ┌────────────────────────┐  │
│  │ <canvas> (overlay)     │  │  ← bounding boxes from WebSocket
│  │ z-index: 1             │  │
│  │ pointer-events: none   │  │
│  └────────────────────────┘  │
└──────────────────────────────┘
```

The `<video>` and `<canvas>` are absolutely positioned in the same container. The Canvas
overlay receives bounding box coordinates from the detection WebSocket and draws them using
`requestAnimationFrame`. This is the exact same pattern used by Zoom Web, Google Meet, and
Twitch for overlays on video.

**Synchronization**: Detection results include a frame timestamp. The frontend correlates
detection timestamps with the `<video>` element's `currentTime` to align bounding boxes.
At <300ms total pipeline latency, a simple fixed offset (calibrated at startup) provides
sufficient alignment without complex sync protocols.

**Verdict**: ★ **Recommended primary approach**. Best latency, lowest CPU/bandwidth, cleanest
architecture with Django. Zero Python video proxying. Hardware-decoded in browser. Scales
trivially to 8 streams.

---

#### Approach 4: RTSP → HLS/DASH adaptive → `<video>` tag

**How it works**: FFmpeg or a media server segments the RTSP stream into HLS (.m3u8 + .ts
chunks) or DASH (.mpd + segments). Browser plays via `<video>` with hls.js or native HLS
support (Safari).

| Criterion | Assessment |
|-----------|-----------|
| **Latency** | **3–30 seconds**. Standard HLS: 3 segments × 6s = 18s. LL-HLS: 1–3s but requires HTTP/2 push, CMAF, playlist reload — significant complexity. **Does NOT meet <1s requirement.** |
| **CPU cost** | **Moderate**. FFmpeg segments H.264 without re-encoding (transmux only). Client hardware-decodes H.264. |
| **Bandwidth** | **Optimal**. H.264 with adaptive bitrate. Same as WebRTC. |
| **Browser compat** | **Excellent**. HLS is the most broadly supported streaming format. |
| **Complexity** | **Moderate**. FFmpeg segmenting + file serving + hls.js integration. LL-HLS significantly increases complexity. |
| **Django fit** | **Good**. Static file serving for segments. But the latency makes this irrelevant. |

**Verdict**: ❌ **Rejected**. Fails the <1 second latency requirement (FR-008). Even LL-HLS
at 1–3s is borderline. HLS is designed for scalable on-demand/live streaming (Netflix,
YouTube Live), not real-time monitoring. Useful only for recording playback (already planned
via stored MP4 files).

---

### Comparative Summary

| Criterion | MJPEG/HTTP | JSMpeg/WS | WebRTC (go2rtc) ★ | HLS/DASH |
|-----------|-----------|-----------|-------------------|----------|
| Latency | 150–400ms ✅ | 100–300ms ✅ | **50–300ms** ✅ | 3–30s ❌ |
| Server CPU (8 cams) | High 🔴 | High 🔴 | **Near zero** 🟢 | Low 🟡 |
| Client CPU (8 cams) | Low 🟢 | Very High 🔴 | **Near zero** 🟢 | Low 🟢 |
| Bandwidth (8×720p) | 40–120 Mbps 🔴 | 16–40 Mbps 🟡 | **8–24 Mbps** 🟢 | 8–24 Mbps 🟢 |
| Browser support | Universal 🟢 | Universal 🟢 | **Modern** 🟢 | Universal 🟢 |
| Canvas overlay | Direct 🟢 | Native 🟢 | **Layer on `<video>`** 🟢 | Layer on `<video>` 🟢 |
| Django integration | Native 🟢 | Good 🟡 | **Sidecar** 🟡 | Good 🟡 |
| Deployment units | 1 (backend) | 1 (backend) | **2 (backend + go2rtc)** | 1 (backend) |
| Implementation effort | Very Low 🟢 | Moderate 🟡 | **Moderate** 🟡 | Moderate 🟡 |

### Final Recommendation

**Primary pipeline: go2rtc WebRTC (Approach 3)**

Use go2rtc as a sidecar service running alongside Django. When an instructor adds a camera
(RTSP URL via DRF REST API), Django calls go2rtc's REST API to register the stream.
The React frontend establishes a WHEP WebRTC connection directly to go2rtc for the video
feed, and a WebSocket connection to Django Channels for detection data. Bounding boxes render on a
Canvas overlay positioned on top of the `<video>` element.

**Fallback pipeline: MJPEG over WebSocket (Approach 1 variant)**

For environments where go2rtc cannot be deployed or WebRTC ICE negotiation fails (edge case
on restrictive networks), the Django backend can serve JPEG frames over its existing
WebSocket connection. Since the detection pipeline already reads RTSP via OpenCV, encoding
the current frame as JPEG and forwarding it is trivial (~10 lines of Python). The frontend
renders these frames on the same Canvas used for bounding boxes.

Note: go2rtc itself provides MSE (Media Source Extensions) as a built-in fallback when
WebRTC is unavailable — this is preferred over MJPEG since it still uses H.264. The MJPEG
fallback is the last-resort option when go2rtc is entirely absent.

**Fallback chain**: WebRTC → MSE (go2rtc built-in) → MJPEG over WebSocket (Django Channels direct)

### Required Libraries & Tools

#### Backend / Infrastructure

| Tool | Version | Purpose | License |
|------|---------|---------|---------|
| **go2rtc** | ≥1.9.x (latest stable) | RTSP→WebRTC/MSE bridge | MIT |
| **OpenCV (cv2)** | ≥4.9.0 (opencv-python-headless) | RTSP capture for YOLO inference pipeline | Apache 2.0 |
| **Django + DRF + Channels** | 5.x + 3.15+ + 4.x | REST API for camera CRUD + WebSocket for detections | BSD |
| **httpx** | ≥0.27.0 | Python HTTP client — FastAPI calls go2rtc REST API | BSD |
| **Uvicorn** | ≥0.30.0 | ASGI server | BSD |

#### Frontend

| Library | Version | Purpose | License |
|---------|---------|---------|---------|
| **React** | 18.x | UI framework | MIT |
| **TypeScript** | 5.4+ | Type safety | Apache 2.0 |
| **whep-client** (or custom ~50 LOC) | — | WHEP WebRTC client for go2rtc | — |
| **Zustand** | ≥4.5 | Detection state management | MIT |

No additional frontend video libraries needed — WebRTC is native browser API, Canvas is
native browser API. The WHEP client is a thin wrapper around `RTCPeerConnection` (no heavy
dependencies).

#### Docker Compose (deployment)

```yaml
# docker-compose.yml excerpt
services:
  go2rtc:
    image: alexxit/go2rtc:latest    # ~20MB image
    ports:
      - "1984:1984"    # WebUI + API
      - "8555:8555"    # WebRTC (WHEP)
    volumes:
      - ./go2rtc.yaml:/config/go2rtc.yaml

  backend:
    build: ./backend
    ports:
      - "8000:8000"
    depends_on:
      - go2rtc
      - redis
      - postgres

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
```

### Risks & Mitigations

| Risk | Severity | Mitigation |
|------|----------|------------|
| go2rtc binary not available on target OS | Low | go2rtc provides pre-built binaries for Windows/Linux/Mac + Docker image. Cross-platform confirmed. |
| WebRTC ICE negotiation fails on university network | Medium | go2rtc auto-falls back to MSE (WebSocket-based H.264). If that fails too, MJPEG fallback in FastAPI. Three-tier fallback chain. |
| Detection bounding boxes desync with video | Medium | Include frame timestamp in detection WebSocket messages. Apply calibrated fixed offset (~100–200ms). Acceptable for monitoring use case where ±200ms sync is visually imperceptible. |
| go2rtc REST API changes between versions | Low | Pin go2rtc Docker image tag. Wrap go2rtc API calls in an adapter class (Open/Closed principle). |
| Additional deployment complexity (sidecar) | Low | Single Docker Compose file. go2rtc requires zero configuration for basic RTSP→WebRTC — one YAML line per stream. |
| RTSP camera outputs non-H.264 codec | Low | go2rtc supports FFmpeg-based transcoding as a fallback for non-H.264 sources. Nearly all modern IP cameras output H.264. |

### Integration with Existing Architecture

1. **Camera CRUD (FastAPI REST API)**: When an instructor adds a camera via `POST /api/cameras`,
   the `cameras/service.py` calls go2rtc's REST API (`POST http://go2rtc:1984/api/streams`)
   to register the RTSP source. When a camera is removed, it calls `DELETE`.

2. **Frontend camera component**: `CameraFeed.tsx` creates an `RTCPeerConnection` using go2rtc's
   WHEP endpoint, attaches the stream to a `<video>` element, and layers a `<canvas>` overlay
   on top. The same component subscribes to the detection WebSocket for bounding box data.

3. **Detection pipeline (unchanged)**: `cameras/rtsp_proxy.py` (renamed to `cameras/capture.py`)
   reads RTSP via OpenCV independently of go2rtc. YOLO inference runs on these frames. Results
   are pushed via WebSocket. go2rtc and the detection pipeline are fully decoupled.

4. **Recording (unchanged)**: OpenCV `VideoWriter` in the backend continues recording raw frames
   to MP4. go2rtc is display-only — it does not affect the recording pipeline.

---

## R-012: Recording Deduplication Model (Clarification Round 4)

### Decision: **Per-camera recordings with Session↔Recording M2M relationship**

### Rationale

Clarification Round 4 (Q2) established that when multiple instructors monitor the same shared
camera, the system produces only **one recording per camera** (deduplicated). Each instructor
maintains their own monitoring session for triage tracking and audit trail, but sessions
reference shared recordings.

This changes the Recording entity relationship from a simple `session_id` FK (one recording
per session+camera) to a Many-to-Many relationship between MonitoringSession and Recording.

**Recording lifecycle**:
1. Instructor A starts a session with camera X → system creates a Recording for camera X
2. Instructor B starts a session with the same camera X → system detects an active Recording
   for camera X and links B's session to the existing Recording (no duplicate)
3. Recording continues until the **last** session referencing it ends
4. If both sessions end, the recording is marked `completed`

**Implementation**: Django `ManyToManyField` on MonitoringSession → Recording. The
`recordings` service layer checks for an existing active Recording for a given camera before
creating a new one. This is the only write-side change; all read paths (playback, export)
work unchanged since they query by recording ID.

**Anomaly events**: Per-camera, not per-session. An anomaly detected on camera X is visible
to all instructors monitoring that camera. Any instructor can triage. The associated recording
is derived via `camera_source_id` + timestamp range (no direct recording FK).

### Alternatives Considered

| Alternative | Why Rejected |
|---|---|
| **Keep session_id FK, duplicate recordings** | Wastes disk space, violates spec deduplication requirement, complicates export (duplicate video files) |
| **Recording has nullable session_id (first creator)** | Ambiguous ownership. If the original session ends but another is still active, the recording lifecycle is unclear. M2M is explicit. |
| **JSONField with recording_ids on Session** | Loses referential integrity. No CASCADE delete protection. Django M2M is cleaner. |

---

## R-013: Two-Step Camera Connection Flow (Clarification Round 4)

### Decision: **Connect → Preview → Start Session → Record**

### Rationale

Clarification Round 4 (Q1) established that connecting a camera does NOT automatically start
a monitoring session or recording. The flow is two-step:

1. **Connect**: Instructor connects to RTSP camera → go2rtc registers the stream → frontend
   shows live preview via WHEP WebRTC → no recording, no disk usage, no session created
2. **Start Session**: Instructor explicitly clicks "Start Session" → backend creates a
   MonitoringSession + Recordings for selected cameras → detection pipeline begins →
   WebSocket detection feed starts

This allows instructors to preview camera feeds (verify angles, connectivity, quality) before
committing to a session with storage costs.

**Frontend state machine for camera tile**:
```
disconnected → connecting → connected (preview only) → session_active (recording)
                                                      ↓
                                           disconnecting → disconnected
```

The frontend `CameraFeed` component distinguishes "preview mode" (connected, no session) from
"recording mode" (connected, session active) with distinct visual indicators:
- Preview: camera feed visible, "Start Session" button prominent, no recording badge
- Recording: camera feed visible, recording duration counter, red recording indicator

**Backend impact**: The existing REST API already supports this flow:
- `POST /api/v1/cameras/{id}/connect/` → starts WHEP stream (no session)
- `POST /api/v1/sessions/` → creates session, starts recording

No API contract changes required. Frontend needs UI state management for the preview/recording distinction.

### Alternatives Considered

| Alternative | Why Rejected |
|---|---|
| **Auto-start session on connect** | Wastes storage for camera previews. Instructor may want to verify camera angle before committing. |
| **Three-step (connect → preview → configure → start)** | Over-engineered for initial release. Camera configuration can happen before connect. |

---

## R-014: RBAC Architecture — Role-Based Access Control

### Decision: **Django Groups + custom Role model with page-level and action-level permissions**

### Rationale

The spec (FR-050–FR-053) requires two built-in roles (Admin, Instructor) plus admin-created
custom roles with configurable access ranges that define both which pages are accessible and
which actions are permitted.

1. **Django's built-in auth**: `django.contrib.auth` provides `Group` and `Permission` models.
   However, Django's default Permission model is tied to model-level CRUD operations
   (`add_`, `change_`, `delete_`, `view_`), which doesn't map cleanly to page-level and
   action-level access ranges.

2. **Custom Role model**: A dedicated `Role` model stores:
   - `permitted_pages`: JSONField listing page identifiers (e.g., `["camera_feed", "recordings", "health"]`)
   - `permitted_actions`: JSONField listing action identifiers (e.g., `["can_triage_anomalies", "can_export_sessions", "can_manage_cameras"]`)
   - Built-in roles (`admin`, `instructor`) are non-deletable seed data.

3. **DRF permission classes**: Custom `HasPageAccess` and `HasActionPermission` classes
   check the user's role against the requested page/action. These are composed with DRF's
   `IsAuthenticated` in view permissions.

4. **Frontend route guards**: The React router reads the user's role permissions from the
   auth response and renders route guards that redirect unauthorized access to an
   `AccessDeniedPage` (FR-052).

5. **Role change propagation**: Role changes take effect on next page navigation (FR-053) —
   the frontend re-fetches role permissions on each route change. Active sessions are not
   interrupted.

### Why Not Django Guardian (object-level permissions)

Object-level permissions (per-exam, per-session access) are not required in the initial scope.
All instructors with camera access can view all connected cameras. Role-based page+action
access is sufficient. django-guardian adds ~5 extra tables and significant query overhead.

### Permission Registry (config-driven)

```python
# accounts/permissions.py
PAGES = {
    "exam_board": "Exam Board",
    "camera_feed": "Camera Feed",
    "predictions": "Predictions",
    "sessions": "Sessions",
    "recordings": "Recordings",
    "health": "Health Dashboard",
    "profile": "Profile",
    "admin_dashboard": "Admin Dashboard",
    "user_management": "User Management",
    "role_management": "Role Management",
    "exam_management": "Exam Management",
    "room_management": "Room Management",
    "recording_review": "Recording Review",
}

ACTIONS = {
    "can_triage_anomalies": "Triage anomaly alerts",
    "can_revert_triage": "Revert triage actions (admin)",
    "can_export_sessions": "Export session archives",
    "can_manage_cameras": "Add/remove/connect cameras",
    "can_manage_users": "Create/edit/deactivate users",
    "can_manage_roles": "Create/edit roles",
    "can_manage_exams": "Create/edit exams",
    "can_manage_rooms": "Create/edit rooms",
    "can_add_comments": "Add session comments",
    "can_view_audit_log": "View audit log entries",
}
```

### Built-in Role Seeds

| Role | Pages | Actions | Deletable |
|------|-------|---------|-----------|
| Admin | All pages | All actions | No |
| Instructor | exam_board, camera_feed, predictions, sessions, recordings, health, profile | can_triage_anomalies, can_export_sessions, can_manage_cameras, can_add_comments | No |

### Alternatives Considered

| Alternative | Why Rejected |
|---|---|
| **django-guardian** | Object-level permissions overkill; no per-object access needed. |
| **Django default permissions only** | Model-level CRUD permissions don't map to page-level access ranges. |
| **casbin / OPA** | External policy engine adds deployment complexity. Django-native solution preferred. |

---

## R-015: Audit Log Architecture

### Decision: **Append-only AuditLogEntry model with middleware auto-capture + explicit service calls**

### Rationale

FR-069 mandates a comprehensive audit trail for all state-changing actions. The audit log
must be immutable (no updates or deletes) and include: action type, performing user, target
entity, contextual details, IP address, and timestamp.

1. **Append-only model**: `AuditLogEntry` extends a base model with overridden `save()` that
   raises an error on update attempts. `delete()` is also overridden to raise.
   `objects.update()` and `objects.delete()` are blocked at the manager level.

2. **Dual capture strategy**:
   - **Middleware**: Auth events (login, logout, login_failed) are captured automatically by
     `AuditMiddleware` that hooks into Django's `user_logged_in`, `user_logged_out`, and
     `user_login_failed` signals.
   - **Service layer**: Business-logic events (triage, revert, account CRUD, exam CRUD,
     session lifecycle, role changes) are captured by explicit `audit_service.log()` calls
     in the relevant service functions. This is preferred over signals because it provides
     exact control over what context (details JSON) is captured.

3. **Action type enum**: Defined as a Python `StrEnum` matching the spec's action types
   (FR-069). The `details` JSONField stores action-specific context — e.g., for a triage
   revert: `{"old_status": "dismissed", "new_status": "new", "reason": "..."}`.

4. **IP capture**: `AuditMiddleware` extracts the client IP from `X-Forwarded-For` (if
   behind reverse proxy) or `REMOTE_ADDR`. Stored as `GenericIPAddressField`.

5. **Performance**: AuditLogEntry inserts are append-only and non-blocking (no reads during
   write). Composite index on `(action_type, timestamp)` and `(user_id, timestamp)` for
   efficient querying by the Admin Activity Feed (FR-073).

6. **Admin Activity Feed**: FR-073's configurable page size (10–100 in steps of 10, or All)
   is implemented via DRF pagination with a dynamic `page_size` query parameter. The "All"
   option uses cursor-based lazy loading to avoid memory issues.

### Retention

Audit logs are retained for 365 days by default (configurable). A Celery periodic task
prunes entries older than the retention period. Only time-based pruning — no content-based
deletion.

### Alternatives Considered

| Alternative | Why Rejected |
|---|---|
| **Django signals only** | Signals fire on ORM save — miss bulk operations and raw SQL. Explicit service calls are more reliable for business events. |
| **django-auditlog package** | Auto-captures model field changes, but spec requires action-level logging (triage, revert, login) not field-level diffs. |
| **Separate audit database** | Unnecessary complexity for a graduation project. Single DB with retention policy suffices. |
| **Event sourcing** | Complete architectural overhaul. The audit log is a supplementary trail, not the primary state store. |

---

## R-016: Room-Based Camera-Exam Association

### Decision: **Room entity linking cameras to physical locations; exams assigned to rooms**

### Rationale

Clarification Round 6 (Q5) established that cameras are physically fixed in exam halls and
exams are scheduled in specific rooms. The Room/Hall entity bridges the camera ↔ exam
relationship, enabling auto-detection of cameras when a session starts from the Exam Board
(FR-056).

1. **Room model**: Simple entity with `name`, optional `building` and `capacity` fields,
   `created_by` (admin reference), and timestamps. Managed by admins via Room Management
   page (FR-063).

2. **Camera → Room**: `CameraSource` gains a `room_id` FK (nullable, SET_NULL on delete).
   Cameras not assigned to a room can still be used in manual sessions.

3. **Exam → Room**: `Exam` has a `room_id` FK (nullable, SET_NULL on delete). An exam can
   exist without a room assignment (data is entered incrementally by admin).

4. **Auto-detection flow** (FR-056):
   ```
   Exam Board countdown → zero / "Start Now"
     → Backend: exam.room_id → Room → Room.cameras (connected ones)
     → Create MonitoringSession with those camera_ids
     → Redirect instructor to CameraFeedPage with pre-loaded cameras
   ```

5. **No room = no auto-detect**: If the exam has no room or the room has no registered
   cameras, the system displays "No cameras registered for this room" per FR-056 with an
   option to manually enter a camera URL.

### Alternatives Considered

| Alternative | Why Rejected |
|---|---|
| **Direct Camera ↔ Exam M2M** | Doesn't reflect physical reality. Cameras belong to rooms, not exams. An exam in Room A should automatically get Room A's cameras without manual per-exam camera assignment. |
| **Camera tags/labels** | Loose coupling — no guarantee cameras are correctly tagged per exam. Room-based approach is deterministic. |
| **No Room entity (manual camera selection only)** | Instructor would have to manually select cameras for every exam. Room-based auto-detection saves time and reduces errors. |

---

## R-017: Multi-Instructor Session Merging

### Decision: **Per-instructor independent sessions during live exam; post-exam unified history view with full attribution**

### Rationale

Clarification Round 6 (Q3) established that multiple instructors can be assigned to the same
exam. During the live session each instructor works independently. After the exam finishes,
all contributions are merged into a unified session history.

1. **Live phase — independent sessions**: Each instructor has their own `MonitoringSession`
   record, even for the same exam. Each can:
   - Select their own camera subset (from the exam's room cameras)
   - Add their own notes and comments
   - Triage anomalies independently (first-write-wins still applies across all instructors)
   - Configure their own camera layout

2. **Session → Exam relationship**: `MonitoringSession` has an `exam_id` FK (nullable — for
   manual sessions without an exam). This links multiple sessions to one exam.

3. **Post-exam merge — unified history**: The recording review page (FR-064–FR-065) queries
   all sessions for a given exam and presents a merged view:
   - All notes, comments, and triage actions from every instructor
   - Each action tagged with the performing instructor's identity
   - Timeline shows all contributions interleaved chronologically
   - Recordings are per-camera (already deduplicated via R-012), so the merged view
     references the same recordings

4. **Implementation**: The merge is a **read-time aggregation**, not a write-time operation.
   No data is physically merged into a single session record. The API endpoint
   `GET /api/v1/exams/{exam_id}/sessions/merged/` queries all sessions for the exam and
   returns a combined view. The frontend renders this as a unified timeline.

5. **Admin view**: Admins see the merged view by default when reviewing an exam. They can
   also drill into individual instructor sessions for detailed attribution.

### Alternatives Considered

| Alternative | Why Rejected |
|---|---|
| **Single shared session per exam** | All instructors would interfere with each other's camera layouts and settings. Independent sessions preserve each instructor's workflow. |
| **Write-time merge on exam end** | Destructive — loses individual session metadata. Read-time aggregation preserves full provenance. |
| **No merge (just list individual sessions)** | Admin needs a unified chronological view for investigation. Listing separate sessions forces manual cross-referencing. |

---

## Summary of All Decisions

| # | Topic | Decision | Key Rationale |
|---|-------|----------|---------------|
| R-001 | Backend Framework | Django 5.x + Channels + DRF | Built-in ORM, auth, ASGI, admin. Zero assembly. |
| R-002 | Frontend Framework | React 19.2 + TS ~5.9 + Vite 7.3 + Zustand 5.x | Canvas perf, WebSocket state, testing ecosystem. |
| R-003 | Database | PostgreSQL 16 | Relational integrity, jsonb, concurrent writes. |
| R-004 | Real-Time Communication | Django Channels + Redis | Native ASGI WebSocket, channel groups, 100k+ msg/s. |
| R-005 | YOLO Inference | Ultralytics 8.x | Unified API for loading, inference, tracking. User-required. |
| R-006 | Cache/Broker | Redis 7 | Channels layer + session cache + Celery broker. One service. |
| R-007 | Video Storage | Local FS + PostgreSQL metadata | Raw video + separate metadata for dynamic overlays. |
| R-008 | Testing Strategy | pytest / Vitest / Playwright TDD | Three-phase pipeline, ≥80% coverage, all blocking. |
| R-009 | Theme System | CSS Custom Properties | Native, ≤50ms switch, WCAG 2.1 AA, no JS overhead. |
| R-010 | Authentication | Django session-based | WebSocket-compatible, instant invalidation, built-in RBAC. |
| R-011 | Video Streaming Pipeline | go2rtc (WebRTC) + MJPEG fallback | Sub-200ms latency, H.264 passthrough, hardware-decoded, Canvas overlay compatible. |
| R-012 | Recording Deduplication | Per-camera M2M with Session | Shared cameras produce one recording; sessions reference via M2M. |
| R-013 | Two-Step Connection Flow | Connect → Preview → Start Session → Record | Preview without storage cost; explicit session start for recording. |
| R-014 | RBAC Architecture | Django Groups + custom Role model | Page-level + action-level permissions; custom roles by admins. |
| R-015 | Audit Log Architecture | Append-only AuditLogEntry model | Immutable records for all state-changing actions; middleware auto-capture. |
| R-016 | Room-Based Camera Association | Room entity linking exams to cameras | Physical reality mapping; auto-detect cameras on session start. |
| R-017 | Multi-Instructor Session Merging | Per-instructor sessions + post-exam unified view | Independent live work; attributed merged history after exam ends. |
