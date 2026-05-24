# Tasks: Exam Monitoring Dashboard

**Input**: Design documents from `/specs/001-exam-monitor-dashboard/`
**Prerequisites**: plan.md (required), spec.md (required), research.md, data-model.md, contracts/, quickstart.md

**Tests**: Included. Spec + constitution require non-negotiable testing with unit/integration/system validation (R-008: ≥80% coverage).

**Organization**: Tasks are grouped by user story for independent implementation and testing of each story.

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Initialize backend structure, frontend routing, Docker services, and shared dev infrastructure.

- [X] T001 Create backend project skeleton with 11 Django app packages (accounts, cameras, detections, anomalies, sessions, recordings, exams, exports, audit, health, pipeline) and core/ directory in backend/
- [X] T002 Create Python dependency manifests (Django 5.x, Channels 4.x, DRF 3.15+, Celery 5.x, pytest, Ultralytics 8.x, factory_boy) in backend/requirements.in and backend/requirements.txt
- [X] T003 [P] Configure Django base settings (installed apps, middleware stack, database, templates, static files) in backend/config/settings/base.py
- [X] T004 [P] Configure development settings (DEBUG, CORS, local database URL) in backend/config/settings/development.py
- [X] T005 [P] Configure production settings (security headers, allowed hosts, HTTPS) in backend/config/settings/production.py
- [X] T006 Create Docker Compose services for PostgreSQL 16, Redis 7, and go2rtc ≥1.9 in docker-compose.dev.yml
- [X] T007 [P] Create go2rtc stream relay configuration in go2rtc.yaml
- [X] T008 Create environment template with all required variables in .env.example
- [X] T009 [P] Configure pytest with coverage baseline, async support, and factory_boy in backend/pytest.ini
- [X] T010 [P] Configure Celery app with Redis broker, result backend, and beat schedule in backend/config/celery.py
- [X] T011 Configure nginx reverse proxy with WHEP path (`/whep/{camera_id}/`) proxying to go2rtc, Django auth subrequest validation, `/api/v1/` proxy to Django, and WebSocket upgrade support in nginx.conf
- [X] T012 [P] Configure frontend React Router with lazy-loaded routes for all pages in frontend/src/App.tsx

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core auth, RBAC, auditing, routing, shared primitives, and middleware required by all user stories.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [X] T013 Implement Role model and custom User model (extending AbstractUser with must_change_password flag, theme_preference, is_active) in backend/apps/accounts/models.py
- [X] T014 Create seed migration for built-in Admin and Instructor roles (is_builtin=True, non-deletable) in backend/apps/accounts/migrations/
- [X] T015 [P] Implement AuditLogEntry append-only immutable model (23 action types, JSON details field) in backend/apps/audit/models.py
- [X] T016 [P] Implement Room base model (name, building, capacity, camera references) in backend/apps/exams/models.py
- [X] T017 Implement auth serializers (login request/response, user profile, password change, must_change_password field) in backend/apps/accounts/serializers.py
- [X] T018 Implement AuthService (login/logout/session management) and RoleService (permission resolution) in backend/apps/accounts/services.py
- [X] T019 Implement page-level and action-level DRF permission classes (IsAdmin, HasPageAccess, HasActionPermission) in backend/apps/accounts/permissions.py
- [X] T020 Implement AuditService.log() utility for structured audit event logging in backend/apps/audit/services.py
- [X] T021 Implement AuditMiddleware for automatic auth event capture (login, logout, login_failed) in backend/apps/audit/middleware.py
- [X] T022 Configure DRF defaults (session auth, JSON renderer, custom exception handler, pagination) in backend/config/settings/base.py
- [X] T023 Configure API URL routing with versioned namespace (/api/v1/) and app inclusion in backend/config/urls.py
- [X] T024 Configure ASGI application with Channels, Redis channel layer, and root WebSocket URL router in backend/config/asgi.py
- [X] T025 [P] Implement base custom exception hierarchy (ServiceError, NotFoundError, ConflictError, PermissionError) in backend/core/exceptions.py
- [X] T026 [P] Implement structured logging utilities (zero print(), component context) in backend/core/logger.py
- [X] T027 [P] Implement shared DRF pagination classes in backend/core/pagination.py
- [X] T028 Implement ForcePasswordChangeMiddleware (redirects users with must_change_password=true to password-change endpoint) in backend/core/middleware.py
- [X] T029 Implement ChangePasswordPage with forced redirect handling and flag clearance in frontend/src/pages/ChangePasswordPage.tsx
- [X] T030 Implement frontend RouteGuards with role-based access re-validation on every navigation (FR-053) in frontend/src/components/auth/RouteGuards.tsx

