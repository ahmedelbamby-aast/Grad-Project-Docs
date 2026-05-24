# Implementation Plan: Exam Monitoring Dashboard

**Branch**: `001-exam-monitor-dashboard` | **Date**: 2026-03-02 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-exam-monitor-dashboard/spec.md`

## Summary

Build a full-stack exam monitoring application enabling university instructors and admins to monitor exam halls in real-time via camera feeds with YOLO-based detection overlays, anomaly alerting, triage workflows, session recording/playback, and comprehensive admin management. The system uses Django 5.x + Channels + DRF (backend), React 19.2 + TypeScript + Vite + Zustand (frontend), PostgreSQL 16 (database), Redis 7 (cache/broker/channel layer), go2rtc (WebRTC video relay), and Ultralytics 8.x (YOLO inference). Three themes (Black+Purple, Full Black, White) ship from Day 1.

## Implementation Updates (2026-03-02)

### Migration graph and database bootstrap (completed)

- Generated initial schema migrations for model apps via `python manage.py makemigrations`.
- Fixed `accounts` migration ordering to prevent pre-schema data seeding errors:
    - schema migration: `accounts.0003_initial`
    - seed migration: `accounts.0004_seed_builtin_roles`
- Updated seed migration dependency so built-in roles are inserted only after `Role`/`User` tables exist.
- Verified migration health with:
    - `python manage.py migrate`
    - `python manage.py showmigrations accounts`
    - shell verification returning `['Admin', 'Instructor']`

### Documentation synchronization (completed)

- Updated [quickstart.md](quickstart.md), [../../../backend/README.md](../../../backend/README.md), and [../../../README.md](../../../README.md) to match the implemented workflow.
- Removed stale command references that no longer exist in code (`seed_roles`, `seed_dev_data`).
- Documented that built-in roles are migration-seeded during `migrate` using `accounts.0004_seed_builtin_roles`.
- Added Celery Beat startup command to operational runbooks where applicable.

## Technical Context

**Language/Version**: Python 3.11+ (backend), TypeScript ~5.9.3 (frontend)
**Primary Dependencies**: Django 5.x, Django Channels 4.x, DRF 3.15+, Celery 5.x (backend); React 19.2, Vite 7.3, Zustand 5.x, react-router-dom 7.x, axios (frontend); go2rtc ≥1.9 (video relay); Ultralytics 8.x (YOLO inference)
**Storage**: PostgreSQL 16 (primary DB), Redis 7 (channel layer, cache, Celery broker), local filesystem (video recordings)
**Testing**: pytest + pytest-django + pytest-asyncio + factory_boy + pytest-cov (backend); Vitest + @testing-library/react + msw (frontend unit/integration); Playwright (E2E)
**Target Platform**: Windows + Linux (cross-platform), modern browsers (Chrome, Firefox, Edge — latest 2 versions)
**Project Type**: Web application (frontend SPA + backend API + sidecar video relay)
**Performance Goals**: ≥15 FPS camera feeds, <1s camera-to-screen latency, <200ms UI interactions, ≤300ms theme switching, <3s initial page load, ≤500ms seek on recording playback, ~8–10 WebSocket messages/sec (micro-batched from ~120 frames/sec)
**Constraints**: 1–8 cameras per instructor, up to 20 concurrent instructors, session cookie auth (no JWT), WCAG 2.1 AA compliance, auto-recovery within 30s
**Scale/Scope**: 16 data entities, ~66 REST endpoints, 6 WebSocket channels (13 message types), 17 pages, 3 themes, 2 built-in roles + custom roles

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| # | Principle | Gate Question | Pass? |
|---|-----------|---------------|-------|
| I | Code Quality Excellence | Does the design follow SOLID, use DI, config-driven behavior, package-per-module structure? | ✅ YES — Django apps as packages (11 apps), DRF ViewSets, Strategy/Factory/Observer patterns (R-005), Pydantic config validation, DI via constructors. Frontend: Zustand stores separated from components, hooks for DI, CSS custom properties for config-driven theming. Functions ≤30 lines enforced. |
| II | Testing Standards (NON-NEGOTIABLE) | Is TDD planned? Are unit/integration/system test directories defined? ≥80% coverage target set? | ✅ YES — Three-phase TDD (R-008): backend `tests/unit/<app>/`, `tests/integration/`, `tests/system/`; frontend `tests/unit/`, `tests/integration/`, `tests/e2e/`. ≥80% coverage gate via pytest-cov + vitest coverage. Red-Green-Refactor cycle enforced. |
| III | User Experience Consistency | Are loading states, error messages, keyboard nav, and 1080p layout addressed? | ✅ YES — FR-033 (component library), FR-034 (loading states), FR-035 (keyboard nav), FR-036 (React error boundaries + actionable messages), FR-037 (1920×1080 minimum). ≤200ms micro-interactions. |
| IV | Security Standards | Are secrets in .env, paths sanitized, PII anonymized, RBAC defined, audit logging planned? | ✅ YES — .env for secrets, PathSanitizer for file paths, session-based auth with secure cookies (R-010), custom Role model with page+action permissions (R-014), AuditLogEntry append-only model (R-015, FR-069). RTSP URL allowlist validation. Log redaction for PII. |
| V | Amazing UI Design & UX | Are 3 themes (Black+Purple, Full Black, White) planned? WCAG 2.1 AA? Component library? | ✅ YES — Three themes via CSS custom properties (R-009) with design tokens, WCAG 2.1 AA contrast validated, reusable component library (FR-033), instant theme switch ≤300ms (FR-022), smooth dark/light transitions. |
| VI | Performance Requirements | Are threading (I/O) and multiprocessing (CPU) assigned? ≥15 FPS target? Lazy loading? | ✅ YES — go2rtc H.264 passthrough for ≥15 FPS (R-011), multi-threading for I/O (RTSP capture, file writes), multi-processing for CPU (YOLO inference), lazy model loading per session (R-005), frontend lazy route loading, WebSocket micro-batching (100ms windows). |
| VII | Delivery Sign-Off Protocol | Is a delivery checklist planned? Issue scan (lint/type/security/spec) → fix → test → sign-off? | ✅ YES — 7-step sign-off protocol per constitution. ESLint + Ruff (lint), tsc + mypy (type), npm audit + pip-audit (security), spec compliance check. Delivery gate: all three test phases green. Checklists in `specs/001-exam-monitor-dashboard/checklists/`. |
| — | Documentation & Commits | Is README.md present? Per-module docs planned? Mermaid diagrams listed? Conventional Commits enforced? | ✅ YES — Root README.md, backend/README.md, frontend/README.md, and quickstart.md are maintained and updated. Mermaid ER diagram in data-model.md and architecture/data-flow diagrams are documented. Conventional Commits (`type(scope): desc`). |

> **Blocking**: All gates pass. No violations requiring justification.

## Project Structure

### Documentation (this feature)

```text
specs/001-exam-monitor-dashboard/
├── plan.md              # This file
├── research.md          # Phase 0 output — 17 research decisions (R-001 through R-017)
├── data-model.md        # Phase 1 output — 16 entities with full schemas, state machines, indexes
├── quickstart.md        # Phase 1 output — dev environment setup guide
├── contracts/
│   ├── rest-api.md      # Phase 1 output — ~66 REST endpoint contracts
│   └── websocket-api.md # Phase 1 output — 6 WebSocket channels, 13 message type contracts
├── tasks.md             # Phase 2 output (unified backend + frontend tasks)
├── tasks-frontend.md    # Phase 2 output (granular frontend tasks, TF-prefixed IDs)
└── checklists/
    ├── comprehensive.md
    ├── frontend.md
    ├── requirements.md
    ├── setup.md
    └── ux.md
