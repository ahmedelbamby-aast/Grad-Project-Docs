# Feature Specification: Exam Monitoring Dashboard

**Feature Branch**: `001-exam-monitor-dashboard`
**Created**: 2026-02-27
**Status**: Draft
**Input**: User description: "Full Stack Application to help instructors at universities track unusual and inappropriate behaviors at exam halls during exams. Clean, colorful UI with Black+Purple themes, full Black theme, White theme with theme switching. Login page for instructors. Camera feed page supporting single or multiple cameras. Bounding boxes per detection class (show/hide), student tracking IDs, model predictions display (especially Anomaly Behavior Analyzer). Video feeds recorded and stored with predictions, analyses, and visualizations for future investigation."

## Assumptions

- **Authentication method**: Email or username login with password for instructors and admins (standard session-based auth). The login page accepts either email or username as the identifier. University SSO integration may be added later but is out of initial scope.
- **Data retention**: Single policy for initial release — recordings, detections/predictions, anomaly artifacts, and session export artifacts are retained until the end of the academic term, then purged together by an automated retention job. Instructors can still manually delete recordings earlier when needed.
- **Privacy policy**: University-internal-only. All recorded data and predictions MUST remain within university-controlled systems. No data is shared externally. Configurable retention with automatic purge after term end. No formal FERPA/GDPR compliance scope — the system relies on university IT governance policies. Exports include full non-anonymized student tracking data (IDs, predictions, anomaly details) for complete traceability during academic integrity investigations.
- **Camera protocol**: RTSP is the primary protocol for IP camera connections. Local USB/webcam devices are also supported.
- **Detection models**: The backend already provides (or will provide) a YOLO-based pyramid detection pipeline. This spec focuses on the full-stack application that consumes detection outputs and presents them to instructors.
- **User roles**: Two roles from initial release — Instructor and Admin. Both share a single User entity (no separate Admin table); differentiation is by assigned role. The first admin is bootstrapped via a CLI management command (e.g., `createsuperuser`). Admin has elevated permissions: user management, role creation with configurable access ranges, exam management, recording review across all instructors, and triage action revert. Custom roles with configurable access ranges (page-level and action-level permissions) can be created by admins and assigned to users. No self-registration — all accounts are created by admins.
- **Exam scheduling**: Exam metadata (subject name, subject code, subject type, credit hours, expected student count, student name roster) is entered by admins within the application. No external university system integration (e.g., SIS API or CSV import) in initial scope. Exam data is displayed read-only to instructors on the Exam Board page.
- **Student rosters**: Student names and university IDs are entered per-exam by admins. Students are tracked during sessions by detection-assigned tracking IDs (not university IDs). The roster serves as a reference checklist for expected attendance.
- **Browser support**: Modern evergreen browsers (Chrome, Firefox, Edge — latest 2 versions). Mobile/tablet view is not in initial scope.
- **Detection processing rate**: The system processes and stores detection results at ~10–15 frames per second per camera. At this rate, with up to 8 cameras, the system may produce ~80–120 detection frames/second. The backend MUST batch detection frames into sub-second micro-batches (100ms window) before sending via WebSocket, reducing WS message count to ~8–10 messages/sec. Time-partitioned database storage is required for Detection Frame data.
- **Concurrent users**: Initial target is up to 20 simultaneous instructors, each monitoring 1-8 camera feeds.
- **Availability**: Auto-recovery model — a process supervisor (e.g., systemd, Docker restart policy) monitors all backend services and auto-restarts any crashed process within 30 seconds. No redundant/failover architecture required. Recordings and in-progress data MUST survive service restarts without corruption.
- **Dependency management**: Backend Python dependencies use `pip-tools` workflow: `requirements.in` contains compatible version ranges (e.g., `Django>=5.0,<5.1`), compiled to a fully pinned `requirements.txt` via `pip-compile` for reproducible builds. Both files are committed to version control. To update dependencies: edit ranges in `.in`, re-run `pip-compile`.

## Clarifications

### Session 2026-02-27

- Q: What communication protocol should the frontend use to receive real-time detection results and anomaly alerts from the backend? → A: WebSocket — persistent bidirectional connection; backend pushes detection frames and anomaly alerts in real-time.
- Q: What privacy compliance framework governs student monitoring data? → A: University-internal-only policy — data stays within university systems; configurable retention; no external sharing. *(Export anonymization originally stated here was superseded — see Clarification Round 2: exports are non-anonymized for full investigative traceability.)*
- Q: How should the system expose its own operational health and observability status? → A: Health dashboard page — dedicated page accessible to both instructors and admins showing backend API status, WebSocket connection health, detection pipeline FPS, storage usage, and active sessions count.
- Q: How should video recordings store detection data — baked into video or separate? → A: Raw video + separate metadata — store original video without overlays; prediction data stored separately; overlays rendered dynamically during playback.
- Q: What actions can instructors take on flagged anomaly alerts? → A: Full triage without escalation — acknowledge (marks as reviewed), dismiss (with mandatory reason), and annotate with timestamped free-text notes. No escalation workflow.
- Q: Should the login endpoint have brute force protection (rate limiting, account lockout, CAPTCHA)? → A: No — internal graduation project; no application-level brute force protection required.
- Q: Can multiple instructors connect to the same physical camera simultaneously? → A: Shared cameras — cameras are system-wide resources; any instructor can view any connected camera; single recording per camera.
- Q: What rules trigger an anomaly alert from pyramid predictions? → A: Duration thresholds AND multi-layer combinations, both configurable via a rules config file (e.g., "looking left" >10s = medium severity; "looking left" + "turned posture" together = high severity).

### Session 2026-02-27 (Clarification Round 2)

- Q: At what rate should the system process and store detection results per camera? → A: ~10–15 fps — near-real-time processing; requires WebSocket message batching and database partitioning to manage storage volume.
- Q: What happens when two instructors try to triage the same anomaly simultaneously? → A: First-write-wins — once an anomaly is acknowledged or dismissed, the status is locked; a second instructor attempting to triage sees "already handled" with the first instructor's action. Real-time WebSocket broadcast notifies all viewers of the status change.
- Q: What is the scope of data export for the initial release? → A: Full session export — non-anonymized anomaly data + video recordings as downloadable archive. All student tracking IDs and prediction data are preserved as-is. No anonymization applied to exports.
- Q: How should anomaly events relate to recordings? → A: Via session + camera (derive the recording) — anomaly events use their existing `session_id` + `camera_source_id` to derive the associated recording; no direct `recording_id` FK needed.
- Q: What is the system availability expectation? → A: Auto-recovery — process supervisor auto-restarts crashed services within 30 seconds; no data corruption on crash; recordings survive service restart.

### Session 2026-02-27 (Refinement)

- Resolved: Camera count inconsistency — Assumptions aligned to "1-8" matching FR-007 and implementation plan.
- Resolved: Bounding box coloring — Updated from "purple for students, gray for teachers" to "cyan for students, purple for teachers" to match research R-009 design tokens (`--color-bbox-student: #22D3EE`, `--color-bbox-teacher: #A78BFA`).
- Resolved: Dashboard page undefined — Added dashboard content description to US1 narrative + acceptance scenario, and new FR-046 defining dashboard requirements.
- Resolved: Grid layout breakpoints undefined — Added explicit grid rules to FR-007 (1→full, 2→1×2, 3–4→2×2, 5–6→2×3, 7–8→2×4).
- Resolved: "Visually striking" unmeasurable — Replaced with concrete visual criteria (full-bleed gradient, centered card, logo branding).
- Resolved: Instructor registration unspecified — Noted in Assumptions that accounts are created via the admin panel.

### Session 2026-02-27 (Clarification Round 3)

