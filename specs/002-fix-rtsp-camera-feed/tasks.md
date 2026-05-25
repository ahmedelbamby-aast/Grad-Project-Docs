# Tasks: Fix RTSP Camera Feed Connection

**Input**: Design documents from `/specs/002-fix-rtsp-camera-feed/`
**Prerequisites**: plan.md (required), spec.md (required), research.md, data-model.md, contracts/, quickstart.md

**Tests**: Not explicitly requested in spec. Constitution mandates TDD — agents implementing tasks must write tests alongside code per constitution §II, but no separate test tasks are generated here.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete tasks in the same phase)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3, US4, US5)
- Include exact file paths in descriptions

## Path Conventions

- **Backend**: `backend/apps/cameras/`, `backend/config/`
- **Frontend**: `frontend/src/`
- **Infra**: `nginx.conf`, `docker-compose.dev.yml`
- **Docs**: `docs/backend/`, `docs/frontend/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Add new dependencies, create foundational modules (exceptions, constants, custom fields, string constants) that all user stories depend on.

- [X] T001 Add httpx>=0.27 to backend/requirements.in and backend/requirements.txt
- [X] T002 [P] Create CameraConnectionError exception hierarchy (9 subclasses per research-rtsp-probe.md) in backend/apps/cameras/exceptions.py
- [X] T003 [P] Create camera constants module (RTSP_PROBE_TIMEOUT, HEALTH_CHECK_INTERVAL, MAX_RECONNECT_RETRIES, RECONNECT_BASE_DELAY, error message templates, Pydantic config model) in backend/apps/cameras/constants.py
- [X] T004 [P] Create EncryptedTextField custom model field with Fernet encryption and HMAC-SHA256 blind index helper in backend/apps/cameras/fields.py
- [X] T005 [P] Create frontend camera message constants (all user-facing strings for errors, progress, status) in frontend/src/constants/cameraMessages.ts

**Checkpoint**: Shared modules ready — all subsequent phases can import these.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core model changes, migrations, service classes, serializer updates, and frontend types that MUST be complete before ANY user story can be implemented.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [X] T006 Update CameraSource model: change connection_url to EncryptedTextField, add connection_url_hash CharField(64), add error_detail CharField(500), add UniqueConstraint on (added_by, connection_url_hash), add status index in backend/apps/cameras/models.py
- [X] T007 Create StreamRegistration model (OneToOne to CameraSource, relay_stream_name, last_health_status, retry_count) in backend/apps/cameras/models.py
- [X] T008 Create ConnectionEvent model (camera FK, event_type with 12 choices, error_detail, triggered_by FK, session FK) in backend/apps/cameras/models.py
- [X] T009 Generate schema migrations for CameraSource field changes, StreamRegistration, and ConnectionEvent in backend/apps/cameras/migrations/
- [X] T010 Create data migration to encrypt existing connection_url values with Fernet and compute HMAC-SHA256 hashes in backend/apps/cameras/migrations/
- [X] T011 Create migration to alter DetectionFrame.camera and AnomalyEvent.camera FK cascade from CASCADE to SET_NULL (nullable) in backend/apps/detections/migrations/ and backend/apps/anomalies/migrations/
- [X] T012 [P] Implement Go2rtcClient HTTP service (register_stream, unregister_stream, list_streams, get_stream_health) using httpx in backend/apps/cameras/services.py
- [X] T013 Implement RtspProber service (tcp_probe, rtsp_describe, frame_decode via cv2.VideoCapture) in backend/apps/cameras/services.py
- [X] T014 Update CameraSourceSerializer: masked connection_url output via regex, error_detail field, auto-compute connection_url_hash on save, per-instructor uniqueness validation in backend/apps/cameras/serializers.py
- [X] T015 [P] Add frontend types (ConnectionEvent, ConnectionEventType, StreamRegistration, updated CameraSource with error_detail) in frontend/src/types/api.ts
- [X] T016 [P] Add WebSocket message types (CameraStatusMessage, RelayHealthMessage) in frontend/src/types/websocket.ts

**Checkpoint**: Foundation ready — all models migrated, service classes available, types defined. User story implementation can now begin.

---

## Phase 3: User Story 1 — Connect Camera and See Live Feed (Priority: P1) 🎯 MVP

**Goal**: An instructor enters an RTSP URL, clicks "Connect," and sees a live video feed. The system validates the URL, registers the stream with go2rtc, and updates the status. Clicking "Disconnect" stops the stream and cleans up.

**Independent Test**: Create a camera with a valid RTSP URL, click Connect, verify the live video appears in the feed page. Click Disconnect, verify the feed stops and status updates.

### Implementation for User Story 1

- [X] T017 [US1] Implement CameraService.connect() pipeline: decrypt URL → RtspProber.validate() → Go2rtcClient.register_stream() → create StreamRegistration → update status to connected → log connect_success ConnectionEvent in backend/apps/cameras/services.py
- [X] T018 [US1] Implement CameraService.disconnect() pipeline: Go2rtcClient.unregister_stream() → delete StreamRegistration → update status to disconnected → clear error_detail → log disconnect ConnectionEvent in backend/apps/cameras/services.py
- [X] T019 [US1] Rewrite connect and disconnect view actions to use CameraService pipeline, return updated camera object, handle already-connected/disconnected states (409) in backend/apps/cameras/views.py
- [X] T020 [P] [US1] Update connect() and disconnect() API functions to handle new response shapes and error codes in frontend/src/api/cameras.ts
- [X] T021 [P] [US1] Extend cameraStore with setCameraStatus action, setLoading per-camera, and error_detail tracking in frontend/src/stores/cameraStore.ts
- [X] T022 [US1] Update CameraFeed.tsx: disable Connect/Disconnect buttons during operation (FR-010), show loading spinner, display success toast with "View Live Feed →" link on connect (FR-021) in frontend/src/components/camera/CameraFeed.tsx

**Checkpoint**: Core camera connection works end-to-end. Instructor can connect a camera and see live feed. MVP deliverable.

---

## Phase 4: User Story 2 — See Clear Error Messages on Connection Failure (Priority: P1)

**Goal**: When a connection attempt fails, the system displays a specific, categorized error explaining what went wrong (unreachable host, auth failure, no video, relay down) and what to try next. Error history is available per camera.

**Independent Test**: Attempt to connect with an unreachable RTSP URL. Verify a descriptive error message appears on the camera tile (inline banner) and as a toast. Check that the error is logged in the events history.

### Implementation for User Story 2

- [X] T023 [US2] Add stage-specific error response formatting to connect view: error_code, stage field, human-readable detail per CameraConnectionError subclass in backend/apps/cameras/views.py
- [X] T024 [P] [US2] Create ErrorBanner.tsx persistent inline error component with dismiss button, themed styling, and ARIA live region in frontend/src/components/camera/ErrorBanner.tsx
- [X] T025 [US2] Integrate ErrorBanner into CameraFeed.tsx: show on error status, persist until retry or dismiss, add toast notification (~5s) for global awareness in frontend/src/components/camera/CameraFeed.tsx
- [X] T026 [US2] Create ConnectionEventSerializer (read-only) and add events list action to CameraSourceViewSet (last 10 for instructors, full paginated for admins) in backend/apps/cameras/serializers.py and backend/apps/cameras/views.py
- [X] T027 [US2] Add events URL route to camera router in backend/apps/cameras/urls.py
- [X] T028 [P] [US2] Add getConnectionEvents() API function in frontend/src/api/cameras.ts

**Checkpoint**: All connection failures show specific, categorized error messages. Instructors see clear guidance on what went wrong and how to fix it.

---

## Phase 5: User Story 3 — Real-Time Camera Status Updates (Priority: P2)

**Goal**: Camera status changes (connected, disconnected, error, reconnecting) are broadcast via WebSocket to all viewing clients within 2 seconds. A background health-check task detects stream loss and auto-reconnects with exponential backoff.

**Independent Test**: Open camera feed in two browser tabs, connect a camera from Tab 1, verify Tab 2 updates within 2 seconds. Restart go2rtc, verify auto-reconnection within 30 seconds.

### Implementation for User Story 3

- [X] T029 [US3] Implement _broadcast_status() with dual group_send (camera_{id} and cameras_all) and wire into all CameraService status-change methods in backend/apps/cameras/services.py
- [X] T030 [US3] Extend CameraStatusConsumer: handle cameras_all group join for global route, add relay_health message handler for admin clients in backend/apps/cameras/consumers.py
- [X] T031 [US3] Add global camera WebSocket route ws/cameras/ in backend/apps/cameras/routing.py
- [X] T032 [P] [US3] Create CameraStatusBadge.tsx accessible component with semantic colors (green/red/yellow/grey), ARIA label, and WCAG AA contrast in frontend/src/components/camera/CameraStatusBadge.tsx
- [X] T033 [P] [US3] Extend cameraStore with setCameraStatus action triggered by WebSocket messages in frontend/src/stores/cameraStore.ts
- [X] T034 [US3] Create useCameraStatus.ts hook: per-camera mode (ws/cameras/{id}/) and global mode (ws/cameras/), update cameraStore on camera.status messages in frontend/src/hooks/useCameraStatus.ts
- [X] T035 [US3] Implement stream_health_check periodic Celery task: poll Go2rtcClient for all connected cameras, detect stream loss, auto-reconnect with exponential backoff (2s→4s→8s, max 3 retries), transition to error on exhaustion in backend/apps/cameras/tasks.py
- [X] T036 [US3] Add stream_health_check beat schedule (15s interval) in backend/config/celery.py

**Checkpoint**: Camera status updates propagate in real time to all viewers. Background health check detects stream loss and auto-reconnects. Status badges show live state.

---

## Phase 6: User Story 4 — Connection Progress Feedback (Priority: P2)

**Goal**: During the multi-second connection process, the user sees step-by-step progress (Validating → Registering → Connecting Video → Live) instead of a generic spinner.

**Independent Test**: Click Connect on a camera and observe distinct progress stages displayed. If connection fails at validation, verify the progress stops at that stage with the error message.

### Implementation for User Story 4

- [X] T037 [P] [US4] Create ConnectionProgress.tsx multi-stage progress component (4 stages: validating, registering, connecting, live) with ARIA live region announcements in frontend/src/components/camera/ConnectionProgress.tsx
- [X] T038 [US4] Add connection progress state (currentStage, failedStage) to cameraStore and update from connect API response in frontend/src/stores/cameraStore.ts
- [X] T039 [US4] Integrate ConnectionProgress into CameraFeed.tsx connect flow: show during connection, stop at failed stage with error, clear on success in frontend/src/components/camera/CameraFeed.tsx

**Checkpoint**: Connection process shows transparent stage-by-stage progress. Users know exactly what is happening and where failures occur.

---

## Phase 7: User Story 5 — WHEP Video Path Through Authenticated Proxy (Priority: P3)

**Goal**: WebRTC video requests are routed through nginx with session auth, preventing unauthenticated video access. The frontend uses same-origin `/whep/{id}/` instead of direct go2rtc access.

**Independent Test**: Verify WHEP requests go through `/whep/{camera_id}/` nginx route. Confirm unauthenticated requests receive 401/403. Verify live feed still works through proxy.

### Implementation for User Story 5

- [X] T040 [US5] Fix nginx WHEP proxy: add go2rtc_api upstream (go2rtc:1984), auth_request subrequest to /api/v1/auth/me/, rewrite /whep/{camera_id}/ to /api/webrtc?src=camera_{camera_id} in nginx.conf
- [X] T041 [US5] Rewrite useWhepClient.ts: remove VITE_GO2RTC_URL, use same-origin /whep/{cameraId}/ URL, implement state machine (idle→connecting→connected→reconnecting→error→disconnected), 3-retry exponential backoff (2s→4s→8s) in frontend/src/hooks/useWhepClient.ts

**Checkpoint**: All video streams are authenticated. No unauthenticated access possible. WHEP client handles retries gracefully.

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Admin features, edge-case handling, FR completeness, documentation, and validation.

### Admin Features (FR-023, FR-024)

- [X] T042 Create admin camera serializers (AdminCameraSerializer with instructor details, RelayHealthSerializer) in backend/apps/cameras/serializers.py
- [X] T043 Implement admin endpoints: camera dashboard list, force-disconnect, connect-on-behalf, relay-health GET in backend/apps/cameras/views.py
- [X] T044 Add admin URL routes for dashboard, force-disconnect, connect-on-behalf, relay-health in backend/apps/cameras/admin_urls.py

### Edge Cases & FR Completeness

- [X] T045 Implement camera deletion flow: confirmation guard, auto-disconnect if connected, preserve all session data (detections, anomalies, recordings), SET_NULL on ConnectionEvent.camera (FR-022) in backend/apps/cameras/views.py
- [X] T046 Implement camera URL edit with auto-disconnect: detect connection_url change on PATCH, disconnect if connected, prompt reconnect (FR-020) in backend/apps/cameras/serializers.py and backend/apps/cameras/views.py
- [X] T047 [P] Create useBulkFailure.ts hook: consolidate 3+ failures within 10s window into single summary banner (FR-027) in frontend/src/hooks/useBulkFailure.ts
- [X] T048 Implement cleanup_connection_events periodic task (delete events older than 90 days) and add beat schedule in backend/apps/cameras/tasks.py and backend/config/celery.py
- [X] T049 Implement monitoring interruption event logging: log monitoring_interruption_start/end events with session FK when stream lost/restored during active exam session (FR-026) in backend/apps/cameras/services.py

### Documentation (Constitution §docs/, §README)

- [X] T050 [P] Create/update docs/ Markdown files for all backend cameras source files (models.md, services.md, views.md, consumers.md, tasks.md, exceptions.md, constants.md) in docs/backend/apps/cameras/
- [X] T051 [P] Create/update docs/ Markdown files for all frontend source files (useWhepClient.md, useCameraStatus.md, useBulkFailure.md, CameraFeed.md, CameraStatusBadge.md, ConnectionProgress.md, ErrorBanner.md, cameraMessages.md) in docs/frontend/src/
- [X] T052 [P] Update module README.md for backend/apps/cameras/ and relevant frontend component/hook directories

### Validation

- [X] T053 Run quickstart.md validation steps: verify environment setup, connect camera, test error handling, test real-time status, test auto-reconnection

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup (Phase 1) completion — **BLOCKS all user stories**
- **User Story 1 (Phase 3)**: Depends on Foundational (Phase 2) — no other story dependencies
- **User Story 2 (Phase 4)**: Depends on Foundational (Phase 2) — can integrate with US1 but independently testable
- **User Story 3 (Phase 5)**: Depends on Foundational (Phase 2) — adds WebSocket layer on top of US1 service methods
- **User Story 4 (Phase 6)**: Depends on Foundational (Phase 2) — pure frontend, uses connect API from US1
- **User Story 5 (Phase 7)**: Depends on Foundational (Phase 2) — can be done independently of other stories
- **Polish (Phase 8)**: Depends on US1-US3 being complete (admin endpoints use service methods and WS broadcast)

### User Story Dependencies

- **US1 (P1)**: Standalone after Phase 2. Core MVP.
- **US2 (P1)**: Standalone after Phase 2. Error formatting extends US1 views but independently testable with mock failures.
- **US3 (P2)**: Standalone after Phase 2. Adds broadcast to US1 service methods — US1 works without US3 (single client gets response). US3 adds multi-client real-time sync.
- **US4 (P2)**: Standalone after Phase 2. Pure frontend progress UI — works with any backend that returns stage info.
- **US5 (P3)**: Fully independent. nginx proxy + frontend WHEP rewrite. No backend code dependency.

### Within Each User Story

- Backend service logic before views/endpoints
- Views before URL routes
- Frontend API functions alongside backend (different files, [P])
- Store updates before component integration
- Components depend on store and API functions

### Parallel Opportunities

**Phase 1** — All 5 tasks can run in parallel (different files):
```
T001 (requirements) || T002 (exceptions.py) || T003 (constants.py) || T004 (fields.py) || T005 (cameraMessages.ts)
```

**Phase 2** — Go2rtcClient and frontend types can run alongside model work:
```
T006→T007→T008→T009→T010→T011 (models + migrations, sequential)
  ||
