# Tasks: Unified Video Analytics Overlay Pipeline (004-video-upload-inference-tab)

**Input**: Design documents from `/specs/004-video-upload-inference-tab/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/
**Tests**: Tests are MANDATORY per the Test-in-Loop methodology (Constitution §II). ALL tests (unit, integration, system) MUST be written BEFORE implementation for every user story. Tests MUST be verified to fail (RED) before any implementation code is written.
**Organization**: Tasks are grouped by user story so each increment can be implemented, tested, and delivered independently. The pipeline must support both the legacy crop-based architecture and the new full-frame multi-model mode without deleting the old path.

## Format: `[ID] [P?] [Story] Description`

- **[ID]**: Sequential task identifier (T001, T002, etc.)
- **[P]**: Task can run in parallel (different files, no blocking dependencies)
- **[Story]**: Which user story the task belongs to (US1, US2, US3, US4, US5)
- Include exact file paths in descriptions

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure for the unified upload + RTSP analytics surface

- [x] T001 [P] Create backend scaffolding for pipeline mode and export helpers in `backend/apps/tracking/pipeline_mode.py`, `backend/apps/tracking/video_exporter.py`, and `backend/apps/video_analysis/constants.py`.
- [x] T002 [P] Create frontend scaffolding for analytics UI in `frontend/src/pages/VideoAnalysisPage.tsx`, `frontend/src/components/VideoUploadTab/VideoUploadTab.tsx`, `frontend/src/components/VideoPlayer/VideoPlayer.tsx`, `frontend/src/components/ModelVisibilityToggles/ModelVisibilityToggles.tsx`, and `frontend/src/components/OverlayModeSelector/OverlayModeSelector.tsx`.
- [x] T003 [P] Create test scaffolding in `backend/tests/unit/tracking/`, `backend/tests/contract/`, `backend/tests/integration/`, `backend/tests/system/`, and `frontend/tests/` for upload, playback, live, visibility, and failure flows.
- [x] T004 [P] Wire base app registration and routing stubs in `backend/config/settings/base.py`, `backend/config/urls.py`, and `backend/config/asgi.py` for video_analysis and tracking apps.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**CRITICAL**: No user story implementation begins until this phase is complete

- [x] T005 [P] Add core data models and migrations in `backend/apps/video_analysis/models.py` and `backend/apps/video_analysis/migrations/0001_initial.py` for VideoAnalysisJob, Frame, DetectionPacket, StudentTrack, BoundingBox with `pipeline_mode` and `annotated_video_path` fields.
- [x] T006 Add serializer foundations in `backend/apps/video_analysis/serializers.py` for job status, results, detection packets, and visibility payloads.
- [x] T007 [P] Add pipeline mode constants and defaults in `backend/apps/tracking/pipeline_mode.py` and `backend/config/settings/base.py` (default to legacy crop pipeline, keep old architecture intact).
- [x] T008 Implement pipeline executor with legacy crop mode and new full-frame multi-model mode in `backend/apps/tracking/pipeline.py`, ensuring only the primary detector runs tracking and secondary models run on full frames without tracking.
- [x] T009 [P] Implement association helper to link secondary model detections to primary tracks without trackers in `backend/apps/tracking/association.py` (IoU or centroid matching).
- [x] T010 Rename pipeline mode labels and refine secondary-model association thresholds in `backend/apps/tracking/pipeline_mode.py`, `backend/apps/tracking/association.py`, and `frontend/src/types/videoAnalysis.ts`.
- [x] T011 [P] Implement detection packet normalization and overlay helpers in `backend/apps/tracking/packets.py` and `backend/apps/tracking/rendering.py` so both pipeline modes emit the shared schema.
- [x] T012 Add unit tests for pipeline mode selection and tracking-only-primary behavior in `backend/tests/unit/tracking/test_pipeline_mode.py`.
- [x] T013 Add unit tests for association and packet normalization in `backend/tests/unit/tracking/test_association.py` and `backend/tests/unit/tracking/test_packets.py`.
- [x] T014 [P] Extend frontend domain types and API client scaffolding for `pipeline_mode`, `annotated_video_url`, and `overlay_mode` in `frontend/src/types/videoAnalysis.ts` and `frontend/src/api/videoAnalysis.ts`.

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - Upload Video for Analysis (Priority: P1)

**Goal**: Users can upload `.mp4` files, choose a pipeline mode, and trigger the automated analysis pipeline

**Independent Test**: Upload a valid video from `Raw Data/`, confirm the system creates a job with the selected pipeline mode, starts processing, and persists the uploaded file under `/data/videos/`

### Tests for User Story 1 (MANDATORY - RED first)

- [x] T015 [P] [US1] Contract test for POST `/api/v1/video-analysis/jobs/` in `backend/tests/contract/test_upload.py`: verify success, validation failures, and pipeline_mode persistence.
- [x] T016 [P] [US1] Unit tests for upload validation and metadata extraction in `backend/tests/unit/video_analysis/test_upload_validation.py`: file presence, `.mp4` only, size, corruption, fps, duration, pipeline_mode default.
- [x] T017 [P] [US1] Integration test for upload-to-job lifecycle in `backend/tests/integration/test_upload_workflow.py`: real video persists under `/data/videos/`, creates job, and starts processing.

### Implementation for User Story 1

- [x] T018 [US1] Implement upload validation and job creation with pipeline_mode in `backend/apps/video_analysis/views.py` (default to legacy crop pipeline if omitted).
- [x] T019 [US1] Implement `process_video_upload()` routing to the pipeline executor in `backend/apps/video_analysis/tasks.py` based on job.pipeline_mode.
- [x] T020 [P] [US1] Update upload UI to select pipeline mode in `frontend/src/components/VideoUploadTab/VideoUploadTab.tsx` and persist selection in `frontend/src/stores/uploadStore.ts`.
- [x] T021 [P] [US1] Update upload API wiring in `frontend/src/api/videoAnalysis.ts` and `frontend/src/pages/VideoAnalysisPage.tsx` to send pipeline_mode and show queued/processing state.

**Checkpoint**: User Story 1 should be independently functional

---

## Phase 4: User Story 2 - Review Processed Video with Overlays (Priority: P1)

**Goal**: Users can play an uploaded video with client-side overlays and optionally view/download an annotated export with bounding boxes and tracking IDs

**Independent Test**: Complete one analysis job, fetch synchronized results, verify client overlays render correctly, and confirm an annotated export is available when requested

### Tests for User Story 2 (MANDATORY - RED first)

- [x] T022 [P] [US2] Contract test for `GET /api/v1/videos/{id}/results` in `backend/tests/contract/test_results.py`: verify `video_url`, `fps`, `duration_ms`, `detections`, `pipeline_mode`, and optional `annotated_video_url`.
- [x] T023 [P] [US2] Integration test for playback synchronization in `backend/tests/integration/test_playback_sync.py`: frame index and timestamp mapping stay aligned across pipeline modes.
- [x] T024 [P] [US2] Integration test for annotated export generation in `backend/tests/integration/test_annotated_export.py`: ensure exported video includes boxes and tracking IDs.
- [x] T025 [P] [US2] System test for uploaded-video overlay playback in `backend/tests/system/test_upload_to_playback.py`: upload real video, fetch results, and confirm overlays render on top of playback.

### Implementation for User Story 2

- [x] T026 [US2] Implement playback results endpoint in `backend/apps/video_analysis/views.py` to return `GET /api/v1/videos/{id}/results` with synchronized metadata and detection packets.
- [x] T027 [US2] Implement playback bundle assembly in `backend/apps/video_analysis/serializers.py` to emit frame-indexed detections with stable tracking IDs, labels, and parent links.
- [x] T028 [P] [US2] Implement annotated video export in `backend/apps/tracking/video_exporter.py` and integrate in `backend/apps/video_analysis/tasks.py` to store `annotated_video_path` for completed jobs.
- [x] T029 [US2] Expose `annotated_video_url` and `pipeline_mode` in `backend/apps/video_analysis/models.py` and `backend/apps/video_analysis/serializers.py`.
- [x] T030 [US2] Add explicit annotated-export download endpoint in `backend/apps/video_analysis/views.py` and `backend/apps/video_analysis/urls.py`, returning a download link in `backend/apps/video_analysis/serializers.py`.
- [x] T031 [P] [US2] Update playback UI to support client overlays and an annotated-video mode selector in `frontend/src/components/VideoPlayer/VideoPlayer.tsx` and `frontend/src/components/OverlayModeSelector/OverlayModeSelector.tsx`.
- [x] T032 [P] [US2] Update frontend state and API integration for results, annotated video, and overlay mode in `frontend/src/stores/videoAnalysisStore.ts` and `frontend/src/api/videoAnalysis.ts`.
- [x] T033 [P] [US2] Fix overlay canvas sizing to match the video element in `frontend/src/components/VideoPlayer/VideoPlayer.tsx` and `frontend/src/components/VideoPlayer/OverlayCanvas.tsx`.

**Checkpoint**: User Story 2 should be independently functional

---

## Phase 5: User Story 3 - View RTSP Camera Feed with Overlays (Priority: P1)

**Goal**: Users can open a live camera feed and see detections overlaid in real time

**Independent Test**: Connect to a registered camera, receive live detection packets, and confirm the frontend overlays boxes and tracking IDs above the stream

### Tests for User Story 3 (MANDATORY - RED first)

- [x] T034 [P] [US3] Contract test for `/ws/camera/{camera_id}` in `backend/tests/contract/test_camera_websocket.py`: verify live detection packet schema, timestamp, and stable tracking IDs.
- [x] T035 [P] [US3] Integration test for RTSP worker reconnect and throttling in `backend/tests/integration/test_rtsp_worker.py`: simulate disconnects, verify exponential backoff, health updates, and FPS throttling.
- [x] T036 [P] [US3] System test for live camera overlay in `backend/tests/system/test_rtsp_live_overlay.py`: connect to a seeded camera, receive packets, and confirm stale overlays clear correctly.

### Implementation for User Story 3

- [x] T037 [US3] Implement camera registry persistence and API in `backend/apps/video_analysis/models.py`, `backend/apps/video_analysis/views.py`, and `backend/apps/video_analysis/serializers.py` for registering RTSP sources, status, and timestamps.
- [x] T038 [US3] Implement RTSP ingest worker in `backend/apps/tracking/rtsp_worker.py` (or `backend/apps/video_analysis/tasks.py`) to decode frames, throttle FPS, reconnect with exponential backoff, and emit detection packets.
- [x] T039 [US3] Implement live websocket broadcaster in `backend/apps/video_analysis/consumers.py` and `backend/config/asgi.py` to publish packets to `/ws/camera/{camera_id}`.
- [x] T040 [P] [US3] Update `frontend/src/components/camera/CameraFeed.tsx`, `frontend/src/components/camera/BoundingBoxCanvas.tsx`, and `frontend/src/pages/VideoAnalysisPage.tsx` to render the RTSP stream with live overlay packets.

**Checkpoint**: User Story 3 should be independently functional

---

## Phase 6: User Story 4 - Control Bounding Box Visibility (Priority: P2)

**Goal**: Users can show or hide bounding boxes, labels, and model-specific overlays while viewing analyzed footage or live RTSP streams

**Independent Test**: Toggle each model on and off in both playback and live views, verify the video/stream remains visible, and confirm the overlays update quickly with colors and tracking IDs preserved

### Tests for User Story 4 (MANDATORY - RED first)

- [x] T041 [P] [US4] Contract test for PATCH visibility in `backend/tests/contract/test_visibility.py`: verify request validation and response state.
- [x] T042 [P] [US4] Integration test for overlay filtering in `backend/tests/integration/test_visibility_toggle.py`: verify both playback and live overlays respect the same visibility payload.
- [x] T043 [P] [US4] System test for toggle responsiveness in `backend/tests/system/test_toggle_response_time.py`: confirm visibility updates are reflected within 1 second.

### Implementation for User Story 4

- [x] T044 [US4] Implement shared visibility persistence in `backend/apps/video_analysis/views.py` so playback and live packets use the same per-session visibility payload.
- [x] T045 [P] [US4] Update frame and packet filtering in `backend/apps/video_analysis/views.py` and `backend/apps/tracking/rendering.py` to omit disabled model overlays while preserving color and tracking ID stability.
- [x] T046 [P] [US4] Update `frontend/src/stores/visibilityStore.ts` and `frontend/src/components/ModelVisibilityToggles/ModelVisibilityToggles.tsx` to drive one visibility state across uploaded playback and RTSP feeds.
- [x] T047 [P] [US4] Update `frontend/src/components/VideoPlayer/VideoPlayer.tsx` and `frontend/src/components/camera/BoundingBoxCanvas.tsx` to re-render overlays on visibility changes and disable toggles when annotated-video mode is active.

**Checkpoint**: User Story 4 should be independently functional

---

## Phase 7: User Story 5 - Recover From Failures and Partial Results (Priority: P3)

**Goal**: Users can still review available results when a job fails partially and can retry from the same analytics surface

**Independent Test**: Force a component failure, confirm usable frames and detections remain available, then retry the job without leaving the page

### Tests for User Story 5 (MANDATORY - RED first)

- [x] T048 [P] [US5] Contract test for partial completion and failure details in `backend/tests/contract/test_status.py`: verify `completed_partial` and `component_failures` appear in polling and websocket messages.
- [x] T049 [P] [US5] Integration test for partial-result preservation in `backend/tests/integration/test_partial_results.py`: simulate a stage failure and ensure successful frames and detections remain available.
- [x] T050 [P] [US5] System test for retry and recovery flow in `backend/tests/system/test_failure_recovery.py`: verify failed jobs can be retried without leaving the analytics surface.

### Implementation for User Story 5

- [x] T051 [US5] Extend `backend/apps/video_analysis/tasks.py` to mark partial failures, preserve successful frames, and populate `component_failures` for each pipeline stage.
- [x] T052 [P] [US5] Extend `backend/config/celery.py` and `backend/apps/video_analysis/tasks.py` with 5-minute time limits, two retries, and exponential backoff for recoverable failures.
- [x] T053 [US5] Update `backend/apps/video_analysis/views.py` and `backend/apps/video_analysis/serializers.py` to return failure details and retryable status data.
- [x] T054 [P] [US5] Add a retry action in `backend/apps/video_analysis/urls.py`, `backend/apps/video_analysis/views.py`, and `frontend/src/components/VideoUploadTab/VideoUploadTab.tsx` so recoverable jobs can be requeued from the analytics surface.

**Checkpoint**: User Story 5 should be independently functional

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories and final readiness

- [x] T055 [P] Update `docs/backend/api/README.md`, `docs/backend/apps/video_analysis/README.md`, `docs/backend/apps/tracking/README.md`, `docs/frontend/src/components/README.md`, and `docs/ARCHITECTURE.md` for dual pipeline modes and annotated export behavior.
- [x] T056 [P] Regenerate or validate Mermaid diagrams and cross-links in `docs/backend/apps/video_analysis/data-model-diagram.md`, `docs/backend/apps/tracking/class-diagram.md`, `docs/backend/VIDEO_UPLOAD_SEQUENCE.md`, `docs/backend/JOB_STATE_DIAGRAM.md`, `docs/INDEX.md`, and `specs/004-video-upload-inference-tab/quickstart.md`.
- [x] T057 [P] Run backend/frontend test and coverage verification in `backend/tests/` and `frontend/tests/`, then validate `specs/004-video-upload-inference-tab/quickstart.md` against the implemented flows.
- [x] T058 [P] Perform security and performance hardening in `backend/apps/video_analysis/views.py`, `backend/apps/tracking/rtsp_worker.py`, and `frontend/src/components/VideoPlayer/VideoPlayer.tsx` for upload validation, RTSP reconnects, and overlay toggle latency.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: No dependencies - can start immediately.
- **Phase 2 (Foundational)**: Depends on Phase 1 - blocks all user stories.
- **User Story Phases (Phase 3+)**: Depend on Phase 2 - each story remains independently testable.
- **Phase 8 (Polish)**: Depends on desired user stories being complete.

### User Story Dependencies

- **US1 (Upload)**: Starts after the foundational phase; establishes job lifecycle and pipeline_mode selection.
- **US2 (Playback)**: Starts after the foundational phase and builds on US1 job outputs.
- **US3 (RTSP Live)**: Starts after the foundational phase; can run in parallel with US2.
- **US4 (Visibility)**: Best scheduled after US2/US3 so both viewers exist, but remains independently testable.
- **US5 (Partial Results)**: Extends US1 job lifecycle and can start after foundational phase.

### Dependency Graph

Setup -> Foundational -> US1 -> (US2, US3) -> US4 -> US5 -> Polish

---

## Parallel Execution Examples

### User Story 1

T015, T016, T017 can run in parallel (tests).
T020 and T021 can run in parallel (frontend files).

### User Story 2

T022, T023, T024, T025 can run in parallel (tests).
T031 and T032 can run in parallel (frontend files).

### User Story 3

T034, T035, T036 can run in parallel (tests).
T037 and T040 can run in parallel (backend vs frontend).

### User Story 4

T041, T042, T043 can run in parallel (tests).
T045, T046, T047 can run in parallel (rendering vs UI).

### User Story 5

T048, T049, T050 can run in parallel (tests).
T052 and T054 can run in parallel (backend vs frontend).

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup.
2. Complete Phase 2: Foundational (blocks all stories).
3. Complete Phase 3: User Story 1.
4. Validate User Story 1 independently.

### Incremental Delivery

1. Setup + Foundational -> foundation ready.
2. US1 -> test independently -> demo.
3. US2 + US3 -> test independently -> demo.
4. US4 -> test independently -> demo.
5. US5 -> test independently -> demo.

### Parallel Team Strategy

Once Phase 2 completes:
1. Developer A: US1
2. Developer B: US2
3. Developer C: US3
4. US4 and US5 follow when base views exist

### Within Each User Story

- Tests MUST be written and fail before implementation.
- Backend contracts and data assembly before frontend rendering.
- Services and workers before UI polish.
- Complete one story before moving to the next priority when working sequentially.

### Parallel Opportunities