- Q: How should WebSocket connections (detection feed, anomaly alerts, health updates) be authenticated? → A: Session cookie passthrough — Django validates the existing session cookie during the WebSocket upgrade handshake. No separate token or auth message required.
- Q: How should the frontend authenticate with the go2rtc WHEP endpoint for WebRTC video streams? → A: Reverse proxy with cookie auth — go2rtc sits behind the same reverse proxy (nginx) as Django; the frontend's session cookie is validated on WHEP requests.
- Q: What lifecycle states should a session export job go through? → A: Full lifecycle with progress — pending → processing (with progress %) → completed/failed. Async polling with progress feedback; archives can be large.
- Q: Should the UI use "Monitoring Session" or "Recording" as the primary navigable concept? → A: Session ⊃ Recordings — "Monitoring Session" is the primary entity. Instructors browse sessions (SessionListPage), then drill into individual camera recordings within a session (SessionDetailPage). Recordings are children of sessions.
- Q: How should WebSocket detection message volume (80–120 frames/sec for 8 cameras) be managed? → A: Backend micro-batching — the backend batches detection frames into sub-second micro-batches (e.g., every 100ms) before sending via WebSocket, reducing message count from ~80–120/sec to ~8–10/sec.
- Q: What triggers a Monitoring Session's "interrupted" status? → A: Any abnormal termination — server crash beyond recovery, unrecoverable network loss, or manual abort. Browser disconnect alone does NOT interrupt (recording continues server-side per FR-032).
- Q: Should the frontend implement centralized error tracking beyond React error boundaries? → A: Error boundaries only — React error boundaries at page level + structured console.error logging. No external service (Sentry). Browser DevTools suffice for debugging in this graduation project.
- Q: What maximum character lengths should be enforced for anomaly dismiss reasons and notes? → A: No maximum — only enforce the existing minimum 5 characters for dismiss reason. Notes have no length constraint.
- Q: Which vocabulary should be canonical for anomaly alert status values? → A: Use spec terms everywhere — `new` / `acknowledged` / `dismissed`. Frontend stores, components, and types MUST use these exact values; no mapping layer (e.g., open/resolved) permitted. Code vocabulary MUST match spec and REST API.
- Q: Should the Predictions Panel be available as both a collapsible sidebar on CameraFeedPage AND a standalone page? → A: Yes, both modes are required per FR-016. The sidebar provides at-a-glance monitoring alongside live feeds; the page allows detailed analysis. A task must be added to integrate PredictionsPanel as a collapsible sidebar on CameraFeedPage.

### Session 2026-02-27 (Clarification Round 4)

- Q: Does connecting a camera automatically start a monitoring session (and recording), or are these separate actions? → A: Two-step — connecting a camera shows a live preview feed (no recording, no disk usage). The instructor must explicitly click "Start Session" to begin recording for the selected cameras. This allows camera preview without storage cost.
- Q: When two instructors view the same shared camera, how do sessions and recordings relate? → A: Own session, shared recording — each instructor has their own monitoring session (for triage tracking and audit trail), but the camera produces only ONE recording (deduplicated). Sessions reference the shared recording. Anomaly events are per-camera (not per-session); any instructor can triage them.

### Session 2026-02-27 (Clarification Round 5)

- Q: Should the login page accept both email and username? → A: Yes — the login page accepts either email or username along with password. The system resolves the identifier to the correct account. Both Admin and Instructor use the same login page, with role-based redirect after authentication.
- Q: What does "access range" mean for custom roles? → A: Page-level permissions (which pages/sections are accessible) combined with action-level permissions (e.g., can_triage_anomalies, can_manage_cameras, can_export_sessions). Role assignment is per-instructor. An instructor cannot access pages or perform actions outside their assigned role's range.
- Q: How is exam data provided to the system? → A: Admins manually enter exam details (subject name, code, type, credit hours, student count, student roster with names and university IDs) through admin management pages. No external system integration or CSV import in initial scope.
- Q: What is the session start flow from the Exam Board? → A: When an instructor opens the Exam Board and selects an upcoming exam, a countdown timer (maximum 5 minutes) starts. The instructor can click "Start Now" at any time to begin the monitoring session. On session start, the system searches for already-registered and connected cameras, transitions to the Camera Feed page, and begins recording.
- Q: What does "revert teacher actions" mean for admins? → A: Admins can reset any anomaly triage action (acknowledge/dismiss) back to `new` status with a mandatory reason. Notes are append-only and cannot be reverted. The revert is logged in the audit trail with admin identity, timestamp, and reason. All connected viewers receive a WebSocket notification of the status change.
- Q: Can the teacher add general comments during a live session? → A: Yes — instructors can add timestamped free-text comments during an active session. These are general observations (e.g., "Student in row 3 appears nervous") not necessarily tied to a specific anomaly. Comments are stored with the session and camera reference, and displayed during playback alongside anomaly markers.

### Session 2026-02-27 (Clarification Round 6)

- Q: Is "Admin" a separate user entity or a User record with the Admin role? How is the first admin bootstrapped? → A: Single User entity — both admins and instructors are User records differentiated by their assigned role. The first admin account is seeded via a CLI management command (e.g., `createsuperuser`). No separate Admin table.
- Q: When does the exam countdown appear and what happens at zero? → A: The Exam Board always shows the actual remaining time for upcoming exams (e.g., "Exam starts in 25 minutes"). At T−5 minutes the timer becomes a prominent countdown. When the countdown reaches zero the session auto-starts (no confirmation dialog). If the instructor joins after the exam start time, they see a "Start Now" button to manually create the session.
- Q: Can multiple instructors be assigned to the same exam, and how do their sessions interact? → A: Yes — multiple instructors can be assigned to one exam. During the live session each instructor works independently (own camera selections, own notes, own triage actions). After the exam finishes the system merges all instructor contributions into a unified session history with full attribution. The recording/history page shows all notes, triage actions, and comments from every assigned instructor, each tagged with the performing instructor’s identity. Admins can see exactly which instructor performed which action.
- Q: What actions does the audit log capture, and is AuditLog a formal entity? → A: The audit log captures all state-changing actions: anomaly triage (acknowledge/dismiss), admin reverts, role changes, account create/edit/deactivate, exam create/edit, session start/end, and login/logout events. AuditLog is a formal Key Entity — an append-only table with a polymorphic action_type field.
- Q: How are cameras associated with exams for auto-detection on session start? → A: Room-based — exams are assigned to a room/hall, and cameras are registered to rooms. When a session auto-starts from the Exam Board, the system loads all cameras assigned to that exam’s room. This reflects the physical reality that cameras are fixed in exam halls.
- Q: Can users change or reset their password? → A: Admin-managed only. Users can change their own password from a profile/settings page (requires current password). There is no self-service "Forgot Password" email reset flow. If a user forgets their password, an admin resets it manually via the User Management page.
- Q: Do admins get real-time anomaly notifications on the dashboard across all sessions? → A: Yes — the Admin Dashboard shows a unified real-time anomaly feed displaying high and medium severity anomalies from all active sessions, each with a link to jump directly into the relevant live session. This gives admins system-wide alert visibility without needing to click into each session individually.
- Q: What does the Exam Board show when a session is already active? Can the session be ended from the Exam Board? → A: Active exams show an "In Progress" badge with elapsed time and a "Return to Session" link. Sessions can only be ended from the Camera Feed page — no end-session control on the Exam Board. This keeps session control centralized.
- Q: What constitutes "recent system activity" on the Admin Dashboard? → A: Recent system activity is a paginated feed of the latest Audit Log entries, with a configurable page size selector (10, 20, 30, 40, 50, 60, 70, 80, 90, 100, or All), filterable by action type. Default page size is 50. This reuses the Audit Log entity directly — no separate "activity" concept.
- Q: Can admins add session comments when observing a live session? → A: Yes — admins can add timestamped comments in any live session they’re observing, attributed to their admin identity (e.g., "Admin: Dr. Smith"). This aligns with admins having full capabilities in any session.
### Session 2026-02-28

- Q: Does the 30-minute inactivity timeout (FR-004) apply during active monitoring sessions where the instructor is observing but not clicking? → A: WebSocket heartbeat counts as activity — an open WebSocket connection (detection feed, anomaly channel, etc.) continuously resets the inactivity timeout. Instructors passively watching live feeds will NOT be logged out. The timeout only fires when no HTTP requests AND no active WebSocket connections exist for the session.
- Q: Should AnomalyNote be a standalone entity (16 total) or an embedded JSON array inside AnomalyEvent (15 total)? → A: 16 entities — AnomalyNote is a standalone entity with its own table. It has independent author attribution, timestamps, and a one-to-many relationship with AnomalyEvent. Spec and plan aligned to 16 entities.
- Q: When an exam's scheduled end time is reached during an active session, does the session end immediately or with a grace period? → A: 5-minute grace period with prompt. The system displays a persistent prompt: "Exam time has ended — End Session?" with a visible countdown and an "End Now" button. The instructor can click "End Now" to end immediately or simply continue working during the countdown. No "Extend" button is needed. If no action is taken within 5 minutes, the session auto-ends gracefully (all data saved, recordings closed). The edge case notification ("Session ended — exam time elapsed") fires after the grace period expires.
- Q: Should backend Python dependencies use pinned versions or version ranges? → A: Ranges in `requirements.in` + pinned `requirements.txt` via `pip-compile` (pip-tools). `requirements.in` expresses intent with compatible ranges (e.g., `Django>=5.0,<5.1`); `pip-compile` resolves and pins exact versions in `requirements.txt` for reproducibility. Both files are committed to version control.
- Q: What HTTP status codes and per-subsystem thresholds should the `/health` endpoint use? → A: Two-tier HTTP status: `200 OK` when all subsystems are healthy, `503 Service Unavailable` when any critical subsystem is unhealthy. JSON body includes per-subsystem status (`healthy`/`degraded`/`unhealthy`). Thresholds: DB connection fails = unhealthy, Redis unreachable = unhealthy, detection pipeline <5 FPS = degraded, storage >95% = degraded, storage >99% = unhealthy, WebSocket layer unreachable = unhealthy.
- Q: Which retention policy should govern monitoring artifacts for v1? → A: Single term-end policy — keep recordings, detections/predictions, anomaly artifacts, and export artifacts until the academic term ends, then purge them together.
- Q: Can multiple admins/instructors observe the same live monitoring session simultaneously? → A: Concurrent observers allowed — multiple admins and instructors can observe the same live session at the same time. Each observer's actions (comments, triage) are independently attributed to their identity. No single-controller lock is applied.
- Q: What is the canonical empty-state message when no cameras are found for a room on session start? → A: "No cameras registered for this room" — this is the canonical wording used in FR-056, edge cases, and all UI implementations. Options to manually enter a camera URL or contact the admin are shown alongside.