**Checkpoint**: Foundation complete — all user stories can now begin implementation.

---

## Phase 3: User Story 1 — User Login & Role-Based Landing (Priority: P1) 🎯 MVP

**Goal**: Login/logout/profile flows with role-based redirect (instructors → Exam Board, admins → Admin Dashboard) and secure session handling.

**Independent Test**: Instructor and admin can both sign in and land on role-specific pages; invalid login shows generic error; logout invalidates session.

- [X] T031 [P] [US1] Implement auth API client methods (login, logout, me, password change) in frontend/src/api/auth.ts
- [X] T032 [US1] Implement login, logout, me, and password-change view endpoints in backend/apps/accounts/views.py
- [X] T033 [US1] Register auth URL routes in backend/apps/accounts/urls.py
- [X] T034 [US1] Implement auth state management with role-based redirects in frontend/src/stores/authStore.ts
- [X] T035 [US1] Build LoginPage with form validation, error states, and role-appropriate redirect in frontend/src/pages/LoginPage.tsx
- [X] T036 [P] [US1] Apply login page visual design (Black+Purple theme, full-bleed gradient, centered card layout) in frontend/src/pages/LoginPage.module.css
- [X] T037 [US1] Implement SettingsPage with profile info, password change form, and theme preference in frontend/src/pages/SettingsPage.tsx
- [X] T038 [P] [US1] Add backend integration tests for all auth endpoints in backend/tests/integration/test_auth_api.py
- [X] T039 [P] [US1] Add frontend unit tests for login flow and role redirect in frontend/tests/unit/components/auth/LoginPage.test.tsx

**Checkpoint**: Authentication works end-to-end; instructors and admins land on correct pages.

---

## Phase 4: User Story 2 — Live Camera Feed Monitoring (Priority: P1)

**Goal**: Connect cameras via RTSP, render real-time video via WebRTC WHEP, display bounding box overlays with color-coded classes and persistent tracking IDs.

**Independent Test**: A valid RTSP camera connects, live feed appears with <1s latency, bounding boxes render with tracking IDs, reconnection works on disconnect.

- [X] T040 [US2] Implement CameraSource model with status state machine (disconnected→connected→reconnecting→error) in backend/apps/cameras/models.py
- [X] T041 [US2] Implement camera serializers in backend/apps/cameras/serializers.py
- [X] T042 [US2] Implement go2rtc camera service client (register/unregister streams) in backend/apps/cameras/services.py
- [X] T043 [US2] Implement camera REST endpoints (list, create, update, delete, connect, disconnect) and register URL routes in backend/apps/cameras/views.py and backend/apps/cameras/urls.py
- [X] T044 [US2] Implement camera status WebSocket consumer (camera.status events) and routing in backend/apps/cameras/consumers.py and backend/apps/cameras/routing.py
- [X] T045 [US2] Implement DetectionFrame and Detection models with time-based partitioning setup in backend/apps/detections/models.py
- [X] T046 [US2] Implement detection serializers in backend/apps/detections/serializers.py
- [X] T047 [US2] Implement detection WebSocket consumer (detection.frame, prediction.update) and routing in backend/apps/detections/consumers.py and backend/apps/detections/routing.py
- [X] T048 [US2] Register detection URL routes in backend/apps/detections/urls.py
- [X] T049 [US2] Implement camera API client methods in frontend/src/api/cameras.ts
- [X] T050 [US2] Implement camera connection state store with reconnection logic in frontend/src/stores/cameraStore.ts
- [X] T051 [US2] Implement detection frame store and useDetectionSocket hook in frontend/src/stores/detectionStore.ts and frontend/src/hooks/useDetectionSocket.ts
- [X] T052 [US2] Implement WebRTC WHEP client hook for go2rtc streams in frontend/src/hooks/useWhepClient.ts
- [X] T053 [US2] Implement CameraFeed component with WHEP video player and reconnection UI in frontend/src/components/camera/CameraFeed.tsx
- [X] T054 [US2] Implement BoundingBoxCanvas overlay rendering (color-coded by class, persistent tracking IDs) in frontend/src/components/camera/BoundingBoxCanvas.tsx
- [X] T055 [US2] Implement responsive CameraGrid layout (auto-adjust: 1→full, 2→1×2, 3-4→2×2, 5-6→2×3, 7-8→2×4) in frontend/src/components/camera/CameraGrid.tsx
- [X] T056 [US2] Integrate camera grid, feeds, and overlays in CameraFeedPage in frontend/src/pages/CameraFeedPage.tsx
- [X] T057 [P] [US2] Add backend integration tests for camera lifecycle in backend/tests/integration/test_camera_lifecycle.py
- [X] T058 [P] [US2] Add frontend component tests for CameraFeed in frontend/tests/unit/components/camera/CameraFeed.test.tsx

