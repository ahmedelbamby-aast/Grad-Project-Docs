# Tasks: Exam Monitoring Dashboard — Frontend

**Input**: Design documents from `/specs/001-exam-monitor-dashboard/`
**Prerequisites**: plan.md (required), spec.md (required), contracts/rest-api.md, contracts/websocket-api.md

**Tests**: Included — TDD is mandated by Constitution Principle II and research R-008 (≥80% coverage gate). Frontend testing: Vitest + React Testing Library (unit/integration), Playwright (E2E).

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story. This file covers **frontend only** (React 19.2 + TypeScript ~5.9.3 + Vite 7.3 + Zustand 5.x).

**Assumes**: Backend API endpoints exist and are functional. All frontend tasks consume the REST API and WebSocket contracts defined in `contracts/rest-api.md` and `contracts/websocket-api.md`.

**Relationship to tasks.md**: This file provides granular frontend-only task breakdown with TF-prefixed IDs. The unified `tasks.md` covers both backend and frontend at a higher level with T-prefixed IDs. When both files reference the same feature, tasks.md provides the authoritative scope and this file provides detailed frontend implementation steps.

## Format: `[TF-ID] [P?] [Story] Description`

- **[TF-ID]**: Frontend task ID (TF-001 through TF-118) — distinct from tasks.md T-prefixed IDs
- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Frontend**: `frontend/src/`, `frontend/tests/`
- Unit tests: `frontend/tests/unit/{components,stores,hooks}/`
- Integration tests: `frontend/tests/integration/`
- E2E tests: `frontend/tests/e2e/`

---

## Phase 1: Setup (Frontend Project Initialization)

**Purpose**: Initialize React + TypeScript + Vite project, configure tooling, install dependencies

- [x] TF-001 Initialize React 19.2 + TypeScript ~5.9.3 + Vite 7.3 project with package.json, tsconfig.json, vite.config.ts in frontend/
- [x] TF-002 [P] Install runtime dependencies (react, react-dom, react-router-dom, zustand, axios) in frontend/package.json
- [x] TF-003 [P] Install dev dependencies (vitest, @testing-library/react, @testing-library/jest-dom, @testing-library/user-event, playwright, eslint, prettier, typescript) in frontend/package.json
- [x] TF-004 [P] Configure ESLint (typescript-eslint + react-hooks + import + a11y rules) in frontend/.eslintrc.cjs
- [x] TF-005 [P] Configure Vitest with jsdom environment, coverage thresholds (≥80%), and path aliases in frontend/vitest.config.ts
- [x] TF-006 [P] Configure Playwright for E2E testing with chromium in frontend/playwright.config.ts
- [x] TF-007 [P] Create .env.example with VITE_API_BASE_URL, VITE_WS_BASE_URL in frontend/

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core frontend infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

### Styles & Theming Foundation

- [x] TF-008 Create CSS custom property token definitions (color, spacing, typography, radius, shadow, transitions) in frontend/src/styles/tokens.css
- [x] TF-009 [P] Create global CSS reset and base element styles in frontend/src/styles/global.css
- [x] TF-010 [P] Create Black+Purple theme token overrides in frontend/src/styles/themes/black-purple.css
- [x] TF-011 [P] Create Full Black theme token overrides in frontend/src/styles/themes/full-black.css
- [x] TF-012 [P] Create White theme token overrides in frontend/src/styles/themes/white.css
- [x] TF-013 Create FOUC-prevention inline script in frontend/index.html that reads localStorage theme before first paint

### Shared TypeScript Types

- [x] TF-014 Define shared TypeScript interfaces from REST API contracts (Instructor, CameraSource, MonitoringSession, DetectionFrame, Detection, PyramidPrediction, AnomalyEvent, AnomalyNote, Recording, SessionExport, HealthStatus, DashboardStats, PaginatedResponse, ApiError) in frontend/src/types/api.ts
- [x] TF-015 [P] Define WebSocket message type interfaces (DetectionFrameMessage, PredictionUpdateMessage, AnomalyAlertMessage, AnomalyStatusChangeMessage, CameraStatusMessage, HealthUpdateMessage, StorageWarningMessage, FilterUpdateMessage, CameraSubscribeMessage, ErrorMessage) in frontend/src/types/websocket.ts

### HTTP & WebSocket Clients

- [x] TF-016 Create base HTTP API client with session cookie handling, error formatting (ApiError → actionable user messages per FR-036), request/response interceptors, and base URL configuration in frontend/src/api/client.ts
- [x] TF-017 [P] Create WebSocket connection manager with exponential backoff reconnection (1s → 2s → 4s → 8s → 16s max, 10 retries → "Connection Lost" banner), ping/pong health monitor (30s/10s), and auto-resubscribe on reconnect in frontend/src/hooks/useWebSocket.ts

### UI Component Library (FR-033)