### Session 2026-02-28 (Clarification Round 2)

- Q: Who can manually delete recordings, given FR-031 promises an "option to delete old recordings" but no DELETE endpoint exists? → A: Admin-only deletion — only admins can delete completed recordings via an admin endpoint (`DELETE /api/v1/admin/recordings/{id}/`). Instructors cannot delete recordings; they can only download. Active recordings cannot be deleted. Automatic retention (30-day default) still applies independently.
- Q: Does admin password reset enforce a mandatory password change on the user's next login, or is it advisory only? → A: Enforced — a `must_change_password` boolean is added to the User model (default: `false`). When an admin resets a user's password, this flag is set to `true`. On next login, middleware redirects the user to a password-change page. The flag is cleared after a successful password change. Until the flag is cleared, all other pages return a redirect to the password-change form.
- Q: What is the canonical URL pattern the frontend uses to reach the go2rtc WHEP endpoint through the reverse proxy? → A: `/whep/{camera_id}/` — a top-level path separate from the Django `/api/v1/` namespace. Nginx proxies this to go2rtc's internal `http://go2rtc:1984/api/webrtc?src={camera_id}`. The session cookie is validated by nginx (via Django auth subrequest) before proxying. The frontend `useWhepClient` hook POSTs the SDP offer to `/whep/{camera_id}/`.

### Session 2026-02-28 (Clarification Round 3)

- Q: What UI controls appear during the exam-end 5-minute grace period — "End Now" only, or "End Now" plus "Extend 5 min"? → A: "End Now" button only — the instructor can click "End Now" to end immediately or simply continue working during the countdown. No "Extend" button is needed. If no action is taken, the session auto-ends after 5 minutes.
- Resolved: Terminology drift — Key Entity "Export Job" renamed to "SessionExport" to match data-model.md and plan.md canonical entity name.
- Resolved: `must_change_password` field missing from REST API contract — added to login response, `/auth/me/` response, and admin user serializers in rest-api.md.
- Resolved: `DELETE /api/v1/admin/recordings/{id}/` endpoint missing from REST API contract — added admin recording deletion endpoint to rest-api.md; task added to tasks.md.
- Resolved: Nginx WHEP proxy configuration not covered by any task — task added to tasks.md Phase 1.
- Resolved: `ForcePasswordChangeMiddleware` and password-change page not covered by tasks — tasks added to tasks.md Phase 2.
- Resolved: React error boundary component not covered by any task — task added to tasks.md Phase 14.
- Resolved: T022 description updated to explicitly cover FR-053 role enforcement on navigation.
- Resolved: T054b renumbered to T055; subsequent task IDs shifted by +1 for sequential consistency.
- Resolved: plan.md "6 WebSocket message types" corrected to "6 WebSocket channels, 13 message types".
- Resolved: plan.md "~40+ REST endpoints" corrected to "~66 REST endpoints".
- Resolved: SC-006 annotated as post-implementation UX validation goal (not automated test).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - User Login & Role-Based Landing (Priority: P1)

A user (instructor or admin) navigates to the application and is presented with a polished, colorful login page featuring the Black+Purple theme — full-bleed dark gradient background, centered card layout with the application logo and branding. The user enters their email or username and password, submits the form, and upon successful authentication is redirected based on their role: instructors land on the Exam Board page showing their scheduled exams, admins land on the Admin Dashboard showing system-wide overview. If credentials are invalid, a clear error message is displayed without revealing which field was incorrect.

**Why this priority**: Authentication is the gateway to all other features. Without login, no other functionality is accessible. Role-based routing ensures each user type immediately sees the most relevant content.

**Independent Test**: Can be fully tested by navigating to the login URL, entering valid/invalid credentials for both instructor and admin accounts, and verifying role-appropriate redirect or error display. Delivers secure access control as standalone value.

**Acceptance Scenarios**:

1. **Given** an instructor with a valid account, **When** they enter correct email (or username) and password and click "Sign In", **Then** they are redirected to the Exam Board page within 2 seconds with their session active.
2. **Given** an admin with a valid account, **When** they enter correct credentials and click "Sign In", **Then** they are redirected to the Admin Dashboard within 2 seconds.
3. **Given** a user entering incorrect credentials, **When** they click "Sign In", **Then** a non-specific error message ("Invalid email or password") is displayed and the form remains on the login page.
4. **Given** a user who is already authenticated, **When** they navigate to the login page, **Then** they are automatically redirected to their role-appropriate landing page.
5. **Given** an authenticated user, **When** their session expires or they click "Logout", **Then** they are redirected to the login page and their session is invalidated.
6. **Given** an authenticated instructor lands on the Exam Board, **When** the page loads, **Then** it displays their scheduled exams with subject details, or "No exams scheduled" if none exist.
7. **Given** an authenticated admin lands on the Admin Dashboard, **When** the page loads, **Then** it displays: count of active live sessions (with instructor names), connected camera count, recent system activity, and navigation to User Management, Exam Management, and Recording Review pages.

---

### User Story 2 - Live Camera Feed Monitoring (Priority: P1)

After logging in, an instructor navigates to the Camera Feed page. They can connect a single camera or multiple cameras by providing RTSP URLs or selecting local devices. The live video feed is displayed in real-time. For multiple cameras, feeds are arranged in a responsive grid layout. On each feed, bounding boxes are drawn around detected entities (teachers, students) with color-coding by class. Each tracked student displays a persistent tracking ID overlaid on the bounding box.

**Why this priority**: This is the core value proposition — real-time exam monitoring with visual detection overlays. Without the camera feed, the system has no purpose.

**Independent Test**: Can be tested by connecting to a single RTSP camera URL (or local webcam), verifying the feed renders in real-time, and confirming bounding boxes and IDs are visible. Delivers real-time visual monitoring as standalone value.

**Acceptance Scenarios**:

1. **Given** an instructor on the Camera Feed page, **When** they enter a valid RTSP URL and click "Connect", **Then** the live video feed appears within 3 seconds with bounding boxes rendered on detected entities.
2. **Given** a connected camera feed, **When** students and teachers are detected, **Then** bounding boxes are displayed with distinct colors per class (cyan for students, purple for teachers per the design token palette) and each student box shows a persistent tracking ID label.
3. **Given** multiple cameras connected, **When** the instructor views the feed page, **Then** all feeds are displayed in a responsive grid layout (2x2, 3x3, etc.) that auto-adjusts based on the number of feeds and available screen space.
4. **Given** a camera feed is active, **When** the camera connection drops, **Then** the feed tile shows a "Connection Lost — Reconnecting..." status with automatic reconnection using exponential backoff (1s → 2s → 4s → 8s → 16s intervals, up to 5 retries). After all retries are exhausted, the tile shows a persistent "Connection Lost" status with a manual retry option.

---

### User Story 3 - Bounding Box Class Filtering (Priority: P2)

While viewing a live camera feed, an instructor can toggle the visibility of bounding boxes for specific detection classes. A control panel (sidebar or overlay) lists all available detection classes (e.g., Teacher, Student, Standing, Sitting, Looking Left, Looking Right, etc.) with toggle switches. The instructor can show or hide any class to reduce visual clutter and focus on specific behaviors of interest.

**Why this priority**: Enhances usability by letting instructors focus attention. Without filtering, the feed could become visually overwhelming with too many overlaid boxes.

**Independent Test**: Can be tested by connecting a camera, verifying all class boxes are shown by default, toggling one class off and confirming those boxes disappear from the feed, then toggling back on.

**Acceptance Scenarios**:

1. **Given** a live camera feed with multiple detected classes, **When** the instructor opens the class filter panel, **Then** all detection classes are listed with toggle switches, all enabled by default.
2. **Given** "Teacher" bounding boxes are visible, **When** the instructor toggles "Teacher" off, **Then** teacher bounding boxes disappear from the feed within 200ms while all other boxes remain visible.
3. **Given** multiple classes toggled off, **When** the instructor clicks "Show All", **Then** all bounding box classes are re-enabled simultaneously.
4. **Given** the instructor has customized class visibility, **When** they switch camera feeds, **Then** the filter settings persist for the current session across all feeds.