```

### Source Code (repository root)

```text
backend/
├── config/                    # Django project settings
│   ├── __init__.py
│   ├── settings/
│   │   ├── base.py            # Shared settings (DB, Redis, Channels, DRF, auth)
│   │   ├── development.py     # Dev overrides (DEBUG, CORS, logging)
│   │   └── production.py      # Prod overrides (security, static files)
│   ├── urls.py                # Root URL configuration (/api/v1/, /ws/)
│   ├── asgi.py                # ASGI application (HTTP + WebSocket routing)
│   └── wsgi.py                # WSGI fallback
├── apps/
│   ├── accounts/              # User, Role, auth, RBAC
│   │   ├── models.py          # User (extends AbstractUser), Role
│   │   ├── serializers.py     # LoginSerializer, UserSerializer, RoleSerializer
│   │   ├── views.py           # LoginView, LogoutView, MeView, UserViewSet, RoleViewSet
│   │   ├── services.py        # AuthService, RoleService
│   │   ├── permissions.py     # HasPageAccess, HasActionPermission
│   │   ├── signals.py         # Audit log signals for auth events
│   │   ├── urls.py
│   │   └── README.md
│   ├── cameras/               # CameraSource, go2rtc integration
│   │   ├── models.py          # CameraSource
│   │   ├── serializers.py
│   │   ├── views.py           # CameraViewSet (CRUD + connect/disconnect actions)
│   │   ├── services.py        # CameraService, Go2rtcClient
│   │   ├── urls.py
│   │   └── README.md
│   ├── detections/            # DetectionFrame, Detection, PyramidPrediction
│   │   ├── models.py          # DetectionFrame, Detection, PyramidPrediction
│   │   ├── serializers.py
│   │   ├── consumers.py       # DetectionConsumer (WebSocket)
│   │   ├── services.py        # DetectionService (micro-batching)
│   │   ├── routing.py         # WebSocket URL routing
│   │   ├── urls.py
│   │   └── README.md
│   ├── anomalies/             # AnomalyEvent, AnomalyNote, triage
│   │   ├── models.py          # AnomalyEvent, AnomalyNote
│   │   ├── serializers.py
│   │   ├── views.py           # AnomalyViewSet, triage actions
│   │   ├── consumers.py       # AnomalyConsumer (WebSocket for alerts)
│   │   ├── services.py        # TriageService (first-write-wins), RuleEngine
│   │   ├── routing.py
│   │   ├── urls.py
│   │   └── README.md
│   ├── sessions/              # MonitoringSession, InstructorComment
│   │   ├── models.py          # MonitoringSession, InstructorComment
│   │   ├── serializers.py
│   │   ├── views.py           # SessionViewSet, CommentViewSet
│   │   ├── services.py        # SessionService (lifecycle, grace period)
│   │   ├── urls.py
│   │   └── README.md
│   ├── recordings/            # Recording, video file management
│   │   ├── models.py          # Recording (M2M with MonitoringSession)
│   │   ├── serializers.py
│   │   ├── views.py           # RecordingViewSet (playback, metadata)
│   │   ├── services.py        # RecordingService (deduplication, storage monitoring)
│   │   ├── urls.py
│   │   └── README.md
│   ├── exams/                 # Exam, ExamStudent, Room
│   │   ├── models.py          # Exam, ExamStudent, Room
│   │   ├── serializers.py
│   │   ├── views.py           # ExamViewSet, RoomViewSet, ExamStudentViewSet
│   │   ├── services.py        # ExamService (scheduling), RoomService
│   │   ├── urls.py
│   │   └── README.md
│   ├── exports/               # SessionExport
│   │   ├── models.py          # SessionExport
│   │   ├── serializers.py
│   │   ├── views.py           # ExportViewSet
│   │   ├── services.py        # ExportService
│   │   ├── tasks.py           # Celery tasks (ZIP generation)
│   │   ├── urls.py
│   │   └── README.md
│   ├── audit/                 # AuditLogEntry
│   │   ├── models.py          # AuditLogEntry (append-only, immutable)
│   │   ├── serializers.py
│   │   ├── views.py           # AuditLogViewSet (read-only)
│   │   ├── services.py        # AuditService.log()
│   │   ├── middleware.py       # AuditMiddleware (auth event capture)
│   │   ├── urls.py
│   │   └── README.md
│   ├── health/                # System health monitoring
│   │   ├── views.py           # HealthCheckView (/health endpoint)
│   │   ├── services.py        # Subsystem health checkers
│   │   ├── consumers.py       # HealthConsumer (WebSocket)
│   │   ├── routing.py
│   │   ├── urls.py
│   │   └── README.md
│   └── pipeline/              # YOLO inference pipeline
│       ├── base.py            # BasePyramidLayer abstract interface
│       ├── layers/            # One module per YOLO layer (Strategy pattern)
│       │   ├── base_detector.py
│       │   ├── posture.py
│       │   ├── horizontal_gaze.py
│       │   ├── depth_gaze.py
│       │   └── vertical_gaze.py
│       ├── rule_engine.py     # Config-driven anomaly rule evaluation
│       ├── tracker.py         # BoT-SORT/ByteTrack wrapper
│       ├── services.py        # Pipeline orchestration
│       └── README.md
├── core/                      # Shared utilities
│   ├── security.py            # PathSanitizer, SecurityVault
│   ├── logger.py              # Structured logging (zero print())
│   ├── exceptions.py          # Base custom exception hierarchy
│   ├── pagination.py          # DRF pagination classes
│   ├── middleware.py           # Shared Django middleware
│   └── README.md
├── manage.py
├── requirements.in            # Version ranges (pip-tools workflow)
├── requirements.txt           # Pinned versions (pip-compile output)
├── .env.example
├── pytest.ini
├── README.md
└── tests/
    ├── unit/                  # Mirrors apps/ structure
    ├── integration/           # Cross-app integration tests
    └── system/                # End-to-end system tests