- [x] TF-018 [P] Implement Button component (variants: primary, secondary, danger, ghost; sizes: sm, md, lg; loading state; disabled state; keyboard focusable) in frontend/src/components/ui/Button.tsx
- [x] TF-019 [P] Implement Card component (header, body, footer slots; elevation variants; themed border) in frontend/src/components/ui/Card.tsx
- [x] TF-020 [P] Implement Modal component (overlay, content, close button, focus trap, Escape key dismiss, body scroll lock) in frontend/src/components/ui/Modal.tsx
- [x] TF-021 [P] Implement StatusBadge component (status: up/down/warning/error/connected/disconnected/reconnecting; severity: high/medium/low; themed colors) in frontend/src/components/ui/StatusBadge.tsx
- [x] TF-022 [P] Implement FormInput component (text, email, password types; label, error message, helper text; validation state styling) in frontend/src/components/ui/FormInput.tsx
- [x] TF-023 [P] Implement Toggle component (on/off switch; label; accessible aria-checked; themed track/thumb colors) in frontend/src/components/ui/Toggle.tsx
- [x] TF-024 [P] Implement LoadingSpinner component (size variants; accessible aria-busy; themed color) in frontend/src/components/ui/LoadingSpinner.tsx
- [x] TF-025 [P] Export all UI components from barrel file in frontend/src/components/ui/index.ts

### Layout Shell

- [x] TF-026 Implement Header component (app logo/branding, nav links: Dashboard/Camera Feed/Recordings/Health, theme toggle placeholder, user menu with logout, responsive) in frontend/src/components/layout/Header.tsx
- [x] TF-027 [P] Implement Sidebar component (collapsible nav, active route indicator, keyboard navigable) in frontend/src/components/layout/Sidebar.tsx
- [x] TF-028 Implement AppLayout wrapper component (Header + Sidebar + main content area with outlet) in frontend/src/components/layout/AppLayout.tsx

### App Shell & Routing

- [x] TF-029 Scaffold App.tsx with React Router routes (login, dashboard, cameras, predictions, recordings, recording/:id, health, settings), AppLayout wrapper for authenticated routes, and route guard placeholders in frontend/src/App.tsx
- [x] TF-030 [P] Create main.tsx entry point with React.StrictMode, RouterProvider, and theme CSS imports in frontend/src/main.tsx

### Tests for Foundational Phase

- [x] TF-031 [P] Write unit tests for all UI components (Button, Card, Modal, StatusBadge, FormInput, Toggle, LoadingSpinner) verifying variants, states, keyboard interaction, and theme token usage in frontend/tests/unit/components/ui/

**Checkpoint**: Foundation ready — project builds, all UI components render in all 3 themes, routing works, HTTP client configured, WebSocket hook ready. User story implementation can now begin.

---

## Phase 3: User Story 1 — Instructor Login (Priority: P1) 🎯 MVP

**Goal**: Instructors authenticate with email/password, get redirected to dashboard showing session counts / camera status / recent anomalies / quick-actions, and can log out. Invalid credentials show non-specific error.

**Independent Test**: Navigate to /login, enter valid/invalid credentials, verify redirect to /dashboard or error display. Verify session persistence on reload and logout redirect.

### Tests for User Story 1 ⚠️

> **NOTE: Write these tests FIRST, ensure they FAIL before implementation**

- [x] TF-032 [P] [US1] Write unit tests for authStore (login, logout, getMe, session restore, error state) in frontend/tests/unit/stores/authStore.test.ts
- [x] TF-033 [P] [US1] Write unit tests for LoginPage (form validation, submit, error display, redirect on success, redirect if already authenticated) in frontend/tests/unit/components/LoginPage.test.tsx
- [x] TF-034 [P] [US1] Write unit tests for DashboardPage (stats cards render, loading states, error states, quick-action buttons) in frontend/tests/unit/components/DashboardPage.test.tsx

### Implementation for User Story 1

- [x] TF-035 [P] [US1] Create auth API service (login, logout, getMe, updateProfile) consuming POST /auth/login/, POST /auth/logout/, GET /auth/me/, PATCH /auth/me/ in frontend/src/api/auth.ts
- [x] TF-036 [P] [US1] Create dashboard API service (getDashboardStats) consuming GET /api/v1/dashboard/ in frontend/src/api/dashboard.ts
- [x] TF-037 [US1] Create Zustand auth store (user state, isAuthenticated, isLoading, error, login/logout/getMe/clearError actions) in frontend/src/stores/authStore.ts
- [x] TF-038 [US1] Create useAuth hook (session restoration on app load via getMe, redirect logic for protected routes) in frontend/src/hooks/useAuth.ts
- [x] TF-039 [US1] Implement route guards — ProtectedRoute (redirect to /login if unauthenticated) and PublicRoute (redirect to /dashboard if authenticated) — in frontend/src/components/auth/RouteGuards.tsx
- [x] TF-040 [US1] Integrate route guards into App.tsx route definitions in frontend/src/App.tsx
- [x] TF-041 [US1] Implement LoginPage with full-bleed dark gradient background, centered card layout, logo/branding, email + password form (FormInput), "Sign In" button with loading state, non-specific error display ("Invalid email or password"), and redirect to /dashboard on success in frontend/src/pages/LoginPage.tsx
- [x] TF-042 [US1] Implement DashboardPage (FR-046) with: active monitoring session count card, camera status summary card (connected/disconnected), recent anomaly alerts section (last 24h with severity breakdown and count), quick-action navigation buttons (Camera Feed, Recordings), loading states, and error handling in frontend/src/pages/DashboardPage.tsx
- [x] TF-043 [US1] Add logout action to Header user menu (calls auth store logout, redirects to /login) in frontend/src/components/layout/Header.tsx