---

### User Story 4 - Model Predictions & Anomaly Display (Priority: P2)

The instructor can view a predictions panel showing the output of each detection layer in the YOLO pyramid. This panel is available in two modes: (1) as a collapsible sidebar on the Camera Feed page for at-a-glance monitoring alongside live feeds, and (2) as a dedicated full-screen Predictions page for detailed analysis. For each tracked student, the panel displays the current predictions: posture (standing/sitting), gaze direction (left/right, forward/backward, up/down), and logical constraint status. Most critically, the Anomaly Behavior Analyzer section highlights students flagged as potentially cheating, with severity indicators and behavioral descriptions.

**Why this priority**: The anomaly detection output is the actionable intelligence that makes this system valuable beyond simple monitoring. Instructors need to see which students are flagged and why.

**Independent Test**: Can be tested by connecting a feed with at least 2 detected students, verifying each student's prediction breakdown is displayed, and confirming anomaly alerts appear for students exhibiting unusual behavior.

**Acceptance Scenarios**:

1. **Given** a live feed with tracked students, **When** the instructor opens the Predictions Panel, **Then** each tracked student (by ID) is listed with their current predictions from each pyramid layer (posture, gaze direction).
2. **Given** the Anomaly Behavior Analyzer detects unusual behavior for a student, **When** the alert is generated, **Then** the student entry in the panel is highlighted with a severity indicator (e.g., red border for high severity, yellow for medium) and a brief behavioral description (e.g., "Frequent gaze shifts — potential cheating").
3. **Given** multiple students are being tracked, **When** an anomaly is detected for Student #5, **Then** the instructor can click on the alert to visually highlight Student #5's bounding box on the camera feed.
4. **Given** the predictions panel is open, **When** new predictions arrive from the backend, **Then** the panel updates in real-time without requiring manual refresh.

---

### User Story 5 - Theme Switching (Priority: P3)

The application ships with three themes: Black+Purple (default), Full Black, and White. The instructor can switch between themes at any time from a settings menu or quick-toggle in the header. Theme changes apply immediately across all pages and components without requiring a page reload. The selected theme persists across browser sessions.

**Why this priority**: Theming is a UX polish feature. The core functionality works regardless of theme, but it significantly improves the visual experience and accommodates different lighting conditions in exam halls.

**Independent Test**: Can be tested by logging in, switching to each theme, verifying all components render correctly in each theme, closing the browser, reopening, and confirming the last-selected theme is restored.

**Acceptance Scenarios**:

1. **Given** the application loads for the first time, **When** the instructor views any page, **Then** the Black+Purple theme is applied by default with purple accent colors on a dark background.
2. **Given** the instructor is on any page, **When** they click the theme toggle in the header, **Then** a theme selector appears offering three options: Black+Purple, Full Black, and White.
3. **Given** the instructor selects "White" theme, **When** the selection is made, **Then** all UI components (backgrounds, text, borders, buttons, cards, bounding box overlays) transition to the White theme within 300ms without page reload.
4. **Given** the instructor selected "Full Black" theme and closes the browser, **When** they reopen the application and log in, **Then** the Full Black theme is automatically applied.

---

### User Story 6 - Video Recording & Storage (Priority: P2)

While monitoring live camera feeds, the system automatically records the video streams along with all associated detection predictions, anomaly analyses, and visualization overlays. Recordings are stored and organized by monitoring session (date, time, camera). The instructor can navigate to a Sessions page to browse past monitoring sessions, drill into individual camera recordings within a session, play back recorded feeds with synchronized prediction data and bounding box overlays, and review flagged anomaly events. The canonical UI term is "Session" (not "Recording") — sessions are the primary navigable entity; recordings are children of sessions.

**Why this priority**: Recorded evidence is essential for post-exam review, academic integrity investigations, and providing documentation. Without storage, real-time insights are lost after the exam ends.

**Independent Test**: Can be tested by starting a camera feed session, letting it run for 30 seconds, stopping, navigating to Recordings, and verifying the session appears with playable video and prediction timeline.

**Acceptance Scenarios**:

1. **Given** a live camera feed is active (preview mode, no recording), **When** the instructor clicks "Start Session", **Then** the system begins recording the video feed and all associated prediction data for all currently connected cameras.
2. **Given** a recording is in progress, **When** the instructor navigates to the Sessions page, **Then** the current session appears as "Recording..." with elapsed time, and completed sessions are listed with date, time, duration, camera name, and number of anomaly events.
3. **Given** a completed recording exists, **When** the instructor clicks "Play", **Then** the video plays back with all original bounding boxes, tracking IDs, and predictions synchronized to the video timeline.
4. **Given** a recording with anomaly events, **When** the instructor views the recording detail, **Then** anomaly events are displayed as clickable markers on a timeline that jump playback to the exact moment of the flagged behavior.
5. **Given** an instructor wants to investigate a specific student, **When** they filter the recording by student ID, **Then** only bounding boxes and predictions for that student are shown during playback.

---

### User Story 7 - Multi-Camera Management (Priority: P3)

The instructor can add, remove, and manage multiple camera sources from the Camera Feed page. A camera management panel allows adding cameras by RTSP URL or selecting local video devices. Each camera can be named, and the layout can be customized (e.g., focus on one feed full-screen while others remain as thumbnails). The instructor can also disconnect individual cameras without affecting others.

**Why this priority**: Multi-camera support scales the system for larger exam halls. Single-camera works for MVP, but real exam halls typically have 2-8 cameras.

**Independent Test**: Can be tested by adding two RTSP camera sources, verifying both feeds render simultaneously, removing one, and confirming the other continues uninterrupted.

**Acceptance Scenarios**:

1. **Given** the Camera Feed page, **When** the instructor clicks "Add Camera", **Then** a dialog appears allowing them to enter an RTSP URL or select a local device, and optionally assign a camera name.
2. **Given** 4 cameras are connected, **When** the instructor double-clicks one feed, **Then** that feed expands to full-screen while others collapse to small thumbnails. Double-clicking again restores the grid layout.
3. **Given** multiple cameras are connected, **When** the instructor clicks "Disconnect" on one camera, **Then** only that feed is removed from the grid and remaining feeds reorganize to fill the space. Other cameras continue uninterrupted.
4. **Given** the instructor has configured camera names and layout, **When** they log out and log back in, **Then** camera configurations are restored (names, URLs) but feeds require manual reconnection for security.

---

### User Story 8 - Exam Board & Session Start (Priority: P1)

After logging in, the instructor sees the Exam Board page — a schedule of their assigned exams. Each exam entry shows: subject name, subject type (midterm/final/quiz), subject code, credit hours, number of expected students, and a scrollable list of student names (all data pre-populated by admin). When an exam is upcoming (within the scheduled window), a countdown timer displays the remaining time (maximum 5 minutes). The instructor can click "Start Now" to begin the monitoring session immediately, or wait for the countdown to reach zero. Starting a session transitions the instructor to the Camera Feed page, where the system automatically searches for registered connected cameras and begins the monitoring and recording flow.

**Why this priority**: The Exam Board is the instructor’s entry point to every monitoring session. It provides critical exam context (which students, which subject) and controls session timing — without it, instructors have no structured way to begin monitoring.

**Independent Test**: Can be tested by logging in as an instructor with a pre-configured exam, verifying exam details display correctly, clicking "Start Now", and confirming the Camera Feed page opens with cameras detected.

**Acceptance Scenarios**:

1. **Given** an instructor with scheduled exams, **When** the Exam Board loads, **Then** each exam displays subject name, type, code, credit hours, expected student count, and a scrollable student name list.
2. **Given** an exam is upcoming but more than 5 minutes away, **When** the instructor views the exam entry, **Then** the remaining time is displayed (e.g., "Exam starts in 25 minutes") without a prominent countdown.
3. **Given** the instructor clicks "Start Now", **When** the session begins, **Then** the system transitions to the Camera Feed page, auto-detects registered connected cameras, and begins the live monitoring session with recording.
4. **Given** the exam is within 5 minutes of its scheduled start time, **When** the instructor views the exam entry, **Then** a prominent countdown timer displays the remaining minutes/seconds with visual emphasis.
5. **Given** the countdown timer reaches zero, **When** the exam start time is reached, **Then** the session auto-starts — the system transitions to the Camera Feed page, auto-detects cameras, and begins recording without requiring instructor confirmation.
6. **Given** an instructor joins after the exam start time has passed, **When** the Exam Board loads, **Then** the exam entry shows "Exam started [X] minutes ago" with a prominent "Start Now" button to manually create the session.
7. **Given** an instructor has no scheduled exams, **When** the Exam Board loads, **Then** the page shows "No exams scheduled" with guidance to contact the administrator.
8. **Given** an exam has an active monitoring session, **When** the instructor views the Exam Board, **Then** the exam entry shows an "In Progress" badge with elapsed time and a "Return to Session" link. No "End Session" control is shown on the Exam Board — sessions are ended only from the Camera Feed page.
9. **Given** a session is started from an exam, **When** the exam's scheduled end time is reached, **Then** the system displays a persistent prompt: "Exam time has ended — End Session?" with a visible 5-minute countdown and an "End Now" button. If no action is taken within the 5-minute grace period, the session auto-ends gracefully (all data saved, recordings closed, notification: "Session ended — exam time elapsed").

