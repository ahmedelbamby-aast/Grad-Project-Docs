# Implementation Gate Checklist: Exam Monitoring Dashboard

**Purpose**: Deep implementation-readiness gate — validate that all spec artifacts have sufficient detail, cross-document consistency, and completeness for developers to begin coding without ambiguity
**Created**: 2026-02-27
**Feature**: [spec.md](../spec.md) | [plan.md](../plan.md) | [tasks.md](../tasks.md) | [tasks-frontend.md](../tasks-frontend.md) | [data-model.md](../data-model.md) | [contracts/rest-api.md](../contracts/rest-api.md) | [contracts/websocket-api.md](../contracts/websocket-api.md)
**Depth**: Deep | **Audience**: Implementation gate (pre-coding)

---

## Cross-Document Traceability — Spec ↔ Tasks

- [ ] CHK001 — Does every FR (FR-001 through FR-073) have at least one implementing task in tasks.md or tasks-frontend.md? [Traceability, Spec → tasks.md]
- [ ] CHK002 — Are there tasks in tasks.md that reference no FR or US? If so, is the rationale (infrastructure, tooling) documented? [Traceability, tasks.md]
- [ ] CHK003 — Does FR-R-017 (merged session view) in rest-api.md have a corresponding task? It is referenced only in the contract, not in spec.md FRs. [Traceability, Gap, rest-api.md §Merged Session View]
- [ ] CHK004 — Are frontend tasks (TF-001 through TF-118) traceable to specific FRs or US acceptance scenarios? [Traceability, tasks-frontend.md]
- [ ] CHK005 — Does the tasks.md task T123 (AnomalyEvent and AnomalyNote models) reference the correct FR numbers for anomaly triage (FR-041, FR-042, FR-043)? [Traceability, tasks.md §T123]

## Cross-Document Traceability — Spec ↔ Data Model

- [ ] CHK006 — Does every spec Key Entity (9 listed) map to a data-model entity? Are the 7 additional data-model entities (AnomalyNote, AuditLogEntry, Role, Room, ExamStudent, Recording, SessionExport) justified by FRs? [Traceability, Spec §Key Entities → data-model.md]
- [ ] CHK007 — Are all field-level constraints in data-model.md (e.g., AnomalyEvent description ≤500 chars) traceable to specific FRs or explicitly marked as implementation decisions? [Traceability, data-model.md]
- [ ] CHK008 — Does the data model's camera state machine (connected/disconnected/reconnecting/error) align with spec FR-010's reconnection states? The spec defines 3 states; data-model defines 4 (adds `error`). [Consistency, Spec §FR-010 vs data-model §CameraSource]
- [ ] CHK009 — Are the 16 data-model entities sufficient to implement all 73 FRs, or are derived/aggregate tables needed for dashboard (FR-046), activity feed (FR-073), or anomaly feed (FR-071)? [Completeness, data-model.md → Spec §FR-046, FR-071, FR-073]

## Cross-Document Traceability — Spec ↔ REST API Contract

- [ ] CHK010 — Does every FR that implies a REST endpoint have a matching contract in rest-api.md? Specifically verify: FR-030 (playback student filter), FR-031 (storage warnings), FR-035 (keyboard navigation — no API needed?), FR-053 (role change propagation). [Traceability, Spec → rest-api.md]
- [ ] CHK011 — Does the REST API contract for recording deletion exist? Spec Assumptions and Edge Case #6 reference "manually delete" recordings but no FR or endpoint is defined. [Gap, Spec §Assumptions + Edge Case #6 → rest-api.md]
- [ ] CHK012 — Are all REST API error codes from rest-api.md traceable to specific validation rules in spec FRs? (e.g., DUPLICATE_USERNAME → FR-060, ALREADY_TRIAGED → FR-041) [Traceability, rest-api.md error responses]
- [ ] CHK013 — Does the REST API contract cover the admin camera-to-room assignment flow (POST/DELETE on `/admin/rooms/{id}/cameras/`) referenced by FR-063? [Traceability, Spec §FR-063 → rest-api.md §Admin Room Management]
- [ ] CHK014 — Is the `page_size` parameter for audit log and activity feed (accepting 10–100 and "all") consistently defined between rest-api.md and FR-073? [Consistency, Spec §FR-073 vs rest-api.md §Audit Log]

