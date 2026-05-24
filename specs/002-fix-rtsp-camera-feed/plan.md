# Implementation Plan: Fix RTSP Camera Feed Connection

**Branch**: `002-fix-rtsp-camera-feed` | **Date**: 2026-03-04 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/002-fix-rtsp-camera-feed/spec.md`

## Summary

The "Connect" button only toggles the `CameraSource.status` field in the database — it never registers the RTSP stream with the go2rtc media relay, so the WHEP video player has nothing to play back. Additionally, the nginx WHEP proxy targets port 8555 (WebRTC ICE/TCP transport) instead of port 1984 (HTTP API where WHEP signaling lives), and the frontend WHEP client connects directly to go2rtc bypassing authentication.

This plan fixes the three-layer break (backend service → nginx proxy → frontend client), adds a three-stage RTSP probe for pre-connect validation, encrypts RTSP URLs at rest, implements real-time status broadcasting via WebSocket, adds a Celery-beat health-check with auto-reconnection, and provides clear, categorized error messages at every failure point.

## Technical Context

**Language/Version**: Python 3.10+ (backend), TypeScript 5.9.3 (frontend)
**Primary Dependencies**: Django 5.1.5, DRF 3.15.2, Django Channels 4.2.2, Celery 5.4.0, Pydantic 2.10.6, psycopg 3.2.5, React 19.2, Vite 7.3.1, Zustand 5.0.11, Axios 1.13.6
**Storage**: PostgreSQL 16 (via `postgres:16-alpine`), Redis 7 (via `redis:7-alpine`) for channel layer + Celery broker
**Testing**: pytest 8.3.4 (backend), Vitest 4.0.18 (frontend unit), Playwright 1.58.2 (E2E)
**Target Platform**: Docker Compose dev environment (Linux containers), Windows + Linux CI
**Project Type**: Web application (Django REST + React SPA) with go2rtc media relay sidecar
**Performance Goals**: Camera feed visible within 15 seconds of clicking Connect (SC-001). Status updates reflected across clients within 2 seconds (SC-003). UI interactions ≤200 ms response time (Constitution §VI).
**Constraints**: ≤10 s RTSP probe timeout (configurable), ≤14 s auto-reconnection window (3 retries × exponential backoff 2s→4s→8s), ≥80% test coverage gate (Constitution §II)
**Scale/Scope**: Up to 8 cameras per instructor (FR-017), no hard global limit, 90-day connection event retention

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| # | Principle | Verdict | Evidence |
|---|-----------|---------|----------|
| I | Code Quality Excellence | **PASS** | SOLID: `CameraService` (single responsibility), `EncryptedTextField` (open/closed via `_get_fernet()`), custom exception hierarchy (`CameraConnectionError` subclasses), config via `constants.py` (no magic numbers). Functions ≤30 lines. Structured logging with `mask_rtsp_url()` redaction. |
| II | Testing Standards (NON-NEGOTIABLE) | **PASS** | TDD workflow. Three-phase pipeline: unit (probe, service, serializer, hooks), integration (go2rtc lifecycle, WS broadcast, Celery health-check), E2E (Playwright camera-connect flow). ≥80% coverage gate. Tests mirror source structure. |
| III | User Experience Consistency | **PASS** | Persistent inline error banners + brief toasts (FR-003). Multi-stage progress indicator (FR-004). Status badges with colour semantics (green=connected, red=error, yellow=reconnecting). Loading states on Connect/Disconnect buttons (FR-010). Consolidated bulk-failure banner (FR-027). |
| IV | Security Standards | **PASS** | `EncryptedTextField` with Fernet (FR-015). `FIELD_ENCRYPTION_KEY` in env var (never hardcoded). HMAC-SHA256 blind index for encrypted-field lookups (FR-019). `mask_rtsp_url()` for logs/UI. WHEP through nginx `auth_request` (FR-005). RBAC: instructor owns cameras, admin has system-wide access (FR-023, FR-024). Audit log via `ConnectionEvent` (FR-013). |
| V | Amazing UI Design & UX | **PASS** | Three themes required from Day 1 (constitution §V) — component tasks must style `CameraStatusBadge`, `ConnectionProgress`, `ErrorBanner` across all three themes. WCAG 2.1 AA contrast for status colours. ARIA live regions for dynamic status updates (FR-025). |
| VI | Performance Requirements | **PASS** | WHEP streaming at native frame rate via go2rtc. Health-check polling every 15s (lightweight GET). Exponential backoff prevents thundering herd. Frontend initial load ≤3s. UI interactions ≤200 ms. |
| VII | Delivery Sign-Off Protocol | **PASS** | Lint + type-check + test pipeline. Module READMEs for new/modified packages. `docs/` mirror files with Mermaid diagrams. Conventional Commits. PR + CodeRabbit review gate. |

## Project Structure

### Documentation (this feature)

```text
specs/002-fix-rtsp-camera-feed/
├── plan.md                    # This file
├── research.md                # Phase 0 — go2rtc API, encryption, WS broadcasting, WHEP proxy
├── research-rtsp-probe.md     # Phase 0 — Three-stage RTSP probe library comparison
├── data-model.md              # Phase 1 — CameraSource, StreamRegistration, ConnectionEvent
├── quickstart.md              # Phase 1 — Dev setup and verification guide
├── contracts/
│   ├── rest-api.md            # Phase 1 — 11 REST endpoints
│   ├── websocket.md           # Phase 1 — WS channels and message formats
│   └── whep-signaling.md      # Phase 1 — WHEP proxy config and SDP format
└── tasks.md                   # Phase 2 — 53 tasks across 8 phases
```

### Source Code (repository root)

```text
backend/
├── apps/cameras/
│   ├── models.py              # CameraSource (modified), StreamRegistration (new), ConnectionEvent (new)
│   ├── services.py            # CameraService: connect, disconnect, probe, health-check, broadcast
│   ├── views.py               # DRF viewsets: connect/, disconnect/, events/, admin endpoints
│   ├── serializers.py         # CameraSourceSerializer (masked URL), ConnectionEventSerializer
│   ├── consumers.py           # CameraStatusConsumer (per-camera + global groups)
│   ├── tasks.py               # NEW: Celery beat health-check, event cleanup
│   ├── exceptions.py          # NEW: CameraConnectionError hierarchy (network, auth, no-video, relay)
│   ├── constants.py           # NEW: Timeouts, retry params, error message templates
│   ├── fields.py              # NEW: EncryptedTextField (Fernet encrypt/decrypt)
│   ├── routing.py             # WebSocket URL routes (per-camera + global)
│   ├── admin_urls.py          # Admin camera dashboard endpoints
│   └── migrations/            # 5 new migrations (see data-model.md)
├── core/
│   └── fields.py              # EncryptedTextField (shared, importable from core)
└── tests/
    ├── unit/cameras/          # Probe, service, serializer, field, exception tests
    ├── integration/cameras/   # go2rtc lifecycle, WS broadcast, Celery task tests
    └── system/cameras/        # End-to-end connect flow tests

frontend/
├── src/
│   ├── api/cameras.ts         # Axios: connect, disconnect, events, admin endpoints
│   ├── components/camera/
│   │   ├── CameraFeed.tsx     # Video tile with WHEP player + status overlay
│   │   ├── CameraStatusBadge.tsx  # Colour-coded status badge (3 themes)
│   │   ├── ConnectionProgress.tsx # Multi-stage progress indicator (FR-004)
│   │   └── ErrorBanner.tsx    # Persistent inline error banner (FR-003)
│   ├── hooks/
│   │   ├── useWhepClient.ts   # WHEP signaling via /whep/{id}/ + retry logic (FR-009)
│   │   ├── useCameraStatus.ts # NEW: WS subscription for camera.status updates
│   │   └── useBulkFailure.ts  # NEW: Consolidated failure notification (FR-027)
│   ├── stores/cameraStore.ts  # Zustand: setCameraStatus action
│   ├── constants/cameraMessages.ts  # NEW: User-facing string constants (FR-025)
│   └── types/
│       ├── api.ts             # CameraSource, StreamRegistration, ConnectionEvent types
│       └── websocket.ts       # CameraStatusUpdate type
└── tests/
    ├── unit/                  # Hook, store, component tests
    ├── integration/           # API mock + WS mock flow tests
    └── e2e/                   # Playwright camera-connect flow