---

### User Story 9 - Admin Dashboard & Live Session Monitoring (Priority: P2)

After logging in as admin, the Admin Dashboard provides a system-wide overview: active live sessions (which instructors are currently monitoring, which cameras, session duration), total registered instructors, total exams created, storage usage, and recent system activity. The admin can click into any active live session to observe the instructor’s camera feed in read-only mode — seeing detection overlays, predictions, anomaly alerts, and instructor comments in real-time without interfering with the instructor’s monitoring.

**Why this priority**: The admin needs oversight of all ongoing exam monitoring to ensure instructors are performing their duties and the system is functioning properly. Live session monitoring enables real-time quality assurance.

**Independent Test**: Can be tested by having one instructor run a live session, logging in as admin, verifying the dashboard shows the session, and clicking into it to observe the feed in read-only mode.

**Acceptance Scenarios**:

1. **Given** an admin on the Admin Dashboard, **When** one or more instructors have active sessions, **Then** the dashboard displays each session with instructor name, exam name, camera count, session duration, and anomaly count.
2. **Given** a live session is active, **When** the admin clicks "View Session", **Then** the admin sees the instructor’s camera feed(s) in read-only mode with bounding boxes, predictions, anomaly alerts, and instructor comments — all updated in real-time via WebSocket. The admin can also add their own timestamped comments, attributed to their admin identity.
3. **Given** the admin is viewing a live session, **When** the admin attempts to triage an anomaly, **Then** the action is permitted (admins have full triage capabilities in any session).
4. **Given** no active sessions exist, **When** the admin views the dashboard, **Then** the Live Sessions section shows "No active sessions" with the last session's end time.
5. **Given** a high or medium severity anomaly is detected in any active session, **When** the admin is on the Admin Dashboard, **Then** the unified anomaly feed updates in real-time showing: severity, student ID, exam name, instructor name, timestamp, and a "View Session" link that jumps directly into the live session.
---

### User Story 10 - Admin User & Role Management (Priority: P2)

The admin navigates to a User Management page where they can create new instructor accounts, edit existing accounts, and deactivate accounts. The admin can also create custom roles with configurable access ranges — defining which pages each role can access and which actions it can perform. Roles are then assigned to instructors to control their scope of access. The admin also manages exam data: creating exams with all metadata and entering student rosters (student names and university IDs).

**Why this priority**: Account and role management is essential for onboarding instructors and controlling system access. Exam data entry is required before instructors can use the Exam Board. Without admin management, the system cannot scale to multiple instructors or exams.

**Independent Test**: Can be tested by logging in as admin, creating a new instructor account with a role, creating an exam with a student roster, logging out, and verifying the new instructor can log in and see the exam on their Exam Board.

**Acceptance Scenarios**:

1. **Given** the admin navigates to User Management, **When** they click "Create Instructor", **Then** a form appears for: username, email, first name, last name, initial password, and role assignment.
2. **Given** the admin creates a role called "Proctor" with access only to Camera Feed and anomaly triage, **When** an instructor with the "Proctor" role logs in, **Then** they can only see the Exam Board and Camera Feed pages — other pages (Recordings, Health, Settings) show "Access Denied" or are hidden from navigation.
3. **Given** the admin navigates to Exam Management, **When** they create a new exam, **Then** the form accepts: subject name, subject type, subject code, credit hours, expected student count, exam date/time, duration, assigned instructor(s), and a student roster (name and university ID per student).
4. **Given** the admin deactivates an instructor account, **When** the deactivated instructor attempts to log in, **Then** the login is rejected with a non-specific error message and any active session for that instructor is gracefully terminated.
5. **Given** the admin edits an instructor’s role, **When** the change is saved, **Then** the instructor’s access range updates on their next page navigation (no active session interruption).

---

### User Story 11 - Admin Recording Review & Action Revert (Priority: P2)

The admin can browse all past monitoring sessions from all instructors — with filters by instructor name, exam, date range, and anomaly count. For each session, the admin can review: video playback with all detection overlays, instructor comments with timestamps, model predictions and outputs from the anomaly detection pipeline, and the full triage history of each anomaly event. Critically, the admin can revert any triage action taken by an instructor — resetting an acknowledged or dismissed anomaly back to `new` status with a mandatory reason — ensuring administrative oversight and accountability.

**Why this priority**: Administrative review and action revert are essential for accountability in academic integrity investigations. Without admin oversight, instructor triage decisions are final and potentially irreversible, which could undermine the credibility of the monitoring system.

**Independent Test**: Can be tested by having an instructor complete a session with triaged anomalies, logging in as admin, navigating to Recording Review, finding the session, reviewing the recording with overlays and comments, and reverting a dismissed anomaly back to `new` status.

**Acceptance Scenarios**:

1. **Given** the admin navigates to Recording Review, **When** the page loads, **Then** all past sessions from all instructors are listed with: instructor name, exam name, date/time, duration, camera count, anomaly count, and triage summary.
2. **Given** the admin selects a completed session (or exam with multiple instructor sessions), **When** the review page opens, **Then** video playback is available with synchronized bounding boxes, tracking IDs, model predictions, anomaly markers on the timeline, and all instructor comments displayed at their timestamps — each attributed to the instructor who wrote it.
3. **Given** a dismissed anomaly in a reviewed session, **When** the admin clicks "Revert" on the anomaly, **Then** a dialog requires a mandatory reason (minimum 5 characters), and upon confirmation, the anomaly status resets to `new`. The revert is logged in the audit trail.
4. **Given** the admin reverts an anomaly triage action, **When** any instructor is currently viewing the same session, **Then** they receive a real-time WebSocket notification: "Anomaly status reverted to 'New' by Admin [name] — Reason: [reason]".
5. **Given** the admin is reviewing a session, **When** they view the model outputs section, **Then** they can see all pyramid detection predictions, anomaly detection analysis, and rule engine outputs for each flagged student.

---

### Edge Cases

- What happens when an RTSP camera feed provides corrupted or malformed frames? The system MUST skip corrupted frames gracefully, display the last valid frame, and log the corruption event without crashing the feed.
- What happens when the detection backend becomes unavailable mid-session? The camera feed MUST continue displaying raw video without bounding boxes, and a status banner MUST indicate "Detection Service Unavailable — Raw Feed Only."
- What happens when a student leaves and re-enters the camera frame? The tracking system MUST attempt to re-identify the student using track history within a configurable window (default: 60 seconds, adjustable via pipeline config). If re-identification confidence is below the configured threshold or the window has elapsed, a new ID is assigned and the instructor is notified of a possible ID change.
- What happens when the network bandwidth is insufficient for multiple simultaneous high-resolution feeds? The system MUST degrade gracefully by reducing frame rates before dropping feeds, and display a bandwidth warning to the instructor.
- What happens if the instructor opens the application in two browser tabs simultaneously? The system MUST support multiple tabs with independent camera configurations, but recordings MUST NOT duplicate for the same camera feed. This deduplication extends to multiple instructors — a shared camera produces one recording regardless of how many instructors or sessions reference it.
- What happens when storage capacity for recordings is reaching its limit? The system MUST warn the instructor at 80% capacity and 95% capacity, and provide an option to download or delete old recordings.
- What happens when no students are detected in a frame? The feed MUST continue rendering normally with no bounding boxes; the predictions panel MUST show "No students detected in frame."
- What happens when the instructor's browser crashes during an active recording? The recording MUST continue on the backend/server side. When the instructor reconnects, the session MUST resume from the current state with recording intact.
- What happens when two instructors try to triage the same anomaly alert simultaneously? The system uses first-write-wins: the first instructor's action (acknowledge/dismiss) locks the status. The second instructor's attempt MUST be rejected with an inline message showing the current status and which instructor handled it. All connected viewers receive a real-time WebSocket notification of the status change.
- What happens when an admin reverts a triage action while an instructor currently has that anomaly's details open? The instructor MUST receive a real-time WebSocket notification and the UI MUST refresh the anomaly status to `new` automatically.
- What happens when an instructor has no scheduled exams on the Exam Board? The page MUST show "No exams scheduled" with guidance to contact the administrator.
- What happens when the session countdown timer reaches zero but the instructor hasn't clicked "Start Now"? The session auto-starts — the system transitions to the Camera Feed page, auto-detects cameras, and begins recording automatically.
- What happens when a session auto-ends due to exam time expiring? When the exam's scheduled end time is reached, the system MUST display a persistent prompt ("Exam time has ended — End Session?") with a 5-minute grace countdown. The instructor can click "End Now" or continue working. If no action is taken within 5 minutes, the session MUST auto-end gracefully: save all data, close recordings, and show a notification: "Session ended — exam time elapsed."
- What happens when an admin deactivates an instructor who has an active monitoring session? The session MUST be gracefully terminated with all data saved — not abruptly killed.
- What happens when a role's access range is changed while an instructor with that role is logged in? Navigation MUST update on the next page transition; any page the instructor is currently viewing MUST NOT be forcibly closed.
- What happens when an instructor tries to start a session from the Exam Board but no registered cameras are found for the room? The system MUST display a clear message: "No cameras registered for this room" with options to manually enter a camera URL or contact the admin.
- What happens when multiple instructors are assigned to the same exam and both are monitoring simultaneously? Each instructor runs their own independent session. Each can add their own notes, triage anomalies, and configure cameras independently. The first-write-wins rule still applies for anomaly triage across all instructors. After the exam ends, all sessions are merged into a unified history view with full attribution.

## Requirements *(mandatory)*

### Functional Requirements

**Authentication & Session Management**

- **FR-001**: System MUST provide a login page where users (instructors and admins) authenticate with email or username and password. The login form MUST accept either email or username as the identifier and resolve to the correct account.
- **FR-002**: System MUST validate credentials and issue a secure session token upon successful authentication.
- **FR-003**: System MUST redirect authenticated users away from the login page to their role-appropriate landing page — Exam Board for instructors, Admin Dashboard for admins.
- **FR-004**: System MUST invalidate sessions on logout and after a configurable inactivity timeout (default: 30 minutes). An active WebSocket connection (e.g., detection feed, anomaly channel, health updates) counts as activity and continuously resets the timeout. The timeout only triggers when no HTTP requests AND no active WebSocket connections exist for the user's session.
- **FR-005**: System MUST display non-specific error messages for failed login attempts (no field-level hints). No brute force protection (rate limiting, lockout, or CAPTCHA) is required — this is an internal graduation project.

**Main Dashboard**

- **FR-046**: System MUST provide role-based landing pages after login. Instructors land on the Exam Board page displaying their scheduled exams with subject details, student roster, and session start controls. Admins land on the Admin Dashboard displaying: count of active live sessions with instructor names, total registered instructors, total exams, storage usage, a unified real-time anomaly feed (high/medium severity from all active sessions with jump-to-session links), recent system activity, and navigation to User Management, Exam Management, Room Management, and Recording Review pages.

**Camera Feed Management**

- **FR-006**: System MUST allow instructors to connect cameras via RTSP URL or local device selection. Connecting a camera shows a live preview feed without starting a recording. Recording only begins when the instructor explicitly starts a monitoring session (two-step flow: connect → preview → start session → record).
- **FR-007**: System MUST support simultaneous display of 1 to 8 camera feeds in a responsive grid layout. Grid auto-adjusts by feed count: 1 feed = full-width, 2 = 1×2 row, 3–4 = 2×2 grid, 5–6 = 2×3 grid, 7–8 = 2×4 grid. At viewport widths below 1200px, columns reduce by one tier.
- **FR-008**: System MUST render live video feeds with less than 1 second of display latency from the camera source. The go2rtc WHEP endpoint MUST be exposed at `/whep/{camera_id}/` — a top-level reverse-proxy path (not under `/api/v1/`). Nginx proxies requests to go2rtc's internal `http://go2rtc:1984/api/webrtc?src={camera_id}` after validating the frontend's session cookie via a Django auth subrequest. No separate API key or token is required.
- **FR-009**: System MUST allow instructors to add, name, disconnect, and remove individual cameras. Cameras are system-wide shared resources — any instructor can view any connected camera. Disconnecting a camera that other instructors are viewing MUST require confirmation.
- **FR-010**: System MUST attempt automatic reconnection with exponential backoff (1s → 2s → 4s → 8s → 16s max, up to 5 retries) when a camera connection drops. After all retries are exhausted, the feed tile MUST display a persistent "Connection Lost" status with a manual "Retry" button.

**Detection Visualization**

- **FR-011**: System MUST render bounding boxes on the video feed for every detected entity, color-coded by detection class.
- **FR-012**: System MUST display a persistent tracking ID label on each tracked student's bounding box.
- **FR-013**: System MUST provide a filter panel allowing instructors to show or hide bounding boxes per detection class independently.
- **FR-014**: System MUST distinguish teachers from students visually (distinct bounding box colors) and display teacher boxes for visualization only (no tracking data).
- **FR-015**: Filter changes MUST take effect within 200ms on the currently displayed feed.

**Predictions & Anomaly Display**

- **FR-016**: System MUST display a predictions panel showing per-student output from each pyramid detection layer (posture, gaze direction). The panel MUST be accessible both as a collapsible sidebar within the Camera Feed page and as a dedicated full-screen Predictions page.
- **FR-017**: System MUST display anomaly alerts with severity level (high/medium/low), student ID, timestamp, and behavioral description. Anomaly alerts are triggered by a configurable rule engine that supports two mechanisms: (1) **duration-based thresholds** — sustained suspicious prediction over a configurable time window (e.g., "looking left" >10 seconds = medium, >30 seconds = high), and (2) **multi-layer combinations** — specific simultaneous predictions across pyramid layers (e.g., "looking left" + "turned posture" = high). Rules are defined in a config file, not hardcoded.
- **FR-018**: System MUST allow the instructor to click an anomaly alert to highlight the corresponding student on the camera feed.
- **FR-019**: Predictions panel MUST update in real-time via a persistent WebSocket connection. The backend MUST push detection results and anomaly alerts to the frontend over this channel. The frontend MUST NOT rely on polling. WebSocket connections MUST be authenticated via session cookie passthrough — Django Channels validates the existing HTTP session cookie during the WebSocket upgrade handshake; no separate token or auth message is required. The WebSocket connection MUST implement automatic reconnection with exponential backoff (1s → 2s → 4s → 8s → 16s max, up to 10 retries) and re-subscribe to all active channels upon successful reconnect. After 10 failed retries, the system MUST display a persistent "Connection Lost" banner.
- **FR-020**: System MUST enforce logical constraint display — mutually exclusive predictions (e.g., forward + backward) MUST NOT both appear as active for the same student.

**Theme System**

- **FR-021**: System MUST ship with three themes: Black+Purple (default), Full Black, and White.
- **FR-022**: Theme selection MUST apply immediately to all UI components without page reload.
- **FR-023**: Selected theme MUST persist across browser sessions via local storage.
- **FR-024**: All three themes MUST maintain sufficient color contrast for readability (WCAG 2.1 AA minimum).
- **FR-025**: Bounding box overlay colors MUST remain distinguishable across all three themes.

**Video Recording & Storage**

- **FR-026**: System MUST automatically record live camera feeds as raw video (without baked-in overlays) and store all associated prediction data separately (in database or structured files) when a monitoring session is active.
- **FR-027**: System MUST store recordings organized by session (date, time, camera identifier) and make them browsable on a Recordings page.
- **FR-028**: System MUST allow playback of recorded sessions with bounding boxes, tracking IDs, and prediction data rendered dynamically as overlays from the stored metadata — enabling overlay toggling and student filtering during playback.
- **FR-029**: System MUST display anomaly events as clickable markers on the playback timeline that jump to the corresponding moment.
- **FR-030**: System MUST allow filtering playback by student ID to isolate a specific student's bounding boxes and predictions.
- **FR-031**: System MUST warn instructors when storage reaches 80% and 95% capacity. Admins can manually delete completed recordings via `DELETE /api/v1/admin/recordings/{id}/`; instructors can only download recordings. Active recordings cannot be deleted. Automatic retention cleanup (configurable, default 30 days) operates independently.
- **FR-032**: Recordings MUST continue on the server if the instructor's browser disconnects mid-session.

**UI/UX Standards**

- **FR-033**: System MUST use a consistent component library (buttons, cards, modals, status badges, form inputs) across all pages.
- **FR-034**: System MUST provide loading states for all asynchronous operations (camera connecting, data loading, recording processing).
- **FR-035**: System MUST be fully keyboard-navigable for all primary workflows.
- **FR-036**: System MUST display actionable error messages (not raw error codes or stack traces) for all user-facing errors. The frontend MUST implement React error boundaries at the page level to catch render errors and display a user-friendly fallback UI. No external error tracking service is required — structured `console.error` logging with component context is sufficient for debugging.
- **FR-037**: System MUST support a minimum display resolution of 1920x1080 with responsive layout.