**Checkpoint**: Live camera feeds render with detection overlays; WebRTC streaming works end-to-end.

---

## Phase 5: User Story 8 — Exam Board & Session Start (Priority: P1)

**Goal**: Instructor sees scheduled exams with details, countdown timer behavior, auto-start at exam time, session lifecycle management with grace period.

**Independent Test**: Instructor starts session from Exam Board and transitions to Camera Feed with room cameras loaded; countdown auto-start works; active session shows "In Progress" badge.

- [X] T059 [US8] Implement Exam and ExamStudent models (subject details, M2M instructor assignment, status state machine) in backend/apps/exams/models.py
- [X] T060 [US8] Implement MonitoringSession model with lifecycle state machine (active→completed|interrupted) in backend/apps/sessions/models.py
- [X] T061 [US8] Implement exam serializers (exam detail, student roster) in backend/apps/exams/serializers.py
- [X] T062 [US8] Implement session serializers in backend/apps/sessions/serializers.py
- [X] T063 [US8] Implement SessionService with lifecycle management, auto-start logic, and 5-minute grace period in backend/apps/sessions/services.py
- [X] T064 [US8] Implement exam board endpoints (GET /my/, POST /{id}/start-session/) and register routes in backend/apps/exams/views.py and backend/apps/exams/urls.py
- [X] T065 [US8] Implement session create/end/list endpoints and register routes in backend/apps/sessions/views.py and backend/apps/sessions/urls.py
- [X] T066 [US8] Implement exam board and session API client methods in frontend/src/api/sessions.ts
- [X] T067 [US8] Implement ExamBoardPage with countdown timers, auto-start, active session display (FR-072), and "Return to Session" link in frontend/src/pages/ExamBoardPage.tsx
- [X] T068 [P] [US8] Add integration tests for exam board session start flow in backend/tests/integration/test_examboard_session.py

**Checkpoint**: P1 MVP complete — login, live camera feeds, and exam board all work independently.

---

## Phase 6: User Story 3 — Bounding Box Class Filtering (Priority: P2)

**Goal**: Toggle bounding box visibility by detection class with <200ms response time.

**Independent Test**: Disable "Teacher" class and only teacher boxes disappear within 200ms; "Show All" re-enables all classes.

- [X] T069 [US3] Implement ClassFilterPanel component with per-class toggle switches in frontend/src/components/camera/ClassFilterPanel.tsx
- [X] T070 [US3] Add filter state and session persistence to detection store in frontend/src/stores/detectionStore.ts
- [X] T071 [US3] Apply class filters in BoundingBoxCanvas render pipeline (<200ms target) in frontend/src/components/camera/BoundingBoxCanvas.tsx
- [X] T072 [US3] Wire ClassFilterPanel into CameraFeedPage layout in frontend/src/pages/CameraFeedPage.tsx
- [X] T073 [P] [US3] Add unit tests for filter toggles and render behavior in frontend/tests/unit/components/camera/ClassFilterPanel.test.tsx

---

## Phase 7: User Story 4 — Model Predictions & Anomaly Display (Priority: P2)

**Goal**: Show per-student pyramid predictions, anomaly alerts with severity, triage actions (acknowledge/dismiss/note), first-write-wins concurrency, and full detection pipeline.

**Independent Test**: Predictions stream updates live; anomaly alert appears with severity; triage acknowledge/dismiss transitions are enforced with first-write-wins.