**Checkpoint**: US1 fully functional — instructors log in, see dashboard with stats, log out. Independently testable.

---

## Phase 4: User Story 2 — Live Camera Feed Monitoring (Priority: P1) 🎯 MVP

**Goal**: Instructors connect RTSP cameras, view real-time video feeds via go2rtc WebRTC in a responsive grid, see bounding boxes with class-colored overlays and persistent student tracking IDs via WebSocket push. Auto-reconnection on connection drop.

**Independent Test**: Connect to a camera, verify feed renders in real-time, confirm bounding boxes + tracking IDs display. Disconnect camera, verify "Reconnecting..." status and auto-retry.

### Tests for User Story 2 ⚠️

> **NOTE: Write these tests FIRST, ensure they FAIL before implementation**

- [x] TF-044 [P] [US2] Write unit tests for cameraStore (add/remove/update camera, status transitions, grid layout computation) in frontend/tests/unit/stores/cameraStore.test.ts
- [x] TF-045 [P] [US2] Write unit tests for detectionStore (add frame, clear frames, frame buffer management) in frontend/tests/unit/stores/detectionStore.test.ts
- [x] TF-046 [P] [US2] Write unit tests for BoundingBoxCanvas (draws boxes with correct colors, renders tracking IDs, clears on empty detections, respects canvas dimensions) in frontend/tests/unit/components/camera/BoundingBoxCanvas.test.tsx
- [x] TF-047 [P] [US2] Write unit tests for CameraGrid (correct grid columns/rows for 1-8 cameras per FR-007 breakpoints) in frontend/tests/unit/components/camera/CameraGrid.test.tsx

### Implementation for User Story 2