**Observability & Health Monitoring**

- **FR-038**: System MUST provide a dedicated Health Dashboard page accessible to both instructors and admins showing: backend API status (up/down), WebSocket connection health, detection pipeline FPS per camera, storage usage (used/total with percentage), and count of active monitoring sessions.
- **FR-039**: System MUST expose a `/health` API endpoint returning JSON status of all subsystems (API, database, Redis, WebSocket layer, detection pipeline, storage). The endpoint MUST return `200 OK` when all subsystems are `healthy`, and `503 Service Unavailable` when any critical subsystem is `unhealthy`. The JSON response body MUST include a per-subsystem status field with values `healthy`, `degraded`, or `unhealthy`, plus an overall `status` field. Health thresholds: database connection failure = `unhealthy`; Redis unreachable = `unhealthy`; detection pipeline <5 FPS on any active camera = `degraded`; storage usage >95% = `degraded`; storage usage >99% = `unhealthy`; WebSocket channel layer unreachable = `unhealthy`. The endpoint MUST be unauthenticated (for external monitoring tool integration) but MUST NOT expose sensitive configuration details.
- **FR-040**: Health Dashboard MUST auto-refresh at ≤10-second intervals without manual reload.

**Anomaly Triage Actions**

- **FR-041**: System MUST allow instructors to **acknowledge** an anomaly alert, moving it from `new` to `acknowledged` status. Acknowledged alerts MUST be visually de-emphasized but remain visible and tracked. If another instructor has already triaged the alert, the system MUST reject the action and display the current status with the acting instructor's identity (first-write-wins).
- **FR-042**: System MUST allow instructors to **dismiss** an anomaly alert with a **mandatory reason** (free-text, minimum 5 characters, no maximum length). Dismissed alerts move to `dismissed` status, are hidden from the active alert list, but remain retrievable in alert history. If another instructor has already triaged the alert, the system MUST reject the dismissal and display the current status (first-write-wins).
- **FR-043**: System MUST allow instructors to **add timestamped free-text notes** to any anomaly alert regardless of its current status. Notes MUST record the instructor's identity and timestamp. No maximum length constraint on notes.
- **FR-044**: System MUST display anomaly alert status (new/acknowledged/dismissed) with distinct visual indicators across all three themes.
- **FR-045**: System MUST provide an alert history view where instructors can see all anomaly events including dismissed ones, with their status, reasons, and notes.

**Data Export**

- **FR-047**: System MUST allow instructors to export a full session archive containing: raw video recording, all detection/prediction data, anomaly events with triage status and notes, and student tracking IDs — all non-anonymized for complete investigative traceability. Export is asynchronous: initiating an export creates a SessionExport that transitions through `pending` → `processing` (with progress percentage) → `completed` (downloadable) or `failed` (with error message and retry option). The frontend MUST poll the export status endpoint and display progress to the instructor.
- **FR-048**: Session exports MUST be downloadable as a structured archive (e.g., ZIP) containing the video file(s) and a structured data file (JSON or CSV) with all associated predictions, anomaly events, and triage actions.

**System Availability**

- **FR-049**: All backend services MUST be managed by a process supervisor that automatically restarts any crashed service within 30 seconds. In-progress recordings and persisted data MUST NOT be corrupted by a service crash or restart.

**Admin Authentication & Role-Based Access**

- **FR-050**: System MUST support two built-in user roles from initial release: Admin and Instructor. Admins have elevated permissions including user management, role management, exam management, and triage action revert.
- **FR-051**: System MUST enforce role-based access control (RBAC) where custom roles define page-level permissions (which pages/sections are accessible) and action-level permissions (e.g., can_triage_anomalies, can_manage_cameras, can_export_sessions). An instructor MUST NOT access pages or perform actions outside their assigned role's access range.
- **FR-052**: Admin-only pages (User Management, Role Management, Exam Management, Recording Review with revert) MUST NOT be accessible to instructor-role users. Unauthorized access attempts MUST return a clear "Access Denied" response.
- **FR-053**: Role changes MUST take effect on the instructor's next page navigation. Active sessions MUST NOT be interrupted by role changes.

**Exam Board & Scheduling**

- **FR-054**: System MUST provide an Exam Board page as the instructor's post-login landing page, displaying all assigned exams with: subject name, subject type (midterm/final/quiz), subject code, credit hours, expected student count, and a scrollable student name list.
- **FR-055**: The Exam Board MUST display the actual remaining time for all upcoming exams (e.g., "Exam starts in 25 minutes"). When an exam is within 5 minutes of its scheduled start time, the timer MUST transition to a prominent countdown display with visual emphasis. When the countdown reaches zero, the system MUST auto-start the monitoring session (transition to Camera Feed, auto-detect cameras, begin recording) without requiring instructor confirmation. If the instructor joins after the exam start time, a "Start Now" button MUST be displayed to manually create the session.
- **FR-056**: When a monitoring session starts from the Exam Board (countdown auto-start or "Start Now"), the system MUST automatically load all cameras registered to the exam’s assigned room/hall, transition to the Camera Feed page with those cameras pre-loaded, and begin recording. If the room has no registered cameras, the system MUST display "No cameras registered for this room" with an option to manually enter a camera URL.
- **FR-057**: Exam data (subject details and student rosters) MUST be entered by admins through admin management pages. Instructors view exam data in read-only mode on the Exam Board.

**Instructor Session Comments**

- **FR-058**: System MUST allow instructors and admins to add timestamped free-text comments during an active monitoring session. Comments are general observations not necessarily tied to a specific anomaly event. Each comment MUST record the user’s identity (with role attribution, e.g., "Admin: Dr. Smith" or "Instructor: Prof. Jones"), timestamp, and associated camera source. Admins can add comments in any session they are observing.
- **FR-059**: Comments MUST be visible during session recording playback, displayed as clickable markers on the playback timeline alongside anomaly markers.

**Admin User & Role Management**

- **FR-060**: System MUST allow admins to create, edit, deactivate, and reset passwords for user accounts. Account creation MUST require: username, email, first name, last name, initial password, and role assignment. When an admin resets a user's password, the system MUST set the `must_change_password` flag to `true`. On the user's next login, middleware MUST redirect them to a password-change page. All other pages MUST redirect to the password-change form until the user successfully changes their password (which clears the flag).
- **FR-061**: System MUST allow admins to create custom roles with configurable access ranges. Access ranges MUST define which pages are accessible (e.g., Camera Feed, Recordings, Health Dashboard) and which actions are permitted (e.g., triage anomalies, export sessions, manage cameras).
- **FR-062**: System MUST allow admins to assign and change roles for any instructor. A history of role changes MUST be recorded in the audit log.
- **FR-063**: System MUST allow admins to create and manage exams: subject name, subject type, subject code, credit hours, expected student count, exam date/time, duration, assigned room/hall, assigned instructor(s), and student roster (name and university ID per student). System MUST also allow admins to manage rooms/halls: create rooms, assign cameras to rooms, edit room details.

**Admin Recording Review & Action Revert**

- **FR-064**: System MUST allow admins to browse all past monitoring sessions from all instructors, with filters by instructor name, exam, date range, and anomaly count.
- **FR-065**: Admin recording review MUST display: video playback with all overlays, instructor comments with timestamps, model predictions and anomaly detection outputs, and full triage history for each anomaly.
- **FR-066**: System MUST allow admins to revert any triage action (acknowledge/dismiss) on an anomaly, resetting its status to `new`. Revert MUST require a mandatory reason (minimum 5 characters). The revert action MUST be recorded in the audit log with admin identity, timestamp, and reason.
- **FR-067**: When an admin reverts a triage action, all connected viewers of that session MUST receive a real-time WebSocket notification of the status change, identical to the first-write-wins broadcast mechanism.

**Anomaly Behavior Details**

- **FR-068**: Anomaly alerts MUST include the start and end timestamps of the detected unusual behavior pattern, displayed in human-readable format (e.g., "Started 2m 35s ago — Ongoing" or "12:04:15 – 12:04:47"). If the behavior is ongoing, the end time MUST show "Ongoing" and update in real-time.

**Audit Logging**

- **FR-069**: System MUST maintain an append-only audit log recording all state-changing actions: login, logout, failed login attempts, account create/edit/deactivate, role create/edit/assign, exam create/edit, monitoring session start/end, anomaly triage (acknowledge/dismiss), and admin triage reverts. Each entry MUST include: action type, performing user, target entity, contextual details (e.g., revert reason, old/new role), IP address, and timestamp. Audit log entries MUST be immutable (no updates or deletes).

**Password Management**

- **FR-070**: System MUST allow authenticated users to change their own password from a profile/settings page. Password change MUST require the current password for verification. There is no self-service "Forgot Password" or email-based reset flow. Admin-initiated password resets are handled via FR-060.

**Admin Real-Time Anomaly Feed**

- **FR-071**: The Admin Dashboard MUST display a unified real-time anomaly feed showing all high and medium severity anomalies from all active monitoring sessions. Each feed entry MUST include: severity level, student tracking ID, exam/session name, instructor name, behavioral description, timestamp, and a clickable link to jump directly into the live session view. The feed MUST update in real-time via WebSocket.

**Exam Board Active Session Display**

- **FR-072**: When an exam has an active monitoring session, the Exam Board MUST display the exam entry with an "In Progress" status badge, elapsed session time (updating in real-time), and a "Return to Session" link that navigates to the Camera Feed page. No "End Session" control MUST appear on the Exam Board — sessions are ended exclusively from the Camera Feed page.

**Admin Activity Feed**

- **FR-073**: The Admin Dashboard MUST display a "Recent Activity" section backed by the Audit Log (FR-069). The feed shows Audit Log entries with: action type (human-readable label), performing user name, target entity summary, and timestamp. The user MUST be able to select the number of displayed entries via a page size selector offering: 10, 20, 30, 40, 50, 60, 70, 80, 90, 100, or All (default: 50). The feed MUST be filterable by action type. When "All" is selected, the system MUST paginate with lazy loading to avoid performance degradation.

### Key Entities

- **User**: The single account entity for both instructors and admins — differentiated by assigned Role. There is no separate Admin table; an admin is simply a User with the built-in "Admin" role. The first admin is bootstrapped via a CLI management command (e.g., `createsuperuser`). Attributes: username, email, first name, last name, hashed password, role reference, created date, last login, theme preference, is_active (for deactivation by admin).
- **Camera Source**: A system-wide camera resource providing a video feed. Cameras are registered to a specific room/hall (for exam-based auto-detection) and shared — any instructor can view any connected camera, and a single recording is produced per camera (not per instructor). Attributes: name (user-assigned), connection URL (RTSP) or device identifier, room reference, status (connected/disconnected/reconnecting/error), resolution, frame rate, added-by user reference.
- **Monitoring Session**: A time-bounded period during which one or more cameras are actively recording. Each instructor runs their own session even when multiple instructors are assigned to the same exam — sessions are per-instructor, not per-exam. After the exam ends, all sessions for the same exam are merged into a unified history view with full instructor attribution. Attributes: start time, end time, status (active/completed/interrupted), associated cameras, instructor, exam reference. Status transitions: `active` → `completed` (instructor ends session normally) or `active` → `interrupted` (any abnormal termination: server crash beyond auto-recovery, unrecoverable network loss, or manual abort by instructor). Browser disconnect alone does NOT interrupt — recording continues server-side per FR-032.
- **Detection Frame**: A single processed video frame with detection results. At ~10–15 fps per camera, this entity represents high-volume data (~80–120 rows/second across 8 cameras) requiring time-based partitioning and retention policies. Attributes: timestamp, source camera, raw frame reference, list of detections.
- **Detection**: A single entity detected in a frame. Attributes: class (teacher/student), bounding box coordinates, confidence score, tracking ID (students only).
- **Pyramid Prediction**: The full set of predictions for a tracked student from all pyramid layers. Attributes: student tracking ID, posture (standing/sitting), horizontal gaze (left/right), depth gaze (forward/backward), vertical gaze (up/down), logical constraint status, timestamp.
- **Anomaly Event**: A flagged behavioral anomaly for a specific student, triggered by the configurable rule engine (duration thresholds and/or multi-layer prediction combinations). Anomaly events are per-camera, not per-session — when multiple instructors monitor the same camera, they see and can triage the same anomaly events. The associated recording is derived via `camera_source_id` + timestamp range (no direct recording FK). Attributes: student tracking ID, severity (high/medium/low), behavioral description, triggered rule reference (which config rule fired), timestamp, source camera, associated pyramid predictions, status (new/acknowledged/dismissed), dismissed reason (mandatory when dismissed), status changed by (instructor), status changed at (timestamp). Notes are stored as separate AnomalyNote entities (see below).
- **Anomaly Note**: A timestamped annotation attached to a specific Anomaly Event, authored by an instructor or admin. Multiple notes can exist per anomaly. Attributes: anomaly event reference, author (user reference with role attribution), note text (free-text, no maximum length), timestamp. Notes are append-only — once created, they cannot be edited or deleted. Distinct from Instructor Comments (FR-058) which are general session observations not tied to a specific anomaly.
- **Recording**: A stored video file with synchronized metadata. Recordings are per-camera, not per-instructor — when multiple instructors view the same camera, only one recording is produced (deduplicated). Multiple instructor sessions may reference the same recording. Attributes: camera reference (primary key alongside time range), raw video file path (no baked overlays), duration, file size, separate prediction data store reference (structured metadata for dynamic overlay rendering), list of anomaly event timestamps.
- **SessionExport**: An asynchronous job to produce a downloadable session archive (referred to as "export job" in user-facing contexts). Attributes: session reference, status (`pending` / `processing` / `completed` / `failed`), progress percentage (0–100), initiated by (instructor reference), created at, completed at, file path (when completed), file size (when completed), error message (when failed).
- **Role**: A named permission set defining access scope for instructors. Attributes: name, description, list of permitted pages (page-level access), list of permitted actions (action-level access), created by (admin reference), created at, updated at. Two built-in roles (Admin, Instructor) are non-deletable; custom roles can be created by admins.
- **Exam**: A scheduled exam event with metadata, entered by admins. Attributes: subject name, subject type (midterm/final/quiz), subject code, credit hours, expected student count, assigned instructor(s), assigned room/hall reference, exam date/time, duration, status (scheduled/active/completed), created by (admin reference), created at, updated at.
- **Exam Student**: A student expected in a specific exam. Attributes: exam reference, student name, student university ID, seat number (optional). Student rosters are entered by admins per exam.
- **Room/Hall**: A physical exam location with registered cameras. Attributes: name (e.g., "Hall A", "Room 301"), building (optional), capacity (optional), list of registered camera references, created by (admin reference), created at, updated at. Rooms are managed by admins.
- **Instructor Comment**: A timestamped general observation during a monitoring session, not tied to a specific anomaly. Attributes: session reference, camera source reference (optional), instructor identity, comment text, timestamp. Distinct from anomaly notes (FR-043) which are attached to a specific anomaly event.
- **Audit Log Entry**: An append-only record of every state-changing action in the system. Attributes: action_type (enum: `login`, `logout`, `login_failed`, `account_create`, `account_edit`, `account_deactivate`, `role_create`, `role_edit`, `role_assign`, `exam_create`, `exam_edit`, `session_start`, `session_end`, `triage_acknowledge`, `triage_dismiss`, `triage_revert`, `recording_access`, `recording_download`, `config_change`, `anomaly_annotate`, `camera_add`, `camera_remove`, `export_session`), performing user reference, target entity type, target entity ID, details (JSON — stores contextual data such as revert reason, old/new role, etc.), IP address, timestamp. Entries are immutable — no updates or deletes.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Instructors can log in and reach the camera feed page in under 5 seconds from entering credentials.
- **SC-002**: Live camera feeds display with less than 1 second of latency from the physical camera to the on-screen rendered frame.
- **SC-003**: Bounding box class filter toggles take effect within 200ms of user interaction.
- **SC-004**: The system supports at least 4 simultaneous camera feeds for a single instructor without visible frame drops below 15 FPS.
- **SC-005**: Theme switching applies to all visible components within 300ms without page reload.
- **SC-006**: 90% of instructors can successfully connect a camera and view a live feed with detections on their first attempt without external guidance. *(Post-implementation UX validation goal — verified via usability testing, not automated test suite.)*
- **SC-007**: Anomaly alerts appear in the predictions panel within 2 seconds of the anomaly being detected by the backend.
- **SC-008**: Recorded sessions can be played back with synchronized bounding boxes and prediction data, with less than 500ms seek time when jumping to anomaly timeline markers.
- **SC-009**: The system operates without degradation on both Windows and Linux environments. Crashed services auto-recover within 30 seconds via process supervisor.
- **SC-010**: All three themes (Black+Purple, Full Black, White) pass WCAG 2.1 AA contrast requirements for text and interactive elements.
- **SC-011**: Admins can create a new instructor account and assign a role in under 60 seconds.
- **SC-012**: The Exam Board page loads with all exam details and student rosters within 3 seconds of instructor login.
- **SC-013**: Reverting a triage action takes effect and notifies all connected viewers within 2 seconds.
- **SC-014**: Unauthorized page access attempts are blocked and return an "Access Denied" message within 500ms.