## Cross-Document Traceability — Spec ↔ WebSocket Contract

- [ ] CHK015 — Does every real-time FR (FR-019, FR-067, FR-071) map to specific WebSocket message types in websocket-api.md? [Traceability, Spec → websocket-api.md]
- [ ] CHK016 — Does the WebSocket contract define message types for all server-push scenarios: detection frames, prediction updates, anomaly alerts, anomaly status changes (including admin revert per FR-067), comment broadcasts, camera status, health updates, and storage warnings? [Completeness, websocket-api.md]
- [ ] CHK017 — Are WebSocket reconnection parameters in websocket-api.md (10 retries, 1s→16s backoff) consistent with spec FR-019 (10 retries, 1s→16s) AND distinguished from camera reconnection FR-010 (5 retries, 1s→16s)? [Consistency, websocket-api.md vs Spec §FR-010, §FR-019]
- [ ] CHK018 — Does the WebSocket contract specify the `admin.anomaly_alert` message type referenced in FR-071 (admin dashboard anomaly feed)? [Completeness, websocket-api.md → Spec §FR-071]

## Data Model Implementation Readiness

- [ ] CHK019 — Are Django model field types specified for all 16 entities (CharField lengths, DecimalField precision, JSONField schemas, ForeignKey on_delete behavior)? [Completeness, data-model.md]
- [ ] CHK020 — Are database indexes defined for all query-heavy access patterns (DetectionFrame by session+timestamp, AnomalyEvent by session+status, AuditLogEntry by timestamp+action_type)? [Completeness, data-model.md]
- [ ] CHK021 — Is the PostgreSQL date-based partitioning strategy for DetectionFrame fully specified — partition interval (daily), auto-creation mechanism, and cleanup/retention policy? [Clarity, data-model.md §DetectionFrame]
- [ ] CHK022 — Is the MonitoringSession→CameraSource relationship implementation clarified — JSONField `camera_ids` vs M2M table? The ER diagram and entity detail appear inconsistent. [Ambiguity, data-model.md §MonitoringSession]
- [ ] CHK023 — Are cascade/protect deletion rules specified for all FK relationships (e.g., what happens to AnomalyEvents when a MonitoringSession is deleted)? [Completeness, data-model.md]
- [ ] CHK024 — Is the `detection_id` FK on AnomalyEvent defined as nullable or required? ER diagram shows mandatory; entity detail shows nullable. [Conflict, data-model.md §AnomalyEvent ER vs entity detail]
- [ ] CHK025 — Are unique constraints explicitly specified where needed (e.g., ExamStudent university_id per exam, Role name uniqueness, Room name uniqueness)? [Completeness, data-model.md]
- [ ] CHK026 — Is the AuditLogEntry `action_type` enum fully enumerated? rest-api.md lists 22 action types; data-model and plan.md cite different counts (8, 11, 14). Are all artifacts aligned? [Conflict, data-model.md vs rest-api.md §Audit Log action_type list]

## REST API Contract Implementation Readiness

- [ ] CHK027 — Are pagination parameters (page size, cursor vs offset, default ordering) consistently specified across all list endpoints? [Consistency, rest-api.md §Common Response Patterns]
- [ ] CHK028 — Are all PATCH endpoints explicitly defined as partial-update (not full replacement)? Is the behavior specified when a read-only field is included in the PATCH payload? [Clarity, rest-api.md]
- [ ] CHK029 — Are authentication and permission requirements explicitly stated for every endpoint group (which require auth, which are admin-only, which are public like /health)? [Completeness, rest-api.md]
- [ ] CHK030 — Does the camera CRUD contract specify the `room_id` assignment field, consistent with data-model's `room` FK on CameraSource? [Consistency, rest-api.md §Cameras vs data-model §CameraSource]
- [ ] CHK031 — Are request validation rules (required fields, min/max lengths, format constraints) explicitly specified for all POST/PATCH request bodies? [Completeness, rest-api.md]
- [ ] CHK032 — Is the export download endpoint's Range request support fully specified for large file downloads (partial content, resume support)? [Completeness, rest-api.md §Session Exports]
- [ ] CHK033 — Are query parameter filter formats specified for date ranges (ISO 8601?), enum values (comma-separated?), and UUID references? [Clarity, rest-api.md query parameters]
- [ ] CHK034 — Does the session start-from-exam endpoint (POST `/exams/{id}/start-session/`) contract align with the session CRUD contract (POST `/sessions/`)? Are there two ways to create sessions, and is the relationship clear? [Consistency, rest-api.md §Exam Board vs §Sessions]