- [x] TF-048 [P] [US2] Create camera API service (list, create, update, delete, connect, disconnect) consuming /api/v1/cameras/ endpoints in frontend/src/api/cameras.ts
- [x] TF-049 [P] [US2] Create session API service (list, create, end) consuming /api/v1/sessions/ endpoints in frontend/src/api/sessions.ts
- [x] TF-050 [US2] Create Zustand camera store (camera list, statuses, add/remove/update/connect/disconnect actions, grid layout computation for 1-8 cameras) in frontend/src/stores/cameraStore.ts
- [x] TF-051 [US2] Create Zustand detection store (live detections per camera keyed by camera_id, frame buffer with configurable max size, clear/reset actions) in frontend/src/stores/detectionStore.ts
- [x] TF-052 [US2] Implement useWhepClient hook — WHEP WebRTC client for go2rtc (~50 LOC: create RTCPeerConnection, POST SDP offer to go2rtc /api/webrtc, set remote SDP answer, attach MediaStream to <video> ref) in frontend/src/hooks/useWhepClient.ts
- [x] TF-053 [US2] Implement useDetectionSocket hook — connects to wss://{host}/ws/detections/{session_id}/, processes detection.frame and prediction.update messages, dispatches to detection store, handles camera.subscribe/unsubscribe in frontend/src/hooks/useDetectionSocket.ts
- [x] TF-054 [US2] Implement BoundingBoxCanvas component — <canvas> with requestAnimationFrame render loop, class-colored boxes (cyan=#22D3EE for students, purple=#A78BFA for teachers per R-009 tokens), persistent tracking ID labels, canvas resize observer, pointer-events: none overlay in frontend/src/components/camera/BoundingBoxCanvas.tsx
- [x] TF-055 [US2] Implement CameraFeed component — position:relative container with <video> (z-index:0, WebRTC stream from useWhepClient) + BoundingBoxCanvas overlay (z-index:1), connection status indicator (connected/reconnecting/disconnected badge), "Connection Lost — Reconnecting..." overlay on disconnect with retry count display in frontend/src/components/camera/CameraFeed.tsx
- [x] TF-056 [US2] Implement CameraGrid component — responsive CSS Grid layout auto-adjusting by feed count per FR-007 (1=full-width, 2=1×2, 3-4=2×2, 5-6=2×3, 7-8=2×4), viewport <1200px reduces columns by one tier, gap spacing, maintains aspect ratio in frontend/src/components/camera/CameraGrid.tsx
- [x] TF-057 [US2] Implement CameraFeedPage — session management (start/end session), camera list sidebar, CameraGrid main area, connects detection WebSocket on session start, camera status WebSocket for real-time status updates in frontend/src/pages/CameraFeedPage.tsx

**Checkpoint**: US1 + US2 fully functional — instructors log in, connect cameras, see live feeds with detection overlays. MVP functional.

---

## Phase 5: User Story 3 — Bounding Box Class Filtering (Priority: P2)

**Goal**: Instructors toggle visibility of bounding boxes per detection class to reduce visual clutter. ≤200ms effect. Session-persistent filter settings across camera feeds.

**Independent Test**: Connect camera, verify all class boxes shown by default, toggle one class off and confirm those boxes disappear within 200ms, toggle back on. Click "Show All" to reset.

### Tests for User Story 3 ⚠️

- [x] TF-058 [P] [US3] Write unit tests for ClassFilterPanel (toggle class, show all, min-confidence slider, renders all detection classes, persists filter state) in frontend/tests/unit/components/camera/ClassFilterPanel.test.tsx
- [x] TF-059 [P] [US3] Write unit tests for filter integration with BoundingBoxCanvas (hidden classes not drawn, shown classes drawn, filter change applies within render cycle) in frontend/tests/unit/components/camera/BoundingBoxFiltered.test.tsx

### Implementation for User Story 3

- [x] TF-060 [US3] Add filter state to detection store (per-class visibility toggles: Student, Teacher, Standing, Sitting, Looking Left, Looking Right, Looking Up, Looking Down, Looking Forward, Looking Backward; show-all action; min-confidence threshold; session persistence across camera switches) in frontend/src/stores/detectionStore.ts
- [x] TF-061 [US3] Implement ClassFilterPanel component — lists all detection classes with Toggle switches (all enabled by default), "Show All" button, min-confidence slider (0.0–1.0), class count indicators, keyboard navigable in frontend/src/components/camera/ClassFilterPanel.tsx
- [x] TF-062 [US3] Integrate filter state with BoundingBoxCanvas render loop — skip drawing hidden classes, respect min-confidence threshold, apply within current requestAnimationFrame cycle (<200ms per FR-015) in frontend/src/components/camera/BoundingBoxCanvas.tsx
- [x] TF-063 [US3] Add ClassFilterPanel to CameraFeedPage layout as a collapsible sidebar panel with toggle button in frontend/src/pages/CameraFeedPage.tsx

**Checkpoint**: Filtering works independently — instructors can show/hide detection classes on live feeds within 200ms.

---

## Phase 6: User Story 4 — Model Predictions & Anomaly Display (Priority: P2)

**Goal**: Predictions panel shows per-student pyramid outputs. Anomaly alerts with severity levels delivered via WebSocket. Instructors triage alerts (acknowledge/dismiss/annotate) with first-write-wins conflict handling. Full alert history view.

**Independent Test**: Connect feed with 2+ detected students, verify per-student prediction breakdown, trigger an anomaly, confirm alert appears with severity, acknowledge it, add a note, verify status change broadcasts to other viewers.

### Tests for User Story 4 ⚠️

> **NOTE: Write these tests FIRST, ensure they FAIL before implementation**

- [x] TF-064 [P] [US4] Write unit tests for anomalyStore (add alert, acknowledge, dismiss with reason, add note, first-write-wins rejection, status update from WebSocket) in frontend/tests/unit/stores/anomalyStore.test.ts
- [x] TF-065 [P] [US4] Write unit tests for PredictionsPanel (renders per-student cards, updates on new prediction.update messages, "No students detected" empty state) in frontend/tests/unit/components/detection/PredictionsPanel.test.tsx
- [x] TF-066 [P] [US4] Write unit tests for TriageActions (acknowledge button, dismiss opens reason input, min 5 chars validation, add note, disabled state on already-triaged, first-write-wins error display) in frontend/tests/unit/components/anomaly/TriageActions.test.tsx
- [x] TF-067 [P] [US4] Write unit tests for AlertHistory (renders all anomaly statuses, filters by status, shows dismissed reasons and notes) in frontend/tests/unit/components/anomaly/AlertHistory.test.tsx

### Implementation for User Story 4

- [x] TF-068 [P] [US4] Create anomaly API service (list, acknowledge, dismiss, addNote, getNotes) consuming /api/v1/anomalies/ endpoints in frontend/src/api/anomalies.ts
- [x] TF-069 [P] [US4] Create detection API service (getFrames, getPredictions) consuming /api/v1/detections/ endpoints for playback metadata queries in frontend/src/api/detections.ts
- [x] TF-070 [US4] Create Zustand anomaly store (active alerts sorted by severity/timestamp, history with all statuses, triage actions with optimistic updates + rollback on first-write-wins rejection, notes per anomaly, WebSocket status_change handler) in frontend/src/stores/anomalyStore.ts
- [x] TF-071 [US4] Implement useAnomalySocket hook — connects to wss://{host}/ws/anomalies/{session_id}/, processes anomaly.alert and anomaly.status_change messages, dispatches to anomaly store in frontend/src/hooks/useAnomalySocket.ts
- [x] TF-072 [US4] Implement StudentCard component — tracking ID header, posture indicator (standing/sitting icon + confidence), gaze direction indicators (horizontal/depth/vertical with confidence bars), constraint violation warning, highlight border on anomaly, compact and expanded view modes in frontend/src/components/detection/StudentCard.tsx
- [x] TF-073 [US4] Implement PredictionsPanel component — scrollable list of StudentCards keyed by tracking_id, real-time updates from prediction.update messages, "No students detected in frame" empty state, click-to-highlight integration (dispatches highlight event to BoundingBoxCanvas) in frontend/src/components/detection/PredictionsPanel.tsx
- [x] TF-074 [US4] Implement AnomalyAlert component — severity indicator (red=high, yellow=medium, blue=low per FR-017/FR-044), student ID, timestamp, behavioral description, prediction snapshot, click-to-highlight student on camera feed, status badge (new/acknowledged/dismissed) with themed colors across all 3 themes in frontend/src/components/anomaly/AnomalyAlert.tsx
- [x] TF-075 [US4] Implement TriageActions component — "Acknowledge" button (FR-041), "Dismiss" button opening reason textarea (min 5 chars, FR-042), "Add Note" expandable textarea (FR-043), disabled states when already triaged, first-write-wins rejection display ("Already handled by [instructor name]"), loading states per action in frontend/src/components/anomaly/TriageActions.tsx
- [x] TF-076 [US4] Implement AlertHistory component — full anomaly event list with all statuses (new/acknowledged/dismissed per FR-045), filterable by status and severity, shows dismissed reasons, expandable notes list per anomaly, instructor attribution, timestamps, sorted by recency in frontend/src/components/anomaly/AlertHistory.tsx
- [x] TF-077 [US4] Implement PredictionsPage — PredictionsPanel + active AnomalyAlert list + TriageActions per alert + AlertHistory section, connected via anomaly + detection stores and WebSocket hooks in frontend/src/pages/PredictionsPage.tsx
- [x] TF-078 [US4] Integrate AnomalyAlert click-to-highlight with BoundingBoxCanvas — clicking an alert highlights the student's bounding box (thicker border, pulsing glow) on the corresponding camera feed in frontend/src/components/camera/BoundingBoxCanvas.tsx

**Checkpoint**: Predictions and anomaly triage work end-to-end from frontend perspective — real-time predictions display, anomaly alerts with triage actions, alert history, click-to-highlight.

---

## Phase 7: User Story 6 — Video Recording & Playback (Priority: P2)

**Goal**: Instructors browse past sessions on Recordings page, play back recorded feeds with dynamic bounding box overlays rendered from stored metadata, jump to anomaly timeline markers, filter playback by student ID. Session export trigger with progress tracking.

**Independent Test**: Navigate to Recordings, verify session listed, play back video, confirm bounding boxes render, click anomaly marker to jump, filter by student ID, initiate export.

### Tests for User Story 6 ⚠️

> **NOTE: Write these tests FIRST, ensure they FAIL before implementation**

- [x] TF-079 [P] [US6] Write unit tests for RecordingPlayer (video loads, canvas overlay renders detection data, seek controls work, anomaly markers displayed) in frontend/tests/unit/components/recording/RecordingPlayer.test.tsx
- [x] TF-080 [P] [US6] Write unit tests for AnomalyTimeline (markers render at correct positions, click jumps video to timestamp, severity-colored markers) in frontend/tests/unit/components/recording/AnomalyTimeline.test.tsx
- [x] TF-081 [P] [US6] Write unit tests for ExportButton (initiates export, shows progress, download link on completion, error display on failure) in frontend/tests/unit/components/recording/ExportButton.test.tsx

### Implementation for User Story 6

- [x] TF-082 [P] [US6] Create recording API service (list, getStream URL, getAnomalyMarkers) consuming /api/v1/recordings/ endpoints in frontend/src/api/recordings.ts
- [x] TF-083 [P] [US6] Create export API service (initiateExport, getExportStatus, getDownloadUrl) consuming /api/v1/sessions/{id}/export/ endpoints in frontend/src/api/exports.ts
- [x] TF-084 [US6] Implement RecordingList component — session browser with date, duration, camera name, anomaly count, status badge, search/filter by camera and date range, paginated list, loading states in frontend/src/components/recording/RecordingList.tsx
- [x] TF-085 [US6] Implement RecordingPlayer component — <video> element with Range request source for seeking (<500ms per SC-008), <canvas> overlay rendering bounding boxes from stored detection metadata (fetched via /detections/frames/ for current playback time window), play/pause/seek controls, playback speed selector, student ID filter dropdown that isolates one student's boxes and predictions (FR-030) in frontend/src/components/recording/RecordingPlayer.tsx
- [x] TF-086 [US6] Implement AnomalyTimeline component — horizontal timeline bar with severity-colored markers positioned by timestamp_offset_seconds, clickable markers that seek video to the exact anomaly moment (FR-029), hover tooltip showing severity + description + tracking ID, scrollable if many markers in frontend/src/components/recording/AnomalyTimeline.tsx
- [x] TF-087 [US6] Implement ExportButton component — triggers POST /sessions/{id}/export/, shows progress bar polling GET /sessions/{id}/export/{export_id}/ status, download link on completion, error display on failure, disabled when export already in progress (409) in frontend/src/components/recording/ExportButton.tsx
- [x] TF-088 [US6] Implement RecordingsPage — RecordingList with search/sort/filter, navigation to recording detail in frontend/src/pages/RecordingsPage.tsx
- [x] TF-089 [US6] Implement RecordingDetailPage — RecordingPlayer + AnomalyTimeline + student filter + ExportButton, fetches recording metadata + anomaly markers on mount in frontend/src/pages/RecordingDetailPage.tsx

**Checkpoint**: Recording and playback work — sessions browsable, playback renders dynamic overlays, anomaly markers clickable, student filtering works, export trigger functional.

---

## Phase 8: User Story 5 — Theme Switching (Priority: P3)

**Goal**: Three themes (Black+Purple default, Full Black, White) switchable instantly (<300ms) from header toggle. Persists in localStorage and syncs to backend. WCAG 2.1 AA compliant. Bounding box colors distinguishable in all themes.

**Independent Test**: Log in, switch to each theme, verify all components render correctly, close browser, reopen, confirm theme persists. Run contrast ratio validation.

### Tests for User Story 5 ⚠️

- [x] TF-090 [P] [US5] Write unit tests for themeStore (default theme, switch theme, localStorage persistence, apply data-theme attribute) in frontend/tests/unit/stores/themeStore.test.ts
- [x] TF-091 [P] [US5] Write integration tests for theme switch (apply Black+Purple → Full Black → White, verify data-theme attribute, verify localStorage written, verify no FOUC on reload) in frontend/tests/integration/theme.test.tsx

### Implementation for User Story 5

- [x] TF-092 [US5] Create Zustand theme store (currentTheme, setTheme action, localStorage read/write, apply data-theme attribute to document.documentElement, sync to backend PATCH /auth/me/ on change) in frontend/src/stores/themeStore.ts
- [x] TF-093 [US5] Implement ThemeSelector component — 3-option selector (radio group or segmented control) with theme name + preview swatch per option, accessible keyboard navigation, current selection indicator in frontend/src/components/layout/ThemeSelector.tsx
- [x] TF-094 [US5] Integrate ThemeSelector into Header component (replace theme toggle placeholder with ThemeSelector in dropdown or popover) in frontend/src/components/layout/Header.tsx
- [x] TF-095 [US5] Initialize theme on app load — read from localStorage first, fall back to user profile theme_preference from /auth/me/, fall back to "black-purple" default in frontend/src/App.tsx
- [x] TF-096 [US5] Audit all UI components and pages for correct CSS custom property usage — verify no hardcoded colors, all theme tokens applied, bounding box overlay colors (--color-bbox-student, --color-bbox-teacher) remain distinguishable in all themes

**Checkpoint**: All three themes work, switch within 300ms, persist across sessions, and pass WCAG 2.1 AA contrast for text and interactive elements.

---

## Phase 9: User Story 7 — Multi-Camera Management (Priority: P3)

**Goal**: Instructors add/remove/rename multiple cameras from a management dialog. Grid supports full-screen toggle (double-click). Disconnecting one camera doesn't affect others. Camera configs persist server-side.

**Independent Test**: Add two cameras, verify both render, rename one, disconnect one (other continues), double-click for full-screen toggle, log out/in verify configs restored.

### Tests for User Story 7 ⚠️

- [x] TF-097 [P] [US7] Write unit tests for AddCameraDialog (RTSP URL validation, name field, submit, error display, max 8 cameras validation) in frontend/tests/unit/components/camera/AddCameraDialog.test.tsx
- [x] TF-098 [P] [US7] Write unit tests for camera grid full-screen toggle (double-click expand, double-click restore, keyboard Escape restores) in frontend/tests/unit/components/camera/CameraGridFullscreen.test.tsx

### Implementation for User Story 7

- [x] TF-099 [US7] Implement AddCameraDialog component — Modal with RTSP URL input (with rtsp:// validation), local device selector dropdown (if supported via navigator.mediaDevices), name field, "Add Camera" button with loading state, max 8 cameras validation error, shared camera disconnect confirmation dialog in frontend/src/components/camera/AddCameraDialog.tsx
- [x] TF-100 [US7] Add camera management controls to CameraFeedPage — "Add Camera" button opening AddCameraDialog, per-camera context menu (rename inline edit, disconnect with confirmation, remove with confirmation), visual indicator for shared cameras in frontend/src/pages/CameraFeedPage.tsx
- [x] TF-101 [US7] Implement full-screen camera toggle — double-click on CameraFeed expands to full container while others collapse to small thumbnails strip, double-click again or Escape key restores grid layout, smooth CSS transition in frontend/src/components/camera/CameraGrid.tsx
- [x] TF-102 [US7] Implement per-camera independent disconnect — disconnecting one camera removes only its feed tile, remaining feeds reorganize into CameraGrid without interruption, other WebRTC connections and WebSocket subscriptions unaffected in frontend/src/components/camera/CameraFeed.tsx

**Checkpoint**: Multi-camera management works — instructors manage 1-8 cameras independently, full-screen toggle, configs persist.

---

## Phase 10: Polish & Cross-Cutting Concerns

**Purpose**: Health dashboard, settings page, keyboard navigation, accessibility, performance validation, documentation

### Health Dashboard (FR-038–FR-040)

- [x] TF-103 [P] Create health API service (getHealth, getHealthDashboard) consuming /api/v1/health/ and /api/v1/health/dashboard/ in frontend/src/api/health.ts
- [x] TF-104 [P] Create Zustand health store (health metrics, update from WebSocket health.update messages, storage warning state) in frontend/src/stores/healthStore.ts
- [x] TF-105 Implement useHealthSocket hook — connects to wss://{host}/ws/health/, processes health.update and storage.warning messages, dispatches to health store in frontend/src/hooks/useHealthSocket.ts
- [x] TF-106 Implement HealthPanel component — subsystem status cards (API, WebSocket, Detection Pipeline, Database, Redis, Storage), FPS per camera gauge, storage usage bar with warning/critical colors, active session count, uptime display, auto-refresh ≤10s via WebSocket (FR-040) in frontend/src/components/health/HealthPanel.tsx
- [x] TF-107 Implement HealthPage with HealthPanel, storage warning banner integration in frontend/src/pages/HealthPage.tsx

### Settings Page

- [x] TF-108 Implement SettingsPage — theme selection (ThemeSelector), profile info display (email, name, role from auth store), session timeout info, keyboard shortcuts reference in frontend/src/pages/SettingsPage.tsx

### Keyboard Navigation (FR-035)

- [x] TF-109 Implement useKeyboardNav hook — global keyboard shortcuts (Ctrl+1-8 for camera focus, Escape for close modal/fullscreen, Tab order management, skip-to-content link) in frontend/src/hooks/useKeyboardNav.ts
- [x] TF-110 Audit all pages and interactive components for keyboard accessibility — ensure all buttons, toggles, modals, dialogs, menus are keyboard-navigable with visible focus indicators

### Storage Warning Integration

- [x] TF-111 Implement storage warning banner — displayed at top of all pages when storage reaches 80% (warning level) or 95% (critical level), with link to recordings page for cleanup, themed warning/critical colors, dismissible per session in frontend/src/components/layout/StorageWarningBanner.tsx
- [x] TF-112 Integrate StorageWarningBanner into AppLayout, connected to health store storage.warning state in frontend/src/components/layout/AppLayout.tsx

### Performance Validation

- [x] TF-113 Performance profiling — validate Canvas FPS (≥15 on 8 simultaneous feeds), WebSocket message throughput handling (~2400 msg/s without UI jank), theme switch timing (<300ms), recording seek (<500ms), all UI interactions (<200ms)

### E2E Tests

- [x] TF-114 [P] Write Playwright E2E test for login flow (valid credentials → dashboard, invalid → error, logout → login page) in frontend/tests/e2e/login.spec.ts
- [x] TF-115 [P] Write Playwright E2E test for camera feed (connect camera → verify video element → verify canvas overlay → disconnect) in frontend/tests/e2e/camera-feed.spec.ts
- [x] TF-116 [P] Write Playwright E2E test for anomaly triage (receive anomaly alert → acknowledge → verify status change → dismiss with reason → verify history) in frontend/tests/e2e/anomaly-triage.spec.ts
- [x] TF-117 [P] Write Playwright E2E test for recording playback (browse recordings → play → verify overlay → click anomaly marker → verify seek → filter by student) in frontend/tests/e2e/recording-playback.spec.ts

### Documentation

- [x] TF-118 Write frontend README.md with: project setup instructions, component tree overview, theme system documentation, state management architecture (Zustand stores), WebSocket integration docs, build/test/lint commands, Mermaid component dependency diagram in frontend/README.md

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup (Phase 1) completion — **BLOCKS all user stories**
- **User Stories (Phases 3–9)**: All depend on Foundational (Phase 2) completion
  - User stories can then proceed in parallel (if staffed)
  - Or sequentially in priority order (P1 → P2 → P3)
- **Polish (Phase 10)**: Depends on all desired user stories being complete

### User Story Dependencies

| Story | Phase | Priority | Can Start After | Dependencies on Other Stories |
|-------|-------|----------|-----------------|-------------------------------|
| US1 — Login | 3 | P1 | Phase 2 (Foundational) | None |
| US2 — Camera Feed | 4 | P1 | Phase 2 (Foundational) | None (but US1 provides auth + route guards) |
| US3 — Filtering | 5 | P2 | Phase 2 (Foundational) | Soft dependency on US2 (needs BoundingBoxCanvas + detectionStore) |
| US4 — Predictions | 6 | P2 | Phase 2 (Foundational) | Soft dependency on US2 (needs detection WebSocket + CameraFeedPage) |
| US6 — Recording | 7 | P2 | Phase 2 (Foundational) | None (uses own API services; independent playback) |
| US5 — Themes | 8 | P3 | Phase 2 (Foundational) | None (theme tokens set up in Phase 2, this phase adds store + toggle) |
| US7 — Multi-Camera | 9 | P3 | Phase 2 (Foundational) | Soft dependency on US2 (extends CameraGrid + CameraFeed) |

### Within Each User Story

1. Tests MUST be written and FAIL before implementation (TDD: Red → Green → Refactor)
2. API services before stores (stores call API services)
3. Stores before hooks (hooks use stores)
4. Hooks before components (components use hooks)
5. Components before pages (pages compose components)
6. Story complete before moving to next priority

### Parallel Opportunities

**Within Phase 2 (Foundational):**
- TF-008 is sequential (core tokens first)
- TF-009–TF-013 can ALL run in parallel after TF-008 (different CSS files)
- TF-014–TF-015 can run in parallel (different type files)
- TF-016–TF-017 can run in parallel (different service files)
- TF-018–TF-025 can ALL run in parallel (different UI component files)
- TF-026–TF-028 layout components (TF-026 first, TF-027 parallel, TF-028 depends on both)
- TF-029–TF-030 app shell (parallel, different files)

**Once Foundational completes, stories can run in parallel:**
- Developer A: US1 (Login) + US5 (Themes) — auth/UI focused
- Developer B: US2 (Camera Feed) + US3 (Filtering) + US7 (Multi-Camera) — camera-related
- Developer C: US4 (Predictions/Anomalies) + US6 (Recording) — data display stories

**Within each story**, tasks marked [P] can run in parallel.

---

## Parallel Example: User Story 2

```bash
# Batch 1 — Tests (write first, ensure they fail):
TF-044: Unit tests for cameraStore                    [P] [US2]
TF-045: Unit tests for detectionStore                 [P] [US2]
TF-046: Unit tests for BoundingBoxCanvas              [P] [US2]
TF-047: Unit tests for CameraGrid                     [P] [US2]

# Batch 2 — API services (parallel, different files):
TF-048: Camera API service                            [P] [US2]
TF-049: Session API service                           [P] [US2]

# Batch 3 — Stores (parallel, different files):
TF-050: Camera store                                  [US2]
TF-051: Detection store                               [US2]

# Batch 4 — Hooks (depends on stores):
TF-052: useWhepClient hook                            [US2]
TF-053: useDetectionSocket hook                       [US2]

# Batch 5 — Components (sequential: low-level → high-level):
TF-054: BoundingBoxCanvas                             [US2]
TF-055: CameraFeed                                    [US2]
TF-056: CameraGrid                                    [US2]

# Batch 6 — Page (depends on all components):
TF-057: CameraFeedPage                                [US2]
```

---

## Parallel Example: User Story 4

```bash
# Batch 1 — Tests (write first, ensure they fail):
TF-064: Unit tests for anomalyStore                   [P] [US4]
TF-065: Unit tests for PredictionsPanel               [P] [US4]
TF-066: Unit tests for TriageActions                  [P] [US4]
TF-067: Unit tests for AlertHistory                   [P] [US4]

# Batch 2 — API services (parallel, different files):
TF-068: Anomaly API service                           [P] [US4]
TF-069: Detection API service                         [P] [US4]

# Batch 3 — Store + hook:
TF-070: Anomaly store                                 [US4]
TF-071: useAnomalySocket hook                         [US4]

# Batch 4 — Components (parallel, different files):
TF-072: StudentCard                                   [US4]
TF-073: PredictionsPanel                              [US4]
TF-074: AnomalyAlert                                  [US4]
TF-075: TriageActions                                  [US4]
TF-076: AlertHistory                                  [US4]

# Batch 5 — Page + integration:
TF-077: PredictionsPage                               [US4]
TF-078: Click-to-highlight integration                [US4]
```

---

## Implementation Strategy

### MVP First (User Stories 1 + 2 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL — blocks all stories)
3. Complete Phase 3: User Story 1 (Login + Dashboard)
4. Complete Phase 4: User Story 2 (Camera Feed + Detection Overlays)
5. **STOP and VALIDATE**: Test US1 + US2 independently — instructor logs in, sees dashboard, connects camera, views live feed with bounding boxes
6. Deploy/demo if ready — this is a functional frontend MVP

### Incremental Delivery

1. Setup + Foundational → Foundation ready (themed UI shell, routing, all shared components)
2. Add US1 (Login + Dashboard) → Test independently → **Authentication + overview works**
3. Add US2 (Camera Feed) → Test independently → **Live monitoring MVP!**
4. Add US3 (Filtering) → Test independently → **Reduced visual clutter**
5. Add US4 (Predictions/Anomalies) → Test independently → **Anomaly intelligence + triage**
6. Add US6 (Recording) → Test independently → **Post-exam review + export**
7. Add US5 (Themes) → Test independently → **Full theme switching**
8. Add US7 (Multi-Camera) → Test independently → **Scale to exam halls**
9. Polish phase → **Production-ready frontend**

Each story adds value without breaking previous stories.

---

## Notes

- [P] tasks = different files, no dependencies on incomplete tasks in the same phase
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Verify tests fail before implementing (TDD: Red → Green → Refactor)
- **ALL test cases (unit + integration + E2E) MUST pass** before marking any task or story as complete — partial passes are a blocking failure
- **Commit after each task** with Conventional Commits: `type(scope): description` — vague messages prohibited
- **Create/update `.md` documentation** for every module — documentation is created alongside code
- **Update the root `README.md`** whenever modules, features, or setup steps change
- **Mermaid diagrams** MUST be colorful, readable, and updated when architecture changes
- Stop at any checkpoint to validate story independently
- Avoid: vague tasks, same-file conflicts, cross-story dependencies that break independence
- **No hardcoded colors** — all colors via CSS custom properties (theme tokens)
- **No hardcoded API URLs** — all via VITE_API_BASE_URL / VITE_WS_BASE_URL env vars
- **Accessible by default** — all interactive elements keyboard-navigable, WCAG 2.1 AA contrast, aria attributes
- **Error boundaries** — React error boundaries at page level to prevent full-app crashes
- **Constitution**: All 6 principles must be upheld — SOLID, ≥80% coverage, UX consistency, security standards, 3 themes, performance targets