frontend/                      # Already initialized
├── src/
│   ├── api/                   # Axios-based API client modules
│   │   ├── client.ts          # Axios instance (base URL, interceptors, cookie auth)
│   │   ├── auth.ts            # POST /auth/login, /auth/logout, GET /auth/me
│   │   ├── cameras.ts         # Camera CRUD + connect/disconnect
│   │   ├── sessions.ts        # Session CRUD + comments
│   │   ├── detections.ts      # Detection frame queries (playback)
│   │   ├── anomalies.ts       # Anomaly CRUD + triage actions
│   │   ├── admin.ts           # Admin management endpoints (users, roles, exams, rooms)
│   │   └── dashboard.ts       # Dashboard aggregation endpoints
│   ├── components/
│   │   ├── anomaly/           # AlertHistory, AnomalyAlert, TriageActions
│   │   ├── auth/              # RouteGuards (role-based)
│   │   ├── camera/            # BoundingBoxCanvas, CameraFeed, CameraGrid, ClassFilterPanel
│   │   ├── detection/         # PredictionsPanel, StudentCard
│   │   ├── layout/            # AppLayout, Sidebar, Header
│   │   └── ui/                # Button, Card, Modal, StatusBadge, Toggle, etc.
│   ├── hooks/
│   │   ├── useAuth.ts         # Auth state + role checking
│   │   ├── useWebSocket.ts    # Base WebSocket hook (reconnect logic)
│   │   ├── useDetectionSocket.ts  # Detection feed subscription
│   │   ├── useAnomalySocket.ts    # Anomaly alert subscription
│   │   └── useWhepClient.ts   # WebRTC WHEP client for go2rtc
│   ├── pages/                 # 17 route pages
│   │   ├── LoginPage.tsx
│   │   ├── DashboardPage.tsx  # Admin Dashboard (FR-046)
│   │   ├── CameraFeedPage.tsx # Live feeds + overlays
│   │   ├── CameraListPage.tsx
│   │   ├── PredictionsPage.tsx
│   │   ├── SessionListPage.tsx
│   │   ├── SessionDetailPage.tsx
│   │   ├── RecordingsPage.tsx
│   │   ├── AnomalyListPage.tsx
│   │   ├── ExamBoardPage.tsx  # Exam Board & session start (US8)
│   │   ├── HealthDashboardPage.tsx # System health monitoring (FR-038)
│   │   ├── UserManagementPage.tsx # Admin user CRUD (US10)
│   │   ├── RoleManagementPage.tsx # Admin role/exam/room management (US10)
│   │   ├── RecordingReviewPage.tsx # Admin recording review (US11)
│   │   ├── AuditLogPage.tsx   # Audit log viewer (US11)
│   │   ├── SettingsPage.tsx
│   │   └── NotFoundPage.tsx
│   ├── stores/                # Zustand state management
│   │   ├── authStore.ts       # Auth state, role permissions, theme pref
│   │   ├── cameraStore.ts     # Camera connection state
│   │   ├── detectionStore.ts  # Real-time detection frames per camera
│   │   └── anomalyStore.ts    # Anomaly alerts, triage state
│   ├── styles/
│   │   ├── global.css         # Global resets + base styles
│   │   ├── tokens.css         # Design tokens (CSS custom properties)
│   │   └── themes/            # black-purple.css, full-black.css, white.css
│   └── types/
│       ├── api.ts             # API response type definitions
│       ├── predictions.ts     # Pyramid prediction types
│       └── websocket.ts       # WebSocket message types
├── tests/
│   ├── unit/
│   │   ├── components/
│   │   └── stores/
│   ├── integration/
│   └── e2e/                   # Playwright E2E tests
├── package.json
├── vite.config.ts
├── tsconfig.json
├── vitest.setup.ts
├── playwright.config.ts
└── README.md