## WebSocket Contract Implementation Readiness

- [ ] CHK035 — Are JSON schemas for all WebSocket message types (10+ server→client, 3 client→server) fully specified with field types, required/optional markers, and example payloads? [Completeness, websocket-api.md]
- [ ] CHK036 — Is the micro-batching window (100ms) for detection frame messages defined with behavior when fewer than batch-size frames arrive within the window? [Clarity, websocket-api.md §detection.frame]
- [ ] CHK037 — Are WebSocket error codes (4001, 4003, 4004, 4005) explicitly mapped to error conditions and their corresponding client-side recovery actions? [Completeness, websocket-api.md §Error Handling]
- [ ] CHK038 — Is the channel subscription model specified — does a client auto-subscribe to all channels on connect, or must it explicitly subscribe per camera/session? [Clarity, websocket-api.md §Connection Lifecycle]
- [ ] CHK039 — Are ping/pong heartbeat parameters (30s interval, 10s timeout) consistently documented, and is the behavior defined when a pong is not received (server-initiated close vs client retry)? [Clarity, websocket-api.md §Connection Lifecycle]

## Requirement Completeness — Authentication & Authorization

- [ ] CHK040 — Are session token mechanics fully specified — cookie-based HttpOnly with CSRF protection, or JWT? Spec says "session token" but plan says "session cookie passthrough" for WebSocket auth. [Clarity, Spec §FR-002 vs plan.md]
- [ ] CHK041 — Are concurrent session requirements defined — can the same user be logged in from multiple browsers/devices simultaneously? [Gap, Spec §FR-004]
- [ ] CHK042 — Is the password change flow (FR-070) fully specified with session invalidation behavior — are other sessions terminated on password change? [Gap, Spec §FR-070]
- [ ] CHK043 — Are the initial admin account creation requirements defined — how does the first admin get created (Django createsuperuser, migration seeding, fixture)? [Gap, Spec §Assumptions]
- [ ] CHK044 — Is the RBAC permission check mechanism specified for the frontend — route guards reading role from store, or server-side redirect? [Clarity, Spec §FR-051 vs tasks-frontend.md]

## Requirement Completeness — Camera & Video Pipeline

- [ ] CHK045 — Is the go2rtc WHEP endpoint URL pattern specified for frontend integration? The frontend must know the URL to request video streams. [Gap, Spec §FR-008 → plan.md]
- [ ] CHK046 — Are RTSP URL validation rules specified for the camera connect flow? (e.g., schema validation, reachability check, timeout) [Gap, Spec §FR-006]
- [ ] CHK047 — Is the local device selection flow (webcam/USB) fully specified — browser API (getUserMedia), permission handling, device enumeration, and fallback? [Gap, Spec §FR-006]
- [ ] CHK048 — Is the behavior specified when a camera feed's resolution differs from expected — scaling, letterboxing, or cropping within grid tiles? [Gap, Spec §FR-007]
- [ ] CHK049 — Are bounding box coordinate systems specified — normalized (0-1) or pixel coordinates? Does the frontend need to scale based on video resolution vs canvas size? [Gap, websocket-api.md §detection.frame → Spec §FR-011]
- [ ] CHK050 — Is the camera preview state (connected but not recording) visually distinguished from the recording state in the UI requirements? [Clarity, Spec §FR-006]

## Requirement Completeness — Detection & Predictions