docs/
├── backend/apps/cameras/
│   ├── models.md
│   ├── services.md
│   ├── views.md
│   ├── serializers.md
│   ├── consumers.md
│   ├── tasks.md
│   ├── exceptions.md
│   ├── constants.md
│   ├── fields.md
│   └── routing.md
└── frontend/src/
    ├── api/cameras.md
    ├── components/camera/
    │   ├── CameraFeed.md
    │   ├── CameraStatusBadge.md
    │   ├── ConnectionProgress.md
    │   └── ErrorBanner.md
    ├── hooks/
    │   ├── useWhepClient.md
    │   ├── useCameraStatus.md
    │   └── useBulkFailure.md
    ├── stores/cameraStore.md
    └── constants/cameraMessages.md
```

**Structure Decision**: Web application with existing `backend/` + `frontend/` structure. All camera-related changes are scoped to `backend/apps/cameras/` (Django app) and `frontend/src/` (React SPA). The `core/fields.py` module hosts `EncryptedTextField` for reuse across apps. The `docs/` directory mirrors both trees per constitution §docs/.

## Complexity Tracking

> No constitution violations. All patterns are justified by existing codebase conventions.

| Item | Note |
|------|------|
| No violations | All selected patterns (service layer, custom field, Celery tasks) align with existing project conventions and constitution principles. |

## Key Technical Decisions

| # | Decision | Choice | Rationale | Alternatives Rejected |
|---|----------|--------|-----------|----------------------|
| 1 | go2rtc HTTP client | `httpx` ≥0.27 | Async + sync modes, connection pooling, type hints, timeout granularity | `requests` (no async), `aiohttp` (async-only) |
| 2 | RTSP probe library | Raw TCP socket (Stage 1) + `cv2.VideoCapture` (Stages 2–3) | Zero new dependencies — OpenCV already installed via `ultralytics`. Same code path as detection pipeline. | PyAV (new ~15 MB dep, FFmpeg version conflicts), `ffprobe` subprocess (credentials in process args, fragile stderr parsing) |
| 3 | RTSP URL encryption | Custom `EncryptedTextField` with `cryptography.fernet.Fernet` | No maintained Django 5.x library. `cryptography` already installed transitively via `channels`. ~40 lines of code. Multi-key rotation via `MultiFernet`. | `django-fernet-fields` (unmaintained, Django 3+ incompatible), `django-encrypted-model-fields` (dead since 2020) |
| 4 | Encrypted field uniqueness | HMAC-SHA256 blind index (`connection_url_hash`) computed in `model.save()` | Enables `UniqueConstraint` without decrypting. Deterministic hash. Key derived from `FIELD_ENCRYPTION_KEY`. | Full-table decrypt scan (O(n) per query, unacceptable) |
| 5 | WebSocket broadcasting | Service-layer `group_send` via `async_to_sync` in `CameraService._broadcast_status` | Explicit, testable, co-located with business logic. Covers all code paths (views, Celery tasks, admin actions). | Django signals (implicit, fires on every save), model `save()` override (same issue) |
| 6 | Health-check mechanism | Celery beat periodic task (`stream_health_check`, 15s interval) | Decoupled from Django process lifecycle. Survives Django restarts. Detects go2rtc restarts (missing streams → auto-re-register). | go2rtc webhooks (not supported), long-polling (wasteful) |
| 7 | WHEP routing | nginx reverse proxy: `/whep/{camera_id}/` → `go2rtc:1984/api/webrtc?src=camera_{camera_id}` | Leverages existing nginx + `auth_request` middleware. Same-origin eliminates CORS. Session cookie sent automatically. | Direct go2rtc access (no auth, CORS issues), Django proxy view (unnecessary overhead) |
| 8 | Auto-reconnection backoff | Exponential backoff 2s → 4s → 8s, max 3 retries (~14s total) | Prevents thundering herd on transient failures. Fast enough for real-time monitoring. Matches WHEP client-side retry params. | Fixed interval (no backoff = thundering herd), unlimited retries (resource exhaustion) |

## New Dependencies

| Package | Version | Purpose | Justification |
|---------|---------|---------|---------------|
| `httpx` | ≥0.27 | HTTP client for go2rtc REST API (stream registration, health-check) | Only new backend dependency. Needed for go2rtc communication with proper timeout/pooling support. |

**Note**: `cryptography` (for Fernet encryption) and `opencv-python` (for RTSP probe) are already installed transitively via `channels` and `ultralytics` respectively. No new frontend dependencies.

## Clarification Updates (from /speckit.clarify session)

These 5 clarifications were resolved after the initial plan and are now incorporated:

| # | Finding | Resolution | Impact on Plan |
|---|---------|------------|----------------|
| C1 | `ConnectionEvent.camera` on_delete was `CASCADE` | Changed to `SET_NULL` (nullable) | data-model.md entity table updated. Migration 3 uses `SET_NULL`. Events preserved after camera deletion. |
| H1 | State machine had phantom `connected → error` arrow | Removed. Only path: `connected → reconnecting → error` (after 3 retries) | data-model.md state diagram updated. No direct connected→error transition. |
| H2 | FR-002 and FR-012 duplicated three-stage probe text | Split: FR-002 = server-side probe, FR-012 = client-side format validation + rtsps:// acceptance only | No plan change needed — probe implementation was already under FR-002. Frontend validation clearly separate. |
| M3 | "Poor Connection" edge case unresolved | Deferred / out of scope — go2rtc doesn't expose bitrate/packet-loss metrics | No quality degradation monitoring in this feature. Health-check covers complete stream loss only. |
| M6 | SC-005 and SC-007 unmeasurable | Reclassified as aspirational with testable proxy criteria | SC-005: unit tests verify all error subclasses include `error_code`, `stage`, `detail`. SC-007: E2E test verifies ≤3-action connect flow. Neither blocks delivery. |

## Phase 0 Artifacts (Research)

| Artifact | Topics Covered |
|----------|---------------|
| [research.md](research.md) | go2rtc REST API (stream management, health-check strategy, error matrix, naming convention); RTSP URL encryption (custom EncryptedTextField, blind index, key rotation, URL masking); WebSocket broadcasting (service-layer group_send, per-camera + global groups, payload format); WHEP proxy (nginx fix, frontend changes, CORS elimination, ICE transport) |
| [research-rtsp-probe.md](research-rtsp-probe.md) | Library comparison (raw socket, ffprobe, PyAV, cv2.VideoCapture); recommended approach (raw TCP + cv2); error classification matrix; implementation patterns; timeout handling |

## Phase 1 Artifacts (Design & Contracts)

| Artifact | Contents |
|----------|----------|
| [data-model.md](data-model.md) | 3 entities (CameraSource modified, StreamRegistration new, ConnectionEvent new); ER diagram; state machine (8 transitions, no direct connected→error); 5 migrations; frontend type additions |
| [contracts/rest-api.md](contracts/rest-api.md) | 11 REST endpoints: CRUD + connect/disconnect + events + admin dashboard/force-disconnect/connect-on-behalf/relay-health; error response shape with `error_code`, `stage`, `detail` |
| [contracts/websocket.md](contracts/websocket.md) | 2 WS channels (per-camera `camera_{id}`, global `cameras_all`); `camera.status` message format; `camera.relay_health` admin message; `useCameraStatus` hook interface |
| [contracts/whep-signaling.md](contracts/whep-signaling.md) | WHEP endpoint `/whep/{camera_id}/`; nginx proxy config (go2rtc:1984); SDP request/response format; ICE transport on port 8555; `useWhepClient` hook interface |
| [quickstart.md](quickstart.md) | Prerequisites, environment setup (Docker, backend, Celery, frontend), verification steps (connect, error handling, real-time status, auto-reconnect), test commands, new env vars, new deps, configuration constants |