# Infrastructure (repository root)
docker-compose.dev.yml         # PostgreSQL 16 + Redis 7 + go2rtc
go2rtc.yaml                    # Camera stream configuration
.env.example                   # Environment variables template
README.md                      # Root project README with architecture diagram
```

**Structure Decision**: Web application with separate `backend/` (Django 5.x) and `frontend/` (React 19.2 SPA) directories. Both backend and frontend are implemented and actively maintained. Infrastructure services (PostgreSQL, Redis, go2rtc) run via Docker Compose sidecar.

## Complexity Tracking

> All constitution gates pass. Complexity is justified by spec requirements, not over-engineering.

| Aspect | Why This Scale | Simpler Alternative Rejected Because |
|--------|---------------|-------------------------------------|
| 16 data entities | Spec mandates 16 distinct key entities for audit trail, triage workflow, multi-instructor sessions, recording deduplication, RBAC, exam scheduling | Fewer entities violate spec FRs (e.g., no AnomalyNote = violates FR-043, no AuditLogEntry = violates FR-069, no Room = violates FR-056) |
| 11 Django apps | One-app-per-domain keeps SRP (Constitution Principle I), enables independent testing (Principle II), matches package-per-module mandate | Collapsing apps violates SRP and makes testing harder; constitution mandates package-per-module |
| go2rtc sidecar | WebRTC requires dedicated media server; Django cannot serve WebRTC streams | MJPEG/JSMpeg alternatives fail <1s latency for 8 cameras (R-011 analysis) |
| M2M Session↔Recording | Recording deduplication per spec (Clarification Round 4, Q2): shared cameras produce one recording | Direct session FK would duplicate recordings when multiple instructors view same camera |

## Phase 0 Artifacts

- [research.md](research.md) — 17 research decisions (R-001 through R-017) resolving all technical unknowns

## Phase 1 Artifacts

- [data-model.md](data-model.md) — 16 entities with full field schemas, validation rules, state machines, indexes, retention policies, ER diagram
- [contracts/rest-api.md](contracts/rest-api.md) — ~66 REST endpoint contracts with request/response JSON schemas
- [contracts/websocket-api.md](contracts/websocket-api.md) — 6 WebSocket channels, 13 message type contracts with channel group routing
- [quickstart.md](quickstart.md) — Development environment setup guide (Docker Compose, backend, frontend, go2rtc)