- [ ] CHK051 — Are detection class names and their corresponding bounding box colors enumerated for all three themes? FR-011 says "color-coded by detection class" but does not list classes or colors. [Gap, Spec §FR-011, §FR-025]
- [ ] CHK052 — Are the pyramid detection layer names and their possible values fully enumerated? FR-016 mentions "posture, gaze direction" but the WebSocket contract shows 4 layers (posture, horizontal_gaze, depth_gaze, vertical_gaze). Is the spec aligned? [Consistency, Spec §FR-016 vs websocket-api.md §prediction.update]
- [ ] CHK053 — Is the mapping from detection class to tracking ID assignment specified — which detected classes get tracking IDs and which are display-only (e.g., teachers per FR-014)? [Clarity, Spec §FR-012, §FR-014]
- [ ] CHK054 — Are the anomaly rule engine configuration parameters fully specified — default thresholds, duration windows, configurable fields, and storage format (JSON, YAML, admin UI)? [Clarity, Spec §FR-017]
- [ ] CHK055 — Is the "mutually exclusive predictions" constraint (FR-020) fully enumerated — which specific prediction pairs are mutually exclusive? [Completeness, Spec §FR-020]

## Requirement Completeness — Anomaly Triage & Notes

- [ ] CHK056 — Is the first-write-wins conflict resolution mechanism fully specified for implementation — optimistic locking with version field, or status check before update, or database constraint? [Clarity, Spec §FR-041 → data-model.md]
- [ ] CHK057 — Are anomaly status transition rules fully specified as a state machine? Allowed transitions: new→acknowledged, new→dismissed, acknowledged→new (admin revert), dismissed→new (admin revert). Are any other transitions possible? [Completeness, Spec §FR-041, §FR-042, §FR-066]
- [ ] CHK058 — Is the dismissed reason minimum (5 characters) the only validation rule, or are there other constraints (max length, allowed characters, profanity filter)? [Clarity, Spec §FR-042]
- [ ] CHK059 — Are batch triage operations (acknowledge/dismiss multiple anomalies at once) explicitly in scope or out of scope? [Gap, Spec §FR-041, §FR-042]
- [ ] CHK060 — Is the anomaly alert de-emphasis behavior (FR-041 "visually de-emphasized but visible") specified with enough detail for CSS implementation — opacity reduction, color change, position shift? [Clarity, Spec §FR-041]

## Requirement Completeness — Recording & Playback

- [ ] CHK061 — Is the recording file format specified (MP4 container, H.264 codec, resolution, bitrate) beyond "raw video"? [Clarity, Spec §FR-026]
- [ ] CHK062 — Is the synchronization mechanism between video playback timeline and stored detection/prediction metadata specified for overlay rendering during playback? [Gap, Spec §FR-028]
- [ ] CHK063 — Are the anomaly timeline marker interaction requirements specified — click behavior, hover tooltip content, visual indicator style? [Completeness, Spec §FR-029]
- [ ] CHK064 — Is the comment timeline marker specification (FR-059) consistent with the anomaly marker specification (FR-029) — same visual system, or distinct? [Consistency, Spec §FR-029 vs §FR-059]
- [ ] CHK065 — Are recording status values enumerated (recording, completed, failed, corrupted) with transition rules? [Completeness, data-model.md §Recording]
- [ ] CHK066 — Is the server-side recording continuation behavior (FR-032: browser disconnect) specified with enough detail — does it use go2rtc recording, or Django media pipeline, and what triggers stop? [Clarity, Spec §FR-032]

## Requirement Completeness — Exam Board & Countdown