T012→T013 (services, sequential but parallel with model work)
  ||
T015 (types/api.ts) || T016 (types/websocket.ts)
```
Then T014 (serializers) after T006-T008 complete.

**Phase 3 (US1)** — Frontend and backend can proceed in parallel:
```
T017→T018→T019 (backend: services → views)
  ||
T020 (api) || T021 (store)
  →→ T022 (CameraFeed.tsx, after T020+T021)
```

**Phase 5 (US3)** — Components and store alongside backend:
```
T029→T030→T031 (backend: broadcast → consumer → routing)
  ||
T032 (StatusBadge) || T033 (store)
  →→ T034 (hook, after T033)
  →→ T035→T036 (tasks + celery, after T029)
```

**Across Stories** — With multiple developers:
```
Developer A: US1 (Phase 3) → US2 (Phase 4)
Developer B: US3 (Phase 5) → US4 (Phase 6)
Developer C: US5 (Phase 7) → Polish (Phase 8)
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (5 tasks)
2. Complete Phase 2: Foundational — **CRITICAL, blocks all stories** (11 tasks)
3. Complete Phase 3: User Story 1 — Connect Camera and See Live Feed (6 tasks)
4. **STOP and VALIDATE**: Connect a camera, verify live feed appears, disconnect works
5. Deploy/demo if ready — the core bug is fixed