- [X] T074 [US4] Implement PyramidPrediction model (posture, gaze layers, constraint violation flag) in backend/apps/detections/models.py
- [X] T075 [US4] Implement AnomalyEvent and AnomalyNote models (severity, status state machine, prediction snapshot) in backend/apps/anomalies/models.py
- [X] T076 [P] [US4] Implement BasePyramidLayer abstract interface in backend/apps/pipeline/base.py
- [X] T077 [P] [US4] Implement pyramid detection layers (posture, horizontal_gaze, depth_gaze, vertical_gaze) using Strategy pattern in backend/apps/pipeline/layers/
- [X] T078 [US4] Implement config-driven anomaly rule engine (duration thresholds + multi-layer combinations) in backend/apps/pipeline/rule_engine.py
- [X] T079 [US4] Implement BoT-SORT/ByteTrack tracker wrapper with re-identification window in backend/apps/pipeline/tracker.py
- [X] T080 [US4] Implement pipeline orchestration service (frame → detect → track → predict → anomaly check) in backend/apps/pipeline/services.py
- [X] T081 [US4] Implement TriageService with first-write-wins concurrency logic and WebSocket broadcast in backend/apps/anomalies/services.py
- [X] T082 [US4] Implement anomaly serializers (event, note, triage action) in backend/apps/anomalies/serializers.py
- [X] T083 [US4] Implement anomaly REST endpoints (list, acknowledge, dismiss, notes CRUD) and register routes in backend/apps/anomalies/views.py and backend/apps/anomalies/urls.py
- [X] T084 [US4] Implement anomaly WebSocket consumer (anomaly.alert, anomaly.status_change, anomaly.behavior_end) and routing in backend/apps/anomalies/consumers.py and backend/apps/anomalies/routing.py
- [X] T085 [US4] Implement anomaly and predictions API client in frontend/src/api/anomalies.ts
- [X] T086 [US4] Implement anomaly store and useAnomalySocket hook in frontend/src/stores/anomalyStore.ts and frontend/src/hooks/useAnomalySocket.ts
- [X] T087 [US4] Implement PredictionsPanel and StudentCard components (per-student pyramid layers, constraint status) in frontend/src/components/detection/PredictionsPanel.tsx and frontend/src/components/detection/StudentCard.tsx
- [X] T088 [US4] Implement AnomalyAlert, TriageActions, and AlertHistory components in frontend/src/components/anomaly/
- [X] T089 [US4] Integrate collapsible predictions sidebar into CameraFeedPage in frontend/src/pages/CameraFeedPage.tsx
- [X] T090 [US4] Implement full-screen PredictionsPage in frontend/src/pages/PredictionsPage.tsx
- [X] T091 [US4] Implement AnomalyListPage for alert history browsing (all statuses, reasons, notes) in frontend/src/pages/AnomalyListPage.tsx
- [X] T092 [P] [US4] Add backend triage conflict and first-write-wins integration tests in backend/tests/integration/test_anomaly_triage.py

---

## Phase 8: User Story 6 — Video Recording & Storage (Priority: P2)

**Goal**: Record active sessions as raw video, provide playback with synchronized overlays and timeline markers, instructor comments, storage warnings, and async session export.

**Independent Test**: Run session, stop, replay recording with anomaly markers and comments, initiate export and download ZIP.

- [X] T093 [US6] Implement Recording model with dedup-aware M2M linking (one recording per camera regardless of instructor count) in backend/apps/recordings/models.py
- [X] T094 [US6] Implement RecordingService (start/stop/deduplication/storage threshold warnings at 80%/95%) in backend/apps/recordings/services.py
- [X] T095 [US6] Implement recording serializers in backend/apps/recordings/serializers.py
- [X] T096 [US6] Implement recording endpoints (list, stream, anomaly-markers) and register routes in backend/apps/recordings/views.py and backend/apps/recordings/urls.py
- [X] T097 [US6] Implement detection frame and prediction query endpoints for playback in backend/apps/detections/views.py
- [X] T098 [US6] Implement InstructorComment model (session-scoped, camera-optional, immutable) in backend/apps/sessions/models.py
- [X] T099 [US6] Implement session comment list/create endpoints in backend/apps/sessions/views.py
- [X] T100 [US6] Implement session comments WebSocket consumer (comment.new events) and routing in backend/apps/sessions/consumers.py and backend/apps/sessions/routing.py
- [X] T101 [US6] Implement SessionExport model (status: pending→processing→completed|failed, progress %) in backend/apps/exports/models.py
- [X] T102 [US6] Implement ExportService in backend/apps/exports/services.py
- [X] T103 [US6] Implement export Celery task for ZIP archive generation (video + structured data) in backend/apps/exports/tasks.py
- [X] T104 [US6] Implement export endpoints (start, status polling, download) and register routes in backend/apps/exports/views.py and backend/apps/exports/urls.py
- [X] T105 [US6] Implement session and recording API client methods in frontend/src/api/sessions.ts
- [X] T106 [US6] Implement SessionListPage with session browsing and status indicators in frontend/src/pages/SessionListPage.tsx
- [X] T107 [US6] Implement SessionDetailPage with playback, synchronized overlays, timeline markers, comments, and export progress in frontend/src/pages/SessionDetailPage.tsx
- [X] T108 [US6] Implement RecordingsPage for recording browsing with student ID filtering in frontend/src/pages/RecordingsPage.tsx
- [X] T109 [P] [US6] Add integration tests for record-playback-export flow in backend/tests/integration/test_recording_export.py