- [ ] CHK067 — Is the auto-start behavior at countdown zero (FR-055) specified for error scenarios — what happens if camera auto-detection fails at auto-start time? [Edge Case, Spec §FR-055, §FR-056]
- [ ] CHK068 — Is the "grace period" behavior defined for late-joining instructors (FR-055: "if the instructor joins after the exam start time, the session MUST already be active")? Who initiates the session if no instructor is present? [Ambiguity, Spec §FR-055]
- [ ] CHK069 — Are the Exam Board's real-time countdown display requirements specified with update frequency (per-second?) and visual transition details at the 5-minute threshold? [Clarity, Spec §FR-055]
- [ ] CHK070 — Is the behavior specified when multiple instructors are assigned to the same exam — do they share a session or each get their own? [Gap, Spec §FR-055 vs Clarification Round 4]
- [ ] CHK071 — Is the "In Progress" badge (FR-072) specified with enough detail for implementation — position, style, update mechanism (polling vs WebSocket)? [Clarity, Spec §FR-072]

## Requirement Completeness — Admin Features

- [ ] CHK072 — Is the "force change password on next login" flow (FR-060: "password reset by admin sets a new password that the user should change") specified as mandatory or advisory? Is there a backend flag/mechanism? [Ambiguity, Spec §FR-060]
- [ ] CHK073 — Are built-in role immutability rules specified — can Admin/Instructor role permissions be viewed but not edited? (rest-api.md returns 400 for built-in edit) [Completeness, Spec §FR-061 → rest-api.md §Admin Roles]
- [ ] CHK074 — Is the admin session review playback (FR-065) specified with the same overlay capabilities as instructor playback (FR-028), or a subset? [Clarity, Spec §FR-065 vs §FR-028]
- [ ] CHK075 — Is the admin triage revert WebSocket broadcast (FR-067) specified — which channel, which message type, and who receives it? [Completeness, Spec §FR-067 → websocket-api.md]
- [ ] CHK076 — Are user deactivation side effects fully specified — session termination, active monitoring graceful stop, audit trail? [Completeness, Spec §FR-060 → rest-api.md §DELETE user]

## Requirement Completeness — Export & Async Operations

- [ ] CHK077 — Is the Celery task configuration for exports specified — queue name, concurrency limit, timeout, retry policy? [Gap, Spec §FR-047 → plan.md]
- [ ] CHK078 — Is the export progress polling mechanism specified for the frontend — polling interval, exponential backoff, or WebSocket notification on completion? [Clarity, Spec §FR-047 → tasks-frontend.md]
- [ ] CHK079 — Is the export failure cleanup specified — are partial archives deleted, and is disk space reclaimed? [Edge Case, Spec §FR-047]
- [ ] CHK080 — Are concurrent export limits per user/session defined? rest-api.md returns 409 for duplicate session export, but can a user export multiple different sessions simultaneously? [Gap, rest-api.md §Session Exports]
- [ ] CHK081 — Is the export archive JSON schema (detections.json, anomalies.json, session_metadata.json, tracking_ids.json) specified with field definitions? [Completeness, rest-api.md §Export Download]

## Requirement Clarity — Vague or Unmeasurable Terms

- [ ] CHK082 — Is "real-time" quantified consistently across the spec? FR-008 says "<1 second display latency"; FR-019 says "real-time via WebSocket"; FR-068 says "update in real-time." Are latency targets defined for each? [Clarity, Spec §FR-008, §FR-019, §FR-068]
- [ ] CHK083 — Is "configurable" defined with a configuration mechanism for: inactivity timeout (FR-004), anomaly rule engine (FR-017), and retention policy (Assumptions)? [Clarity, Spec §FR-004, §FR-017, §Assumptions]
- [ ] CHK084 — Is "prominent countdown display" (FR-055) specified with measurable visual properties — font size increase, color change, animation, position change? [Clarity, Spec §FR-055]
- [ ] CHK085 — Is "visually de-emphasized" (FR-041 acknowledged anomalies) specified sufficiently for consistent implementation across 3 themes? [Clarity, Spec §FR-041]
- [ ] CHK086 — Is "sufficient color contrast" (FR-024, WCAG 2.1 AA) quantified with specific contrast ratios (4.5:1 for normal text, 3:1 for large text) and verified for all 3 themes? [Measurability, Spec §FR-024]
- [ ] CHK087 — Is "200ms" filter application time (FR-015) a hard requirement with measurement methodology, or a soft target? [Measurability, Spec §FR-015]
- [ ] CHK088 — Is "30 seconds" auto-restart (FR-049) specified with a monitoring mechanism — systemd watchdog, Docker health check, or Supervisor? [Clarity, Spec §FR-049]