### Incremental Delivery

1. **Setup + Foundational** → Foundation ready (16 tasks)
2. **+ US1** → Camera connection works (6 tasks) → **MVP! Deploy/Demo**
3. **+ US2** → Error messages appear on failure (6 tasks) → Deploy/Demo
4. **+ US3** → Real-time status updates + auto-reconnection (8 tasks) → Deploy/Demo
5. **+ US4** → Connection progress feedback (3 tasks) → Deploy/Demo
6. **+ US5** → WHEP secured through auth proxy (2 tasks) → Deploy/Demo
7. **+ Polish** → Admin features, edge cases, docs (12 tasks) → Final delivery

### Parallel Team Strategy

With multiple developers after Phase 2 completes:

1. Team completes Setup + Foundational together (Phases 1-2)
2. Once Foundational is done:
   - **Developer A**: US1 → US2 (P1 stories, core fix + error messages)
   - **Developer B**: US3 → US4 (P2 stories, real-time + progress)
   - **Developer C**: US5 → Polish (P3 story + cross-cutting)
3. Stories integrate independently — each adds value without breaking others

---

## FR Coverage Map

| FR | Task(s) | Story |
|----|---------|-------|
| FR-001 (Register/unregister stream) | T017, T018, T019 | US1 |
| FR-002 (RTSP probe validation) | T013, T017 | Foundation + US1 |
| FR-003 (Error messages by failure type) | T023, T024, T025 | US2 |
| FR-004 (Multi-stage progress) | T037, T038, T039 | US4 |
| FR-005 (WHEP through auth proxy) | T040, T041 | US5 |
| FR-006 (WebSocket status broadcast) | T029, T030, T031, T034 | US3 |
| FR-007 (Status reflects actual state) | T017, T018 | US1 |
| FR-008 (Auto-reconnection) | T035, T036 | US3 |
| FR-009 (WHEP auto-retry) | T041 | US5 |
| FR-010 (Disable button during operation) | T022 | US1 |
| FR-011 (Reconnecting overlay) | T032, T025 | US3 + US2 |
| FR-012 (Server-side RTSP validation) | T013, T014 | Foundation |
| FR-013 (Connection event logging) | T008, T017, T026, T027 | Foundation + US1 + US2 |
| FR-014 (Video-only streams) | T013 | Foundation |
| FR-015 (Encrypted URLs, masked display) | T004, T006, T014 | Setup + Foundation |
| FR-016 (Re-register after relay restart) | T035 | US3 |
| FR-017 (No hard global limit) | T003, T017 | Setup + US1 |
| FR-018 (Auto-resume detection pipeline) | T035, T049 | US3 + Polish |
| FR-019 (URL uniqueness per instructor) | T006, T014 | Foundation |
| FR-020 (Edit URL with auto-disconnect) | T046 | Polish |
| FR-021 (Stay on page, success toast) | T022 | US1 |
| FR-022 (Camera deletion, preserve data) | T011, T045 | Foundation + Polish |
| FR-023 (Admin dashboard) | T042, T043, T044 | Polish |
| FR-024 (Admin force-disconnect/connect) | T043, T044 | Polish |
| FR-025 (String constants, ARIA, WCAG AA) | T005, T024, T032, T037 | Setup + US2 + US3 + US4 |
| FR-026 (Monitoring interruption events) | T049 | Polish |
| FR-027 (Bulk failure consolidation) | T047 | Polish |

---

## Notes

- [P] tasks = different files, no dependencies within the same phase
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Commit after each task or logical group
- Agents MUST create a commit after finishing ANY work — no uncommitted deliveries
- Every source file change MUST include its corresponding `docs/` .md file
- Module README.md MUST be created/updated after finishing work on a module
- Stop at any checkpoint to validate story independently
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence
