# Feature Specification: Unified Video Analytics Overlay Pipeline

**Feature Branch**: `004-video-upload-inference-tab`  
**Created**: 2026-04-30  
**Status**: Draft  
**Input**: User description: "Build a production-grade video analytics pipeline for a full-stack web application that supports two input modes: real-time RTSP camera feed and uploaded video processing. The frontend must display the original stream or video with bounding boxes, tracking IDs, labels, and model-specific detections using backend inference results."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Upload Video for Analysis (Priority: P1)

As an operator, I can open the analytics tab, select a video file, and submit it so the system begins automated analysis.

**Why this priority**: Uploading and starting analysis is the core value of this feature. Without this flow, the tab has no functional purpose.

**Independent Test**: Can be fully tested by uploading a valid video, submitting it, and confirming the system creates an analysis job and starts processing.

**Acceptance Scenarios**:

1. **Given** a user is on the analytics tab and has a valid video file, **When** they submit the file, **Then** the system creates a new analysis job and shows that processing has started.
2. **Given** a user tries to submit without selecting a file, **When** they click upload, **Then** the system prevents submission and shows a clear validation message.
3. **Given** a user selects an unsupported file type, **When** they submit, **Then** the system rejects the file and tells the user which file formats are accepted.

---

### User Story 2 - Review Processed Video with Overlays (Priority: P1)

As an operator, I can play an uploaded video and see backend detections drawn on top so I can verify the analysis output without a burned-in export.

**Why this priority**: After upload, users need immediate visibility into analysis output to gain operational value from the feature.

**Independent Test**: Can be tested by completing one analysis job and confirming that results and overlays are viewable in the upload tab without navigating to other sections.

**Acceptance Scenarios**:

1. **Given** an analysis job is complete, **When** the user opens that job in the tab, **Then** the system displays the original video playback with backend-derived bounding boxes, labels, and tracking IDs drawn on top.
2. **Given** an analysis job is still running, **When** the user views the job, **Then** the system shows current status and does not present incomplete results as final.

---

### User Story 3 - View RTSP Camera Feed with Overlays (Priority: P1)

As an operator, I can open a live camera feed and see detections overlaid in real time so I can monitor the classroom while the stream is running.

**Why this priority**: RTSP monitoring is the second required input mode and must use the same overlay behavior as uploaded playback.

**Independent Test**: Can be tested by connecting a registered camera, receiving live detection packets, and confirming the frontend overlays boxes and tracking IDs above the video stream.

**Acceptance Scenarios**:

1. **Given** a registered RTSP camera is online, **When** the user opens the live view, **Then** the system streams video and overlays detections using the latest backend metadata.
2. **Given** the camera stream disconnects, **When** the worker reconnects successfully, **Then** the live overlays resume without requiring a page refresh.
3. **Given** the worker processes only every Nth frame, **When** the stream is running, **Then** the overlay still reflects the latest processed detections for that camera.

---

### User Story 4 - Control Bounding Box Visibility (Priority: P2)

As an operator, I can show or hide bounding boxes, labels, and model-specific overlays while viewing analyzed footage so I can inspect raw video and model output as needed.

**Why this priority**: Overlay controls improve usability and trust by letting users compare annotated and unannotated video.

**Independent Test**: Can be tested by opening analyzed output and toggling overlays on and off while confirming the video remains visible and playback continues.

**Acceptance Scenarios**:

1. **Given** analyzed video or live video is open with overlays available, **When** the user turns bounding boxes off, **Then** the video displays without bounding boxes.
2. **Given** analyzed video or live video is open with overlays hidden, **When** the user turns bounding boxes on, **Then** the video displays with bounding boxes again.

---

### User Story 5 - Recover From Failures and Partial Results (Priority: P3)

As an operator, I can still review available results when a job fails partially so I do not lose useful detections or have to restart from scratch.

**Why this priority**: Production workflows need partial visibility when some pipeline stages fail.

**Independent Test**: Can be tested by forcing a job failure in one component and confirming the UI shows the available frames, detections, and failure details.

**Acceptance Scenarios**:

1. **Given** an uploaded-video job ends in partial completion, **When** the user opens it, **Then** the system shows usable frames and the stages that failed.
2. **Given** a live camera reconnects after an interruption, **When** the stream recovers, **Then** the UI continues from the latest processed metadata.

### Edge Cases

- Upload is interrupted before completion because the user navigates away or loses connectivity.
- Uploaded file exceeds the allowed size or fails video validation.
- Uploaded video has no detectable subjects, resulting in empty overlays but still valid playback.
- Analysis job fails due to corrupted or unreadable video content.
- Live camera disconnects and remains offline long enough to trigger unhealthy status.
- Worker processes a frame but the overlay packet arrives late or out of order.
- User attempts to open or toggle overlays for a job or camera that has no processed metadata yet.
- Multiple overlapping bounding boxes for the same student or region must keep a consistent color and tracking ID across all enabled models.
- User toggles some models on or off while viewing: disabled model overlays disappear immediately and enabled overlays remain aligned with the same video frame.
- The frontend receives detections without a matching video frame at the current timestamp; stale overlays must clear rather than drift.

## Shared Data Contract

All backend inference services MUST emit a frame-level detection packet with full-frame coordinates and stable tracking IDs.

```json
{
  "frame_index": 123,
  "timestamp_ms": 4100,
  "camera_id": "cam_1",
  "detections": [
    {
      "id": "track_7",
      "parent_id": "vehicle_3",
      "source_model": "model_3",
      "label": "license_plate",
      "confidence": 0.94,
      "bbox": {
        "x1": 100,
        "y1": 200,
        "x2": 180,
        "y2": 240
      }
    }
  ]
}
```

- `frame_index` is required for uploaded-video playback synchronization.
- `timestamp_ms` is required for both live RTSP feeds and uploaded videos.
- `camera_id` identifies the live source; uploaded-video results may wrap the same frame packets in a video-results response keyed by the video/job identifier.
- `bbox` coordinates MUST be expressed in the original full-frame coordinate space after remapping.
- `id` MUST stay stable across RTSP frames for the same tracked object.
- `parent_id` is optional and can be used to preserve ROI lineage from the primary detector to secondary detections.

Uploaded-video results MAY be exposed as a playback bundle:

```json
{
  "video_id": "abc123",
  "video_url": "https://example.invalid/videos/abc123.mp4",
  "fps": 30,
  "duration_ms": 20000,
  "detections": {
    "0": [],
    "1": [],
    "2": []
  }
}
```

The frontend MUST overlay boxes itself whenever possible and treat optional annotated-video generation as a background enhancement, not the primary user experience.

## Inference Pipeline

The backend inference service MUST follow a five-stage pipeline for every frame:

1. **Primary Detector** - run the full-frame detector and produce regions of interest.
2. **ROI Extraction** - crop each ROI from the source frame while preserving the offset back to original coordinates.
3. **Secondary Models** - run models 2-5 on the crops in parallel where capacity allows, batching crops from the same frame when possible.
4. **Coordinate Remapping** - convert crop-relative detections back into full-frame coordinates.
5. **Aggregation** - merge the primary and secondary detections with tracking IDs, labels, and parent-child relationships when available.

## Requirements *(mandatory)*

### Functional Requirements

**FR-001**: System MUST provide a dedicated analytics surface that supports both uploaded video analysis and RTSP live camera monitoring.
**FR-002**: System MUST allow users to upload a valid video file from their local device and create a uniquely identifiable processing job.
**FR-003**: System MUST validate file presence, supported format, and size limits before accepting an upload.
**FR-004**: System MUST process uploaded videos asynchronously and expose frame-indexed detections, tracking IDs, and labels for playback review.
**FR-005**: System MUST render uploaded video in the browser with client-side overlays drawn from backend detection metadata rather than requiring a burned-in annotated export.
**FR-006**: System MUST expose `GET /videos/{id}/results` for completed uploads and return the video source, timing metadata, and detection packets needed for synchronized playback.
**FR-007**: System MUST support a camera registry for RTSP sources, including camera name, RTSP URL, status, and timestamps.
**FR-008**: System MUST ingest RTSP camera feeds continuously, process frames at a configurable cadence, and maintain tracker state per camera stream.
**FR-009**: System MUST reconnect dropped RTSP streams with exponential backoff and mark a camera unhealthy after repeated failures.
**FR-010**: System MUST broadcast live detection packets for each camera over a websocket channel at `/ws/camera/{camera_id}`.
**FR-011**: System MUST synchronize frontend overlays with the current playback frame or live timestamp and clear stale overlays when no matching detections are available.
**FR-012**: System MUST preserve partial results and component-level failure details when a job reaches `completed_partial` so the user can still review available outputs.
**FR-013**: System MUST allow users to show or hide bounding boxes, labels, model-specific detections, and tracking IDs without interrupting playback.
**FR-014**: System MUST assign each tracked subject a unique, consistent color within a job or camera session so no two subjects share a color and all enabled model overlays for that subject remain visually linked.
**FR-015**: System MUST batch crops per secondary model and run the secondary models in parallel when capacity allows to keep the inference path production-ready.
**FR-016**: System MUST display user-friendly error messages when upload, live ingestion, or analysis fails.
**FR-017**: System MUST allow users to retry failed uploads or recoverable processing failures without leaving the analytics surface.
**FR-018**: System MUST restrict the feature to authenticated teacher or admin users.

### Non-Functional Requirements

- Uploaded videos and extracted frames MUST be stored on the local server filesystem under `/data/videos/` with a 30-day retention lifecycle.
- Processing logs MUST be structured JSON and include frame indices, timestamps, detection counts, tracking IDs, labels, and source model names.
- The system MUST expose observability metrics for frame processing rate, embedding generation latency, and live-stream recovery time.
- Video processing jobs MUST time out after 5 minutes and retry failed work up to 2 times with exponential backoff.
- RTSP workers MUST support configurable FPS throttling and avoid unnecessary CPU/GPU copies when batching crops is feasible.
- Overlay visibility changes MUST be reflected in the viewer within 1 second for 95% of toggle actions.
- Annotated video generation, if enabled, MUST run as an optional background job and MUST NOT replace client-side overlays as the primary review flow.

### Key Entities *(include if feature involves data)*

- **Camera**: Represents a registered RTSP source, its connection status, and health metadata.
- **Video Analysis Job**: Represents one uploaded video analysis request; includes job identifier, upload metadata, current processing status, timestamps, and output availability state.
- **Detection Packet**: Represents one frame-level inference payload shared by live streams and uploaded playback.
- **Frame Result**: Represents one uploaded-video frame and its associated bounding boxes and metadata.
- **Student Track**: Represents a persistent tracked subject with a stable tracking ID and a unique color within the job or camera session.
- **Overlay Visibility Preference**: Represents the current user-selected display mode for bounding boxes and labels.

### Dependencies

- Existing authentication and role management can restrict access to teacher/admin users.
- Existing video storage and frame extraction can persist upload artifacts on the server filesystem.
- Existing websocket infrastructure can carry live detection packets for RTSP monitoring.
- The frontend can render the original video or stream source and draw transparent overlays above it.
- The inference subsystem can expose a results API for uploaded videos and a websocket channel for live feeds.

### Assumptions

- The analytics surface uses the same overall access and navigation model as other monitoring sections.
- Bounding box visibility control is session-scoped per viewed job or camera and defaults to visible when first opening analyzed output.
- The frontend overlays detections itself whenever possible; optional annotated video generation is a background enhancement only.
- A reasonable maximum upload size and supported video formats are defined by product policy outside this specification.
- RTSP tracker state is maintained per camera stream and may reset on stream restart if needed.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 95% of valid video uploads begin analysis processing within 10 seconds of submission.
- **SC-002**: 95% of users can complete the upload-and-review flow in under 2 minutes on first attempt.
- **SC-003**: 100% of completed analysis jobs are viewable from the analytics surface without requiring users to switch to another section.
- **SC-004**: 95% of live RTSP reconnect attempts restore overlay updates within 15 seconds under normal network conditions.
- **SC-005**: Bounding box visibility toggles reflect in the viewer within 1 second for 95% of toggle actions.
- **SC-006**: 100% of emitted detection packets include full-frame coordinates and the timestamp or frame index needed for synchronization.
- **SC-007**: At least 90% of users in acceptance testing report they can clearly distinguish between live overlays, uploaded playback overlays, and raw video states.

---

## Clarifications *(resolves ambiguities from planning phase)*

### Session 2026-05-01

Clarification notes for the combined analytics scope:

- Uploaded artifacts and extracted frames remain on the local filesystem under `/data/videos/` with a 30-day retention lifecycle.
- Structured JSON logging and operational metrics are required for frame throughput, embedding latency, and stream recovery.
- Jobs continue to use a strict 5-minute timeout with up to 2 retries and exponential backoff.
- Partial completion is a supported outcome and must preserve usable detections plus component-level failure details.
- Access remains limited to authenticated teacher/admin users.
- The frontend remains overlay-first; annotated video generation is optional and must not replace client-side overlays.