## Requirement Consistency — Cross-FR Alignment

- [ ] CHK089 — Do camera reconnection retries (FR-010: 5 retries) and WebSocket reconnection retries (FR-019: 10 retries) intentionally differ? Is the asymmetry documented with rationale? [Consistency, Spec §FR-010 vs §FR-019]
- [ ] CHK090 — Are anomaly note length constraints consistent between FR-043 ("no maximum") and data-model AnomalyNote content field (TextField)? Does the REST API enforce any limit? [Consistency, Spec §FR-043 vs data-model.md §AnomalyNote]
- [ ] CHK091 — Is the instructor comment model (FR-058) consistent with the anomaly note model (FR-043) — both are timestamped text with user attribution, but they serve different entities. Are their schemas intentionally different? [Consistency, data-model.md §InstructorComment vs §AnomalyNote]
- [ ] CHK092 — Does the dashboard endpoint response shape (rest-api.md) satisfy all FR-046 data requirements (active sessions, camera status, upcoming exams for instructors; stats + feeds for admins)? [Consistency, Spec §FR-046 → rest-api.md §Dashboard]
- [ ] CHK093 — Are theme names consistent across spec (FR-021: "Black+Purple, Full Black, White"), plan.md, and tasks-frontend.md? [Consistency, Spec §FR-021 across artifacts]

## Scenario Coverage — Alternate & Exception Flows

- [ ] CHK094 — Are requirements defined for what happens when a monitoring session is started but no detection pipeline is running (go2rtc healthy, inference worker down)? [Coverage, Edge Case, Spec §FR-038]
- [ ] CHK095 — Are requirements specified for an instructor navigating away from Camera Feed with an active session — warning dialog, background recording confirmation? [Coverage, Gap]
- [ ] CHK096 — Are requirements defined for WebSocket reconnection during active triage — does pending triage state survive reconnection, or must the instructor re-select the anomaly? [Coverage, Spec §FR-019 + §FR-041]
- [ ] CHK097 — Are requirements specified for handling a role change (FR-053) while the instructor has the Predictions Panel open — graceful redirect, or maintain view? [Coverage, Spec §FR-053]
- [ ] CHK098 — Are requirements defined for the admin anomaly feed (FR-071) when all sessions end — does the feed show historical data, empty state, or redirect? [Coverage, Spec §FR-071]
- [ ] CHK099 — Are requirements specified for what happens when go2rtc becomes unreachable mid-session but the Django backend remains healthy (split-brain scenario)? [Coverage, Edge Case]
- [ ] CHK100 — Are requirements defined for exam scheduling conflicts — what happens when two exams are scheduled in the same room at overlapping times? [Coverage, Gap, Spec §FR-063]

## Edge Case Coverage

- [ ] CHK101 — Are requirements defined for extremely long instructor comments or anomaly notes — text wrapping, scrolling, truncation with "show more"? [Edge Case, Gap]
- [ ] CHK102 — Are requirements specified for handling a camera that delivers frames at a rate significantly outside the expected 10-15 fps range (<1 fps or >30 fps)? [Edge Case, Gap]
- [ ] CHK103 — Are requirements defined for the recording playback experience when detection data exists but the video file is missing or corrupted? [Edge Case, Gap]
- [ ] CHK104 — Are requirements specified for the export flow when recording files are larger than available disk space? [Edge Case, Spec §FR-047]
- [ ] CHK105 — Are requirements defined for concurrent admin revert (FR-066) — what if two admins revert different anomalies in the same session simultaneously? [Edge Case, Spec §FR-066]
- [ ] CHK106 — Are requirements specified for timezone handling — are all timestamps stored in UTC, displayed in local time, and is the conversion mechanism documented? [Edge Case, Gap]
- [ ] CHK107 — Are requirements defined for the Exam Board when an instructor has >20 scheduled exams — scrolling, pagination, search? [Edge Case, Spec §FR-054]

## Non-Functional Requirements — Performance

- [ ] CHK108 — Are performance requirements specified under different load conditions — single instructor (1×1 camera) vs peak capacity (20 instructors × 8 cameras)? [Coverage, Spec §Assumptions]
- [ ] CHK109 — Is the maximum acceptable latency for WebSocket detection frame delivery quantified (end-to-end from camera to bounding box render)? [Gap, Spec §FR-011, §FR-019]
- [ ] CHK110 — Are memory and CPU budgets defined for the frontend SPA — especially with 8 simultaneous WHEP video streams plus canvas overlay rendering? [Gap]
- [ ] CHK111 — Is the database query performance target specified for high-volume tables (DetectionFrame with millions of rows, AuditLogEntry with append-only growth)? [Gap, data-model.md]
- [ ] CHK112 — Are WebSocket backpressure requirements defined — what happens when the client cannot consume messages as fast as the server produces them (8-10 batched detection msgs/sec)? [Gap, websocket-api.md]

## Non-Functional Requirements — Security

- [ ] CHK113 — Are CORS configuration requirements specified for the SPA-to-Django communication (allowed origins, methods, headers)? [Gap, Spec §Security context]
- [ ] CHK114 — Are CSRF protection requirements specified for state-changing REST endpoints when using cookie-based session auth? [Gap, Spec §FR-002]
- [ ] CHK115 — Are API authorization checks specified at the object level (not just role level) — e.g., can an instructor access another instructor's session data via direct API call? [Gap, Spec §FR-051]
- [ ] CHK116 — Are WebSocket authorization requirements specified beyond initial handshake — is channel-level authorization enforced (e.g., instructor can only subscribe to their own session's channels)? [Gap, websocket-api.md §Connection Lifecycle]
- [ ] CHK117 — Are password complexity requirements specified for user creation (FR-060) and password change (FR-070)? [Gap, Spec §FR-060, §FR-070]
- [ ] CHK118 — Is the audit log immutability requirement (FR-069: "no updates or deletes") specified with a technical enforcement mechanism (database permissions, append-only table, or application-level)? [Clarity, Spec §FR-069]

## Non-Functional Requirements — Accessibility

- [ ] CHK119 — Are ARIA live region requirements defined for real-time anomaly alerts so screen readers announce new alerts without focus change? [Gap, Spec §FR-017, §FR-024]
- [ ] CHK120 — Are keyboard navigation patterns specified for complex components — camera grid tile selection, playback timeline seeking, anomaly triage actions? [Gap, Spec §FR-035]
- [ ] CHK121 — Are focus management requirements defined for modal dialogs (camera add form, anomaly dismiss reason, export confirmation, user create form)? [Gap, Spec §FR-035]
- [ ] CHK122 — Are skip-navigation landmarks specified for the main layout — sidebar, header, content area, video grid? [Gap, Spec §FR-035]
- [ ] CHK123 — Is the bounding box canvas overlay accessible — are detection results available via non-visual means (screen reader alternative text, data table)? [Gap, Spec §FR-011, §FR-024]

## Non-Functional Requirements — Observability & Operations

- [ ] CHK124 — Are structured logging requirements specified for the backend — log format (JSON?), log levels, correlation IDs for request tracing? [Gap, Spec §Assumptions]
- [ ] CHK125 — Are monitoring/alerting requirements specified beyond the health endpoint — who is notified when the system degrades, and through what channel? [Gap, Spec §FR-038, §FR-039]
- [ ] CHK126 — Are database migration requirements specified — migration tool (Django migrations), zero-downtime migration strategy, rollback procedure? [Gap, plan.md]
- [ ] CHK127 — Are environment configuration requirements documented — which values are env vars vs config files, and is a sample .env file specified? [Gap, plan.md §Technical Context]

## Dependencies & Assumptions

- [ ] CHK128 — Is the go2rtc version requirement (≥1.9) validated against WHEP features used, and is the go2rtc configuration contract (API endpoints, auth integration) documented? [Dependency, plan.md §Technical Context]
- [ ] CHK129 — Is the Ultralytics YOLO model specification documented — model weights file, input resolution, confidence thresholds, supported classes? [Dependency, plan.md §Technical Context]
- [ ] CHK130 — Is the Docker Compose sidecar architecture specified with service definitions, network configuration, volume mounts, and health checks? [Dependency, plan.md]
- [ ] CHK131 — Are Python/Node.js version requirements specified (Python 3.11+, Node 20+) and pinned in project configuration (pyproject.toml, .nvmrc)? [Dependency, plan.md]
- [ ] CHK132 — Is the reverse proxy configuration requirement documented — which proxy (nginx, traefik), routing rules for Django + go2rtc + WebSocket upgrade? [Dependency, Spec §FR-008]

## Acceptance Criteria Quality

- [ ] CHK133 — Are acceptance scenarios in the spec (US1–US5) sufficient to validate all 73 FRs, or do some FRs lack testable acceptance criteria? [Coverage, Spec §User Stories]
- [ ] CHK134 — Can success criteria SC-001 through SC-008 be objectively measured without ambiguity? Specifically: "without visible frame drops below 15 FPS" (SC-004) — measurement method? [Measurability, Spec §Success Criteria]
- [ ] CHK135 — Are negative acceptance criteria (things the system must NOT do) explicitly documented — e.g., system must NOT expose raw stack traces (FR-036), must NOT allow instructor access to admin pages (FR-052)? [Coverage, Spec §FRs]
- [ ] CHK136 — Are state transition acceptance criteria defined for all stateful entities (MonitoringSession: active→completed→interrupted, AnomalyEvent: new→acknowledged→dismissed, Export: pending→processing→completed/failed)? [Coverage, Spec + data-model.md]

## Implementation Blockers — Must Resolve Before Coding

- [ ] CHK137 — Is the AuditLog action_type count reconciled across all artifacts to a single canonical list? (Currently ranges from 8 to 22 depending on the artifact.) [Blocker, Conflict]
- [ ] CHK138 — Is the MonitoringSession↔CameraSource relationship type resolved (JSONField vs M2M)? This is a schema-level decision blocking model implementation. [Blocker, Ambiguity, data-model.md]
- [ ] CHK139 — Is the detection_id nullability on AnomalyEvent resolved (ER says required, entity detail says nullable)? This affects FK constraint in Django model. [Blocker, Conflict, data-model.md]
- [ ] CHK140 — Is the camera state machine reconciled between spec (3 states) and data-model (4 states)? This affects the Django choices/enum definition. [Blocker, Conflict, Spec §FR-010 vs data-model.md]
- [ ] CHK141 — Is the WHEP endpoint URL pattern and authentication passthrough mechanism documented for go2rtc integration? Frontend video streaming is blocked without this. [Blocker, Gap]
- [ ] CHK142 — Is the recording deletion endpoint either specified or explicitly deferred? Multiple spec sections reference deletion but no FR or API contract exists. [Blocker, Gap]

---

## Notes

- **Focus**: All domains — Auth, Camera/Video, Detection/Predictions, Anomaly Triage, Recording/Playback, Exam Board, Admin Features, Export, Health, Themes, Accessibility, WebSocket, Data Model, REST API, Security, Observability
- **Depth**: Deep — exhaustive cross-document traceability plus implementation-blocking ambiguities
- **Audience**: Implementation gate — validates specs are ready for developers to begin coding
- **Relationship to existing checklists**: Complements [comprehensive.md](comprehensive.md) (release gate, 68 items) by focusing on developer-facing implementation readiness and cross-artifact consistency
- **Traceability**: 137/142 items (96.5%) include at least one traceability reference (Spec §, data-model §, rest-api.md, websocket-api.md, plan.md, tasks.md, or gap/conflict marker)
- **Blocker items**: CHK137–CHK142 are flagged as implementation blockers requiring resolution before first sprint begins
- Items use markers: `[Gap]`, `[Ambiguity]`, `[Conflict]`, `[Dependency]`, `[Blocker]`, `[Edge Case]`, `[Consistency]`, `[Completeness]`, `[Clarity]`, `[Measurability]`, `[Coverage]`, `[Traceability]`
