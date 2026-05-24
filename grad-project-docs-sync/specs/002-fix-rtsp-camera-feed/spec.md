# Feature Specification: Fix RTSP Camera Feed Connection

**Feature Branch**: `002-fix-rtsp-camera-feed`  
**Created**: 2026-03-04  
**Status**: Draft  
**Input**: User description: "Check on the RTSP Protocol in our web app, we have an issue that after putting the URL of the RTSP Link and click connect the Camera Feed does not show, we need to add Error Messages, Clarifying Messages to the End User of the WebApp and fix the issue and ensure it is connected to the entire system with excellent integration"

## Problem Statement

When an instructor adds an RTSP camera URL and clicks "Connect," the camera feed never appears on screen. The root cause is that clicking "Connect" only updates the camera's status in the database to `connected` — it does **not** register the RTSP stream with the media relay service (go2rtc). As a result, the browser's WebRTC (WHEP) request fails silently because there is no stream to play back. Additionally, the user receives no meaningful error message explaining why the feed is not showing. This feature fixes the broken camera pipeline end-to-end and adds clear, actionable feedback at every step of the connection lifecycle.

## Assumptions

- The go2rtc media relay service is already deployed and accessible to the backend at its API address (port 1984).
- RTSP cameras are on a network reachable from the go2rtc service container.
- The existing `CameraSource` model, API endpoints, and frontend camera pages remain the foundation — this feature extends them, not replaces them.
- The WHEP video path will be routed through the nginx authenticated proxy (`/whep/{camera_id}/`) rather than directly to go2rtc, in order to enforce session-based auth.
- The maximum number of cameras per instructor remains 8.
- Standard connection timeout of 10 seconds is reasonable for RTSP probe validation. The probe performs a full three-stage check: TCP connect, RTSP DESCRIBE handshake, and decoding one video frame (~5-10 seconds total).
- The existing WebSocket infrastructure (Django Channels, `CameraStatusConsumer`) will be used for real-time status updates.
- RTSP URLs containing embedded credentials are encrypted at rest and masked in all user-facing surfaces and logs.
- Stream health monitoring uses a polling approach (every 15 seconds) against the media relay's stream status API, providing ~15-second detection latency for stream loss events.
- There is no hard global concurrent stream limit. The system relies on the media relay's natural resource limits and surfaces capacity errors gracefully when the relay cannot accept new streams.
- Connection event logs are retained for 90 days, after which they are automatically purged.
- All user-facing strings are English only for this release, but stored as extractable constants for future i18n support. Accessibility follows WCAG 2.1 AA standards.

## Clarifications

### Session 2026-03-04

- Q: How should RTSP URL credentials be handled (storage, display, logs)? → A: Encrypt RTSP URL at rest, mask credentials in logs/UI/error messages (e.g., show `rtsp://***:***@192.168.1.10/stream`).
- Q: How does the backend detect that a go2rtc stream has dropped? → A: A periodic background task polls go2rtc's stream status API every 15 seconds for all connected cameras (simple, decoupled, ~15s detection latency).
- Q: What happens when go2rtc restarts — who re-registers streams and what does the user see? → A: The existing 15s health-check task detects missing streams and auto-re-registers all cameras that were in connected status. Instructors are notified with a banner/toast explaining the streaming service restarted, and a progress indicator (showing each camera being re-registered step-by-step) or a countdown timer is displayed until all streams are fully restored.
- Q: How and where are error messages displayed, and how long do they persist? → A: A persistent inline error banner on the camera tile (stays until the user retries or dismisses it) plus a brief toast notification (~5 seconds) for global awareness. This ensures the error is both immediately noticeable and persistently visible at the point of the problem.
- Q: Is there a global concurrent stream limit across all instructors? → A: No hard global limit. The system relies on go2rtc's natural resource limits. If go2rtc cannot accept a new stream (e.g., out of memory or CPU saturation), the backend surfaces a clear "Streaming service at capacity — please try again later" error to the instructor.
- Q: How deep should RTSP URL validation go (TCP check vs. DESCRIBE vs. full frame probe)? → A: Full probe — TCP connect + RTSP DESCRIBE handshake + attempt to decode one video frame to confirm the stream is actually producing video data. This is the most thorough approach (~5-10 seconds) and catches issues like cameras that respond to DESCRIBE but produce no actual video.
- Q: When a camera reconnects after a disconnection, does the detection pipeline auto-resume? → A: Yes. If the exam session is still active, the detection pipeline (YOLO inference → bounding boxes → anomaly detection) automatically resumes when the camera stream is restored. No instructor action required for seamless continuity during transient disruptions.
- Q: How long are connection event logs retained, and who can see them? → A: Retained for 90 days. Instructors see a simplified recent history (last 10 events) on the camera detail page. Administrators get full access via the admin panel with filtering capabilities.
- Q: Can two instructors (or the same instructor twice) add the same RTSP URL? → A: Unique per instructor — the same instructor cannot add the same URL twice (rejected with a clear message), but different instructors can share the same physical camera stream. The media relay reuses the underlying RTSP connection when multiple registrations point to the same source.
- Q: Can an instructor edit the RTSP URL after a camera is created? → A: Yes, when the camera is in disconnected or error state. If the camera is currently connected, the system auto-disconnects first (unregisters the stream), updates the URL, and prompts the user to reconnect with the new URL.
- Q: After connecting a camera from the Camera List page, should the user be auto-navigated to the Feed page? → A: No. Stay on the current page. Show a success toast with a clickable "View Live Feed →" link to the Camera Feed page. This doesn't interrupt the workflow when connecting multiple cameras in sequence.
- Q: Can an instructor delete a camera that is currently connected and streaming? → A: Yes. The system shows a confirmation dialog warning about the active stream, auto-disconnects and unregisters the stream from the media relay, and removes the camera entry. However, all recorded video, model predictions (detections, anomalies), session metadata, and historical data associated with the camera's sessions MUST be preserved and not deleted.
- Q: What admin-level visibility and control exists for camera connection issues? → A: Admins get a system-wide camera status dashboard in the admin panel showing all cameras across all instructors with statuses and go2rtc relay health. Admins can force-disconnect any camera, connect/disconnect cameras on behalf of instructors, and view live feeds from any camera.
- Q: Should error messages support internationalization and accessibility? → A: English only for now, but use string constants (not inline strings) so they can be easily extracted for i18n later. All dynamic status changes, error banners, and progress indicators must use ARIA live regions, include screen-reader-friendly text for badges, and meet WCAG 2.1 AA color contrast requirements.
- Q: What are the specific WHEP retry parameters (count, backoff timing)? → A: 3 retries with exponential backoff starting at 2 seconds (2s → 4s → 8s), totaling ~14 seconds before showing a final error to the user.
- Q: How is the gap period tracked when a camera disconnects during an active exam session? → A: The gap is recorded as a "monitoring interruption" event with start and end timestamps. The session timeline shows a visual gap indicator, and the session summary includes the total interruption duration.
- Q: Should the system support `rtsps://` (RTSP over TLS) in addition to `rtsp://`? → A: Yes. Support both `rtsp://` and `rtsps://` schemes. The media relay handles both natively. Update frontend and backend validation to accept either scheme.
- Q: What are the specific backend reconnection backoff parameters for stream loss? → A: 3 retries at 2s → 4s → 8s (~14 seconds total). Fast recovery for transient issues; if all retries fail, the camera status moves to "Error" with an appropriate message.
- Q: How does the UI handle multiple cameras failing simultaneously? → A: Consolidated notification — if 3 or more cameras fail within a 10-second window, show a single summary banner (e.g., "4 of 6 cameras lost connection — Reconnecting all…") instead of individual toasts. Each camera tile still shows its own status overlay independently.
- Q: What happens to ConnectionEvent records when their parent camera is deleted? → A: SET_NULL — preserve all connection events with the camera FK set to NULL. Events remain queryable by date, user, event type, and other fields even after the camera is removed. This aligns with FR-022's data preservation mandate.
- Q: Can a camera transition directly from connected to error, skipping reconnecting? → A: No. Every stream loss from connected state always goes through reconnecting first (3 retries with exponential backoff per FR-008). Only if all 3 retries fail does the camera transition to error. There is no direct connected → error path for stream loss.
- Q: Should "Poor Connection" quality degradation detection be in scope? → A: Out of scope / deferred. go2rtc doesn't expose bitrate or packet-loss metrics via its REST API. The existing health check covers complete stream loss. Quality degradation monitoring is a future enhancement.
- Q: How should FR-002 and FR-012 be scoped to avoid duplication? → A: Split responsibilities — FR-002 is the authoritative server-side three-stage probe requirement. FR-012 is narrowed to client-side URL format validation (regex for rtsp:// or rtsps://) and explicit rtsps:// scheme acceptance only. Server-side probe text removed from FR-012 to eliminate overlap.
- Q: Should SC-005 and SC-007 be reclassified as aspirational (non-blocking) goals? → A: Yes. Both are post-deployment UX metrics that cannot be verified during implementation. Reclassified as aspirational with testable proxy criteria added. SC-001–SC-004, SC-006, SC-008–SC-009 remain hard blocking gates.

## User Scenarios & Testing *(mandatory)*

### User Story 1 — Connect Camera and See Live Feed (Priority: P1)

An instructor enters an RTSP URL for a classroom camera, clicks "Connect," and within a few seconds sees a live video feed in the monitoring dashboard. A progress indicator shows connection stages so the instructor knows the system is working.

**Why this priority**: This is the core broken functionality. Without a working camera feed, the entire exam monitoring system is non-functional. No other feature matters if cameras don't display.

**Independent Test**: Can be fully tested by creating a camera with a valid RTSP URL, clicking connect, and verifying the live video appears. Delivers the fundamental value of the monitoring system.

**Acceptance Scenarios**:

1. **Given** an instructor has added a camera with a valid, reachable RTSP URL, **When** they click "Connect" from the Camera List page, **Then** the system displays a progress indicator (e.g., "Validating stream… Connecting… Live"), registers the stream with the media relay, updates the status badge to green "Connected," and shows a success toast with a clickable "View Live Feed →" link. The instructor remains on the Camera List page.
2. **Given** a camera is connected and showing live feed, **When** the instructor navigates to the Camera Feed page, **Then** the video feed is displayed in the camera grid with a green "Live" badge.
3. **Given** a camera is connected, **When** the instructor clicks "Disconnect," **Then** the stream is unregistered from the media relay, the video feed stops, and the camera status updates to "Disconnected."

---

### User Story 2 — See Clear Error Messages on Connection Failure (Priority: P1)

When an instructor tries to connect a camera and something goes wrong — the RTSP URL is unreachable, the stream format is unsupported, or the media relay is down — the system displays a specific, user-friendly error message explaining what happened and suggesting what to try next.

**Why this priority**: Without error messages, users are left staring at a blank screen with no idea what happened. This is equally critical to the fix itself because the user experience is broken even if the connection works 90% of the time.

**Independent Test**: Can be tested by attempting to connect with an invalid/unreachable RTSP URL and verifying that a descriptive error message appears instead of a blank/frozen state.

**Acceptance Scenarios**:

1. **Given** an instructor enters an RTSP URL that is unreachable (wrong IP, camera offline), **When** they click "Connect," **Then** the system displays an error message such as "Camera Unreachable — Could not connect to the camera at the provided URL. Please verify the IP address and that the camera is powered on and connected to the network."
2. **Given** the media relay service is unavailable, **When** the instructor clicks "Connect," **Then** the system displays an error message such as "Streaming Service Unavailable — The video streaming service is temporarily down. Please try again later or contact your administrator."
3. **Given** an instructor enters an RTSP URL with incorrect credentials embedded, **When** they click "Connect," **Then** the system displays an error message such as "Authentication Failed — The camera rejected the provided credentials. Please verify the username and password in the RTSP URL."
4. **Given** a connection error occurs, **When** the error message is displayed, **Then** the camera status is set to "Error" with a red badge, a persistent inline error banner appears on the camera tile (staying until the user retries or dismisses), and a brief toast notification (~5 seconds) is shown for global awareness. The error is visible in both the camera list and the camera feed page.

---

### User Story 3 — Real-Time Camera Status Updates (Priority: P2)

When a camera's status changes (connected, disconnected, error, reconnecting), all open browser tabs viewing that camera receive the update in real time without refreshing the page. Status badges, overlays, and messages update immediately.

**Why this priority**: Real-time feedback is essential for monitoring but depends on the core connection (P1) working first. This ensures the UI stays synchronized across all viewers.

**Independent Test**: Can be tested by opening the camera feed page in two browser tabs, then connecting/disconnecting a camera from one tab and verifying the other tab updates within 2 seconds.

**Acceptance Scenarios**:

1. **Given** two instructors are viewing the same camera feed page, **When** one instructor connects a camera, **Then** both see the status change to "Connected" and the video feed appear within 2 seconds of each other.
2. **Given** a camera is connected and streaming, **When** the RTSP source becomes unreachable (e.g., camera loses power), **Then** the system detects the stream loss, updates the camera status to "Reconnecting," and displays a "Connection Lost — Reconnecting…" overlay on the video tile. If reconnection fails after 3 attempts, the status changes to "Error" with an appropriate message. If reconnection succeeds and an exam session is active, the detection pipeline automatically resumes without instructor action.
3. **Given** a camera is in "Error" or "Disconnected" state, **When** the instructor clicks "Connect" again, **Then** the system retries the full connection flow (validate → register → stream).

---

### User Story 4 — Connection Progress Feedback (Priority: P2)

The connection process shows step-by-step progress so the instructor understands what is happening during the multi-second connection workflow, rather than seeing a spinner with no context.

**Why this priority**: Reduces user anxiety and support requests by making the multi-step connection process transparent.

**Independent Test**: Can be tested by clicking "Connect" on a camera and observing that distinct progress stages are displayed (e.g., "Validating stream…", "Registering with streaming service…", "Establishing video connection…", "Live").

**Acceptance Scenarios**:

1. **Given** an instructor clicks "Connect," **When** the system processes the connection, **Then** the user sees a staged progress indicator showing at least three stages: validation, stream registration, and video connection establishment.
2. **Given** the connection fails at the validation stage, **When** the error occurs, **Then** the progress indicator stops at that stage and shows the relevant error message alongside the stage that failed.

---

### User Story 5 — WHEP Video Path Through Authenticated Proxy (Priority: P3)

The browser's WebRTC video requests are routed through the authenticated reverse proxy rather than directly to the media relay, ensuring that only authenticated users can access camera streams.

**Why this priority**: Security hardening — the core functionality works without this, but unauthenticated video access is a security risk that must be addressed.

**Independent Test**: Can be tested by verifying that WHEP requests go through the `/whep/{camera_id}/` nginx route and that unauthenticated requests are rejected with a 401/403 status.

**Acceptance Scenarios**:

1. **Given** an authenticated instructor is viewing a connected camera, **When** the browser establishes a WebRTC connection, **Then** the WHEP signaling request is sent through the authenticated proxy route, not directly to the media relay.
2. **Given** an unauthenticated request is made to the WHEP endpoint, **When** the proxy validates the session, **Then** the request is rejected and no video stream is provided.

---

### Edge Cases

- What happens when the RTSP URL uses a non-standard port? The system must accept any valid RTSP URL with custom ports (e.g., `rtsp://192.168.1.10:8554/stream`).
- What happens when the user enters a URL without the `rtsp://` scheme? The frontend validation rejects it with a message: "URL must start with rtsp:// or rtsps://"
- What happens when the go2rtc service restarts while cameras are connected? The health-check task detects that previously-connected streams are no longer registered and automatically re-registers them. Instructors see a notification banner (e.g., "Streaming service restarted — restoring camera feeds…") with either a step-by-step progress indicator showing each camera being restored, or a countdown timer estimating time to full recovery. The notification clears automatically once all streams are back online.
- What happens when the instructor rapidly clicks Connect/Disconnect? The system must debounce or queue these actions to prevent race conditions, showing a disabled button with a spinner during the operation.
- What happens when the RTSP stream provides video but no audio? The system must handle video-only streams gracefully without errors.
- What happens when the camera is already connected and the user clicks Connect again? The system should either no-op (if already connected) or re-validate and reconnect, displaying appropriate feedback.
- What happens when the network between go2rtc and the RTSP camera degrades (packet loss, high latency)? **Deferred / Out of scope for this feature.** The existing health check detects complete stream loss. Quality degradation monitoring (bitrate/packet-loss metrics) is a future enhancement — go2rtc does not expose these metrics via its REST API.
- What happens when the same instructor tries to add the same RTSP URL twice? The system rejects the second addition with a clear message: "You already have a camera with this URL." Different instructors adding the same URL is allowed; the media relay reuses the underlying stream.
- What happens when an instructor wants to fix a typo in the RTSP URL? The instructor can edit the URL when the camera is disconnected or in error state. If the camera is connected, the system auto-disconnects first, then allows the edit and prompts the user to reconnect. Delete-and-recreate is not required.
- What happens when an instructor deletes a camera that is currently streaming? A confirmation dialog warns the instructor that the feed will stop immediately. The system auto-disconnects, unregisters the stream, and removes the camera entry. All recorded video, model predictions, session metadata, and historical data from past/current sessions are preserved and remain accessible.
- What happens when multiple cameras fail simultaneously (e.g., network switch reboot)? If 3 or more cameras fail within a 10-second window, the system shows a single consolidated banner ("4 of 6 cameras lost connection — Reconnecting all…") instead of flooding the UI with individual toasts. Each camera tile still shows its own reconnecting/error overlay.
- What happens when the maximum camera limit (8) is reached and a user tries to connect another? The system must display a clear message: "Maximum number of cameras reached. Please disconnect an existing camera first."
- What happens when many instructors run sessions simultaneously and the media relay runs out of resources? The system surfaces a "Streaming service at capacity — please try again later" error to the affected instructor. No streams from other instructors are interrupted.
- What happens to the detection pipeline (YOLO inference, bounding boxes, anomaly alerts) when a camera disconnects and reconnects? If the exam session is still active, the pipeline automatically resumes upon stream restoration. No instructor action is needed. If the session ended during the disconnection, the pipeline does not restart.
- What happens to the exam session timeline when a camera disconnects mid-session? The gap is recorded as a "monitoring interruption" event with start/end timestamps. The session timeline shows a visual gap indicator, and the session summary reports the total interruption duration. This ensures instructors and administrators know exactly when monitoring was interrupted.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST register the RTSP stream with the media relay service when the user clicks "Connect" and unregister it when the user clicks "Disconnect."
- **FR-002**: System MUST validate that the RTSP URL is reachable before registering it with the media relay, using a full probe: TCP connection, RTSP DESCRIBE handshake (verifying valid stream metadata/SDP), and decoding one video frame to confirm the stream is producing video data. The probe MUST complete within a configurable timeout (default 10 seconds). If any stage fails, the system MUST report which stage failed (network unreachable, RTSP handshake rejected, or no video data).
- **FR-003**: System MUST display specific, user-friendly error messages when connection fails, categorized by failure type: unreachable host, authentication failure, unsupported stream, or media relay unavailability. Errors MUST be shown as a persistent inline banner on the affected camera tile (remaining until the user retries or dismisses) and additionally as a brief toast notification (~5 seconds) for global awareness.
- **FR-004**: System MUST show a multi-stage progress indicator during the connection process (validation → registration → video establishment → live).
- **FR-005**: System MUST route WebRTC (WHEP) signaling requests through the authenticated reverse proxy, not directly to the media relay.
- **FR-006**: System MUST broadcast camera status changes (connected, disconnected, error, reconnecting) to all connected clients in real time via WebSocket.
- **FR-007**: System MUST update the camera's status in the database to reflect the actual stream state (not just the user's intent).
- **FR-008**: System MUST attempt automatic reconnection when a connected stream is unexpectedly lost, with a maximum of 3 retry attempts using exponential backoff (2s → 4s → 8s, ~14 seconds total). Stream loss is detected via a periodic background health-check task that polls the media relay's stream status API every 15 seconds for all connected cameras. If all retries fail, the camera status transitions to "Error" with a descriptive message.
- **FR-009**: System MUST provide WHEP auto-retry logic in the browser when the WebRTC connection fails or drops. Retry parameters: 3 attempts with exponential backoff (2s → 4s → 8s, ~14 seconds total). After all retries are exhausted, display a final error message on the camera tile. During retries, show a "Reconnecting…" indicator with the current attempt number (e.g., "Reconnecting… attempt 2/3").
- **FR-010**: System MUST disable the Connect/Disconnect button during an ongoing operation and show a loading state to prevent duplicate requests.
- **FR-011**: System MUST display a "Connection Lost — Reconnecting…" overlay on the camera tile when reconnection is in progress, and a persistent inline error banner on the tile if all retries are exhausted. The error banner remains until the user retries or manually dismisses it. A brief toast notification (~5 seconds) is also shown for global awareness.
- **FR-012**: System MUST validate RTSP URLs on the client side using a format check (regex matching `^rtsps?://`) before submission to the server. Both `rtsp://` and `rtsps://` (RTSP over TLS) schemes MUST be accepted by client-side and server-side validation. The server-side three-stage probe is defined in FR-002; FR-012 governs only client-side format validation and scheme acceptance.
- **FR-013**: System MUST log all connection attempts, failures, and status transitions for troubleshooting purposes. Connection event logs MUST be retained for 90 days. Instructors MUST see a simplified recent connection history (last 10 events) on the camera detail/settings page. Administrators MUST have full access to all connection event logs via the admin panel with filtering by camera, instructor, event type, and date range.
- **FR-014**: System MUST handle video-only streams (no audio track) without errors or degraded behavior.
- **FR-015**: System MUST encrypt RTSP URLs at rest in the database. Credentials embedded in RTSP URLs MUST be masked in all UI displays, error messages, and log output (e.g., `rtsp://***:***@192.168.1.10/stream`). Decryption occurs only at the point of use (stream registration with the media relay).
- **FR-016**: System MUST automatically re-register all previously-connected streams when the media relay service restarts. Instructors MUST be notified with a visible banner or toast message explaining the streaming service restarted. The UI MUST display either a step-by-step progress indicator (showing each camera being restored) or a countdown timer until all streams are fully restored. The notification MUST clear automatically once all feeds are back online.
- **FR-017**: System MUST NOT enforce a hard global limit on concurrent streams. If the media relay rejects a stream registration due to resource exhaustion, the system MUST surface a clear "Streaming service at capacity" error message to the instructor. The per-instructor limit of 8 cameras remains enforced.
- **FR-018**: System MUST automatically resume the detection pipeline (inference, bounding boxes, anomaly detection) when a camera stream is restored after a disconnection, provided the associated exam session is still active. No manual instructor action is required for pipeline resumption.
- **FR-019**: System MUST enforce RTSP URL uniqueness per instructor — the same instructor cannot add two cameras with the same RTSP URL. The system MUST display a clear message: "You already have a camera with this URL." Different instructors MAY add cameras with the same RTSP URL; the media relay SHOULD reuse the underlying RTSP connection when multiple registrations point to the same source to conserve resources.
- **FR-020**: System MUST allow instructors to edit the RTSP URL of an existing camera when it is in disconnected or error state. If the camera is currently connected, the system MUST automatically disconnect and unregister the stream first, then update the URL and prompt the instructor to reconnect. The edited URL MUST go through the same full validation (three-stage probe) as a new camera before reconnection.
- **FR-021**: When a camera is successfully connected from the Camera List page, the system MUST remain on the current page and show a success toast notification with a clickable "View Live Feed →" link that navigates to the Camera Feed page. The Camera List page MUST also update the camera's status badge to green "Connected" immediately.
- **FR-022**: System MUST allow instructors to delete a camera at any time, including when it is currently connected. If the camera is streaming, the system MUST show a confirmation dialog (e.g., "This camera is currently streaming. Deleting it will stop the feed immediately. Continue?"), then auto-disconnect and unregister the stream from the media relay. All recorded video, model predictions (detections, bounding boxes, anomaly alerts), session metadata, and connection event history associated with the camera's sessions MUST be preserved and NOT deleted with the camera.
- **FR-023**: System MUST provide administrators with a system-wide camera status dashboard in the admin panel, displaying all cameras across all instructors with their current statuses, owning instructor, associated room, and a go2rtc media relay health indicator (up/down/degraded). Administrators MUST be able to filter and search cameras by status, instructor, or room.
- **FR-024**: Administrators MUST be able to force-disconnect any camera, connect/disconnect cameras on behalf of any instructor, and view live feeds from any camera directly from the admin dashboard. All admin actions on cameras MUST be logged in the connection event history with the admin user recorded as the actor.
- **FR-025**: All user-facing messages (error banners, toasts, progress indicators, status badges) MUST be defined as string constants (not inline strings) to enable future i18n extraction. All dynamic status updates, error banners, and progress indicators MUST use ARIA live regions for screen-reader announcements. Status badges MUST include screen-reader-friendly alt text. All status colors and badges MUST meet WCAG 2.1 AA color contrast requirements.
- **FR-026**: When a camera disconnects during an active exam session, the system MUST record a "monitoring interruption" event with start timestamp (when the stream was lost) and end timestamp (when the stream was restored or the session ended). The session timeline view MUST display a visual gap indicator for the interruption period. The session summary MUST include the total monitoring interruption duration across all cameras.
- **FR-027**: When 3 or more cameras fail within a 10-second window, the system MUST consolidate toast notifications into a single summary banner (e.g., "4 of 6 cameras lost connection — Reconnecting all…") instead of showing individual toasts per camera. Each camera tile MUST still display its own individual status overlay (reconnecting/error) independently of the consolidated banner.