---

## Phase 9: User Story 9 — Admin Dashboard & Live Session Monitoring (Priority: P2)

**Goal**: Admin system-wide overview with active sessions, unified real-time anomaly feed, recent activity log, and jump-to-session monitoring.

**Independent Test**: Admin sees active sessions + anomaly feed and can jump into a live session view with real-time updates.

- [X] T110 [US9] Implement role-aware dashboard aggregate endpoint (GET /api/v1/dashboard/) in backend/apps/sessions/views.py
- [X] T111 [US9] Implement admin anomaly feed endpoint (GET /api/v1/dashboard/admin/anomaly-feed/) in backend/apps/anomalies/views.py
- [X] T112 [US9] Implement admin activity feed endpoint (GET /api/v1/dashboard/admin/activity-feed/) with page-size and action-type filters in backend/apps/audit/views.py
- [X] T113 [US9] Register audit URL routes in backend/apps/audit/urls.py
- [X] T114 [US9] Implement admin anomaly WebSocket channel (ws/admin/anomalies/) for cross-session high/medium alerts in backend/apps/anomalies/consumers.py
- [X] T115 [US9] Implement dashboard API client in frontend/src/api/dashboard.ts
- [X] T116 [US9] Implement Admin DashboardPage (active sessions, anomaly feed, activity feed FR-073, navigation, jump-to-session, admin comment) in frontend/src/pages/DashboardPage.tsx
- [X] T117 [P] [US9] Add dashboard endpoint integration tests in backend/tests/integration/test_admin_dashboard.py

---

## Phase 10: User Story 10 — Admin User & Role Management (Priority: P2)

**Goal**: Admin manages users (CRUD, deactivate, password reset), roles (custom with page+action permissions), rooms (CRUD, camera assignment), exams (CRUD, rosters), and merged session view.

**Independent Test**: Admin creates instructor with role, creates exam with roster, instructor logs in and sees exam on Exam Board.

- [X] T118 [US10] Implement admin user management endpoints (list, create, detail, edit, deactivate, password reset with must_change_password) in backend/apps/accounts/views.py
- [X] T119 [US10] Implement admin role management endpoints (list, create, detail, edit, delete — built-in roles non-deletable) in backend/apps/accounts/views.py
- [X] T120 [US10] Implement admin room management endpoints (CRUD + assign/remove camera) in backend/apps/exams/views.py
- [X] T121 [US10] Implement admin exam and student roster endpoints (CRUD + student add/remove) in backend/apps/exams/views.py
- [X] T122 [US10] Implement merged session view endpoint (GET /api/v1/exams/{id}/sessions/merged/) for multi-instructor aggregation in backend/apps/exams/views.py
- [X] T123 [US10] Wire audit event emission for all admin management actions (account, role, exam, room changes) in backend/apps/audit/services.py
- [X] T124 [US10] Implement admin management API client in frontend/src/api/admin.ts
- [X] T125 [US10] Implement UserManagementPage (list, create, edit, deactivate, password reset) in frontend/src/pages/UserManagementPage.tsx
- [X] T126 [US10] Implement RoleManagementPage with role, exam, and room management sections in frontend/src/pages/RoleManagementPage.tsx
- [X] T127 [P] [US10] Add admin management integration tests in backend/tests/integration/test_admin_management.py

---

## Phase 11: User Story 11 — Admin Recording Review & Action Revert (Priority: P2)

**Goal**: Admin reviews historical sessions with full context (video, overlays, comments, model outputs, triage history) and can revert triage actions with mandatory reason.

**Independent Test**: Admin reverts a dismissed anomaly back to `new`; reason is logged in audit; connected viewers receive real-time WebSocket notification.

- [X] T128 [US11] Implement admin session browse and review endpoints (GET /api/v1/admin/sessions/, GET /{id}/review/) in backend/apps/sessions/views.py
- [X] T129 [US11] Implement triage revert endpoint (POST /api/v1/admin/anomalies/{id}/revert/) with mandatory reason validation in backend/apps/anomalies/views.py
- [X] T130 [US11] Implement admin recording deletion endpoint (DELETE /api/v1/admin/recordings/{id}/, completed only) in backend/apps/recordings/views.py
- [X] T131 [US11] Wire audit logging and WebSocket broadcast (anomaly.status_change) for triage reverts in backend/apps/anomalies/services.py
- [X] T132 [US11] Implement RecordingReviewPage with filters (instructor, exam, date, anomaly count), playback, overlays, and model outputs in frontend/src/pages/RecordingReviewPage.tsx
- [X] T133 [US11] Implement revert action modal (mandatory reason, min 5 chars) and real-time status refresh UI in frontend/src/components/anomaly/TriageActions.tsx
- [X] T134 [US11] Implement AuditLogPage with action-type filters, page-size selector (10-100 + All with lazy loading, default 50) in frontend/src/pages/AuditLogPage.tsx
- [X] T135 [P] [US11] Add integration tests for revert + WebSocket broadcast flow in backend/tests/integration/test_admin_revert.py

---

## Phase 12: User Story 5 — Theme Switching (Priority: P3)

**Goal**: Three themes (Black+Purple default, Full Black, White) with instant apply (<300ms), localStorage persistence, and WCAG 2.1 AA contrast.

**Independent Test**: User selects theme, reloads app, and selected theme remains active; all components render correctly in each theme.

- [X] T136 [US5] Implement theme selector behavior and localStorage persistence in frontend/src/stores/authStore.ts
- [X] T137 [US5] Integrate theme switcher in AppLayout header and SettingsPage in frontend/src/components/layout/AppLayout.tsx
- [X] T138 [P] [US5] Finalize black-purple theme CSS variables in frontend/src/styles/themes/black-purple.css
- [X] T139 [P] [US5] Finalize full-black theme CSS variables in frontend/src/styles/themes/full-black.css
- [X] T140 [P] [US5] Finalize white theme CSS variables in frontend/src/styles/themes/white.css
- [X] T141 [US5] Verify bounding box overlay contrast tokens across all three themes in frontend/src/styles/tokens.css
- [X] T142 [P] [US5] Add theme persistence and switching unit tests in frontend/tests/unit/stores/authTheme.test.ts

---

## Phase 13: User Story 7 — Multi-Camera Management (Priority: P3)

**Goal**: Add/remove/name cameras, focus mode (double-click expand/restore), persistent layout preferences, and camera list management.

**Independent Test**: Add multiple cameras, focus one feed, disconnect one without impacting others, verify layout persists.

- [X] T143 [US7] Implement camera add/remove/name management interactions in frontend/src/pages/CameraFeedPage.tsx
- [X] T144 [US7] Implement camera focus mode (double-click expand/restore) in frontend/src/components/camera/CameraGrid.tsx
- [X] T145 [US7] Implement CameraListPage for system-wide camera management in frontend/src/pages/CameraListPage.tsx
- [X] T146 [US7] Persist camera layout preferences in frontend/src/stores/cameraStore.ts
- [X] T147 [P] [US7] Add camera grid interaction tests in frontend/tests/unit/components/camera/CameraGrid.test.tsx

---

## Phase 14: Polish & Cross-Cutting Concerns

**Purpose**: Complete observability, hardening, documentation, accessibility, and full validation.

- [X] T148 Implement health check endpoint (/health) with per-subsystem thresholds (DB, Redis, pipeline FPS, storage, WebSocket) in backend/apps/health/views.py
- [X] T149 Implement health WebSocket consumer (health.update every 10s, storage.warning at 80%/95%) in backend/apps/health/consumers.py
- [X] T150 Register health URL routes and WebSocket routing in backend/apps/health/urls.py and backend/apps/health/routing.py
- [X] T151 Implement HealthDashboardPage (API status, WS health, pipeline FPS, storage usage, active sessions) in frontend/src/pages/HealthDashboardPage.tsx
- [X] T152 [P] Validate shared UI component library coverage (Button, Card, Modal, StatusBadge, Toggle, form inputs) per FR-033 in frontend/src/components/ui/
- [X] T153 [P] Implement React ErrorBoundary wrapper with user-friendly fallback UI per FR-036 in frontend/src/components/ui/ErrorBoundary.tsx
- [X] T154 Audit all async operations for loading state coverage per FR-034 in frontend/src/
- [X] T155 Audit all primary workflows for keyboard navigability per FR-035 in frontend/src/
- [X] T156 Validate responsive layout at 1920×1080 minimum resolution per FR-037 in frontend/src/
- [X] T157 Configure process supervisor auto-restart policy (<30s recovery) per FR-049 in docker-compose.dev.yml
- [X] T158 Implement anomaly alert start/end timestamp display format ("Started 2m 35s ago — Ongoing") per FR-068 in frontend/src/components/anomaly/AnomalyAlert.tsx
- [X] T159 [P] Run and fix backend lint, type checks, and full test suite in backend/
- [X] T160 [P] Run and fix frontend lint, type checks, and full test suite in frontend/
- [X] T161 Update root and module READMEs with final architecture and setup steps in README.md
- [X] T162 Run quickstart end-to-end validation per specs/001-exam-monitor-dashboard/quickstart.md
- [X] T163 [P] Add load test harness for concurrent instructor target in backend/tests/system/
- [X] T164 [P] Run WCAG 2.1 AA contrast audit against all three themes in frontend/

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: Starts immediately — no dependencies.
- **Phase 2 (Foundational)**: Depends on Phase 1 — **BLOCKS all user stories**.
- **P1 Stories**: US1, US2, US8 start after Phase 2 (US8 depends on session + camera foundations from US2).
- **P2 Stories**: US3 depends on US2; US4 depends on US2; US6 depends on US2 + US8; US9 depends on US1 + US4; US10 depends on Phase 2; US11 depends on US4 + US6.
- **P3 Stories**: US5 depends on foundational theming/auth; US7 depends on US2.
- **Phase 14 (Polish)**: After selected stories complete.

### User Story Completion Order (Dependency Graph)

```
Phase 1 (Setup)
    │
Phase 2 (Foundational) ← BLOCKS ALL
    │
    ├── US1 (Login) ──────────────────┐
    │                                  │
    ├── US2 (Camera Feed) ────────────┤
    │       │                          │
    │       ├── US3 (Filtering)        ├── MVP Checkpoint
    │       │                          │
    │       ├── US4 (Predictions) ─────┤
    │       │       │                  │
    │       │       ├── US9 (Admin Dashboard)
    │       │       │
    │       ├── US7 (Multi-Camera)     │
    │       │                          │
    │       └── US8 (Exam Board) ──────┘
    │               │
    │               └── US6 (Recording)
    │                       │
    │                       └── US11 (Admin Review)
    │
    ├── US10 (Admin Mgmt) ← only needs Phase 2
    │
    └── US5 (Theme) ← only needs Phase 2 + auth
         │
Phase 14 (Polish)
```

### Recommended Execution Order

1. **US1 + US2** (parallel — independent P1 stories)
2. **US8** (depends on US2 camera + session foundations)
3. **US3 + US4 + US10** (parallel where staffed — US3/US4 need US2, US10 only needs Phase 2)
4. **US6** (depends on US2 + US8 session lifecycle)
5. **US9 + US11** (US9 needs US4 anomaly feed; US11 needs US4 + US6)
6. **US5 + US7** (P3 polish stories)
7. **Phase 14** (hardening and validation)

### Within Each User Story

- Models/services before endpoints.
- Endpoints before frontend integration.
- Backend before dependent frontend tasks.
- Core flow before edge-case handling.
- Tests validate the story independently.

---

## Parallel Execution Examples

### US1

```
# Backend (parallel):
T038: backend/tests/integration/test_auth_api.py
T039: frontend/tests/unit/components/auth/LoginPage.test.tsx

# Frontend (parallel):
T031: frontend/src/api/auth.ts
T036: frontend/src/pages/LoginPage.module.css
```

### US2

```
# Backend models (parallel after T040):
T041: backend/apps/cameras/serializers.py
T046: backend/apps/detections/serializers.py

# Frontend components (parallel):
T053: frontend/src/components/camera/CameraFeed.tsx
T054: frontend/src/components/camera/BoundingBoxCanvas.tsx
T055: frontend/src/components/camera/CameraGrid.tsx

# Tests (parallel):
T057: backend/tests/integration/test_camera_lifecycle.py
T058: frontend/tests/unit/components/camera/CameraFeed.test.tsx
```

### US4

```
# Pipeline layers (parallel):
T076: backend/apps/pipeline/base.py
T077: backend/apps/pipeline/layers/ (all 4 layers)

# Frontend components (parallel):
T087: frontend/src/components/detection/PredictionsPanel.tsx
T088: frontend/src/components/anomaly/ (3 components)
```

### US6