### Key Entities

- **CameraSource**: Represents a camera device. Key attributes: unique identifier, name, RTSP connection URL, connection type, current status (connected / disconnected / reconnecting / error), error detail message, resolution metadata, associated room, owning instructor. The status and error detail must reflect the actual streaming state, not just the UI action. RTSP URL must be unique per instructor (but may be shared across different instructors). Deleting a camera removes the device entry and stops streaming, but all historical session data (recordings, detections, predictions, anomalies) is preserved.
- **StreamRegistration**: Represents the association between a CameraSource and its active stream in the media relay. Key attributes: camera reference, relay stream identifier (e.g., `camera_{uuid}`), registration timestamp, last health check result.
- **ConnectionEvent**: Represents a log entry for connection lifecycle events. Key attributes: camera reference (SET_NULL on camera deletion — events are preserved with NULL camera FK), event type (connect_attempt, connect_success, connect_failure, disconnect, reconnect_attempt, reconnect_success, reconnect_failure, stream_lost, monitoring_interruption_start, monitoring_interruption_end, force_disconnect, admin_connect), timestamp, error details (if applicable), user who triggered the action, associated session reference (if during an active exam session). Retained for 90 days. Instructors see the last 10 events per camera; administrators see all events with search/filter capabilities.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: After entering a valid, reachable RTSP URL and clicking "Connect," the live camera feed is visible in the browser within 15 seconds, for at least 95% of connection attempts.
- **SC-002**: When a connection fails, 100% of failure cases display a specific, categorized error message (not a generic error or blank screen) within 10 seconds of clicking "Connect."
- **SC-003**: Camera status changes are reflected across all viewing clients within 2 seconds of the state transition occurring.
- **SC-004**: The system successfully auto-reconnects after a transient stream loss in at least 80% of cases within 30 seconds.
- **SC-005** *(aspirational — post-deployment metric)*: Users can identify the exact stage of a connection failure (validation, registration, or video establishment) from the error message displayed, reducing support escalations related to camera issues by at least 60%. **Testable proxy**: Every connection failure path (network unreachable, auth failure, no video, relay unavailable) produces an error response containing an `error_code`, `stage`, and human-readable `detail` field — verified by unit tests covering all `CameraConnectionError` subclasses.
- **SC-006**: All WHEP video requests are routed through the authenticated proxy, with 0% of unauthenticated requests resulting in video access.
- **SC-007** *(aspirational — post-deployment metric)*: 90% of instructors can successfully connect their first camera on the first attempt without external assistance, as measured by task completion rate. **Testable proxy**: The connect flow completes in ≤3 user actions (enter URL → click Connect → see feed), verified by an E2E Playwright test that connects a camera from the Camera List page and confirms the live feed appears.
- **SC-008**: Rapid repeated Connect/Disconnect clicks do not produce orphaned streams or inconsistent camera states — the system handles 100% of race conditions gracefully.
- **SC-009**: Administrators can view the status of all cameras across all instructors from a single dashboard, identify cameras in error state, and take corrective action (force-disconnect, reconnect on behalf) without needing to log in as the instructor.