```
# Backend models (parallel):
T093: backend/apps/recordings/models.py
T098: backend/apps/sessions/models.py (InstructorComment)
T101: backend/apps/exports/models.py

# Frontend pages (parallel):
T106: frontend/src/pages/SessionListPage.tsx
T108: frontend/src/pages/RecordingsPage.tsx
```

### US10

```
# Backend endpoints (parallel — different viewsets):
T118: backend/apps/accounts/views.py (users)
T119: backend/apps/accounts/views.py (roles)
T120: backend/apps/exams/views.py (rooms)
T121: backend/apps/exams/views.py (exams)
```

### US5

```
# Theme CSS files (all parallel):
T138: frontend/src/styles/themes/black-purple.css
T139: frontend/src/styles/themes/full-black.css
T140: frontend/src/styles/themes/white.css
```

---

## Implementation Strategy

### MVP First (P1 Stories)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL — blocks all stories)
3. Complete US1 (Login) + US2 (Camera Feed) in parallel
4. Complete US8 (Exam Board)
5. **STOP and VALIDATE**: Test login → exam board → camera feed → live detection flow
6. Deploy/demo MVP

### Incremental Delivery

- **Increment 1 (MVP)**: Setup + Foundation + US1 + US2 + US8 → Auth, live monitoring, exam scheduling
- **Increment 2 (Core Features)**: US3 + US4 + US10 → Filtering, predictions, anomaly triage, admin management
- **Increment 3 (Recording)**: US6 → Recording, playback, comments, export
- **Increment 4 (Admin Oversight)**: US9 + US11 → Admin dashboard, recording review, triage revert
- **Increment 5 (Polish)**: US5 + US7 → Themes, multi-camera UX
- **Increment 6 (Hardening)**: Phase 14 → Health monitoring, accessibility, load testing, docs

### Entity Coverage (16/16)

| Entity | Phase | Task(s) |
|--------|-------|---------|
| User | 2 | T013 |
| Role | 2 | T013, T014 |
| Room | 2 | T016 |
| AuditLogEntry | 2 | T015 |
| CameraSource | 4 (US2) | T040 |
| DetectionFrame | 4 (US2) | T045 |
| Detection | 4 (US2) | T045 |
| Exam | 5 (US8) | T059 |
| ExamStudent | 5 (US8) | T059 |
| MonitoringSession | 5 (US8) | T060 |
| PyramidPrediction | 7 (US4) | T074 |
| AnomalyEvent | 7 (US4) | T075 |
| AnomalyNote | 7 (US4) | T075 |
| Recording | 8 (US6) | T093 |
| InstructorComment | 8 (US6) | T098 |
| SessionExport | 8 (US6) | T101 |

### WebSocket Channel Coverage (6/6)

| Channel | Phase | Task(s) |
|---------|-------|---------|
| Camera Feed (ws/cameras/{id}/) | 4 (US2) | T044 |
| Detections (ws/detections/{session_id}/) | 4 (US2) | T047 |
| Anomalies (ws/anomalies/{session_id}/) | 7 (US4) | T084 |
| Session Comments (ws/sessions/{id}/comments/) | 8 (US6) | T100 |
| Admin Anomaly Feed (ws/admin/anomalies/) | 9 (US9) | T114 |
| Health (ws/health/) | 14 | T149 |

---

## Notes

- [P] tasks = different files, no dependencies on incomplete tasks
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Verify tests fail before implementing (TDD: Red → Green → Refactor)
- **ALL test cases (unit + integration + system) MUST pass** before marking any task or story as complete — partial passes are a blocking failure
- **Delivery Sign-Off Protocol (Principle VII)**: Before marking work as delivered, the agent MUST: (1) scan for ALL issue types (lint, type, security, runtime, spec compliance), (2) determine and apply fixes, (3) verify full test suite passes, (4) verify security gateways pass, (5) confirm spec-driven compliance, (6) produce a brief delivery summary
- **Commit after each task** with an explanatory Conventional Commits message: `type(scope): description` — vague messages like "fix stuff" or "WIP" are prohibited
- **All work MUST be submitted via Pull Request** — commit on the active spec branch (do not create a new branch per PR), open a PR, and ensure it is reviewed (CodeRabbit + human where required) before merging. Direct pushes to protected branches are prohibited
- **Create/update `.md` documentation** for every file created and for every module — documentation is created alongside code, not after
- **Every module MUST have its own `README.md`** — a module without a README is not considered delivered
- **Update the root `README.md`** whenever modules, features, or setup steps change
- **Mermaid diagrams** MUST be colorful, readable, and updated when architecture changes
- Stop at any checkpoint to validate story independently
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence
