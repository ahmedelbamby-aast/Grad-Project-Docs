# Implementation Plan: Unified Video Analytics Overlay Pipeline

**Branch**: `004-video-upload-inference-tab` | **Date**: 2026-04-30 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification + tracking/embedding research from `/specs/004-video-upload-inference-tab/`

**Note**: This plan extends the spec with multi-algorithm tracking, embedding-based re-identification, unified upload/RTSP processing, client-side overlays, and TDD methodology.

## Summary

**Primary Requirement**: Enable users to upload .mp4 videos and monitor RTSP camera feeds, automatically process both sources through a shared five-stage inference pipeline, and visualize backend detections with client-side overlays, stable tracking IDs, and per-subject colors. The upload flow must expose synchronized playback results, and the RTSP flow must broadcast live detection packets over WebSocket.

**Technical Approach**:
- Shared pipeline components are used for both upload playback and RTSP monitoring.
- The inference service follows a five-stage flow: detect, crop, run secondary models, remap coordinates, aggregate results.
- Tracking IDs are stable within each job or camera session, and every tracked subject receives one consistent color across all enabled overlays.
- Uploaded videos are processed asynchronously and exposed through `GET /videos/{id}/results` plus job status APIs.
- RTSP workers ingest frames continuously, throttle processing as needed, reconnect with exponential backoff, and broadcast detections over `/ws/camera/{camera_id}`.
- Test-first implementation with real video data; 80% coverage minimum.
- REST API for upload/status/results; WebSocket for live RTSP detections and job updates.

## Technical Context

**Language/Version**: Python 3.13 (backend), TypeScript ~5.9 (frontend)  
**Primary Dependencies**: Django 5.1.5, DRF 3.15.2, Celery 5.4.0, ultralytics 8.3.61 (YOLO detector with ByteTrack and BoT-SORT tracker support), Pydantic 2.10.6, React 19.2, Zustand 5, Vite 7.3  
**Storage**: PostgreSQL (job metadata, camera registry, embedding history), Redis (visibility state, job status, WebSocket pub-sub for live updates, temporary embeddings)  
**Testing**: pytest (backend), Vitest + Playwright (frontend), real video data from `Raw Data/` folders  
**Target Platform**: Linux server (backend processing); browser-based frontend  
**Project Type**: Web service (Django + React) with AI inference pipeline  
**Performance Goals**: 
  - Upload-to-processing-start: < 10 seconds
  - Frame processing: < 50ms per frame (including detection + tracking + embedding)
  - Embedding generation: < 200ms per student crop
  - Re-ID similarity scoring: < 10ms per comparison
  - Overlay toggle response: < 500ms from user action to visual update
**Constraints**: 
  - Support videos up to 2 hours; split large videos into chunks for processing
  - RTSP worker FPS throttling should be configurable per camera
  - Maintain tracking ID consistency across occlusions (< 3 seconds) and re-entries (< 5 minutes)
  - Embedding similarity threshold: 0.85 for re-ID confidence
  - Redis TTL: 1 hour for temporary embeddings; keep long-term in PostgreSQL
  - Job timeout: 5 minutes maximum per video (per Q3 clarification)
  - Retry strategy: Up to 2 retries with exponential backoff on timeout (per Q3 clarification)
  - File storage: Local filesystem at `/data/videos/` with 30-day retention (per Q1 clarification)
  - Partial failures: Allowed; job status = `completed_partial` with component-level details (per Q4 clarification)
**Scale/Scope**: 
  - Single classroom environment (up to 50 students + 3 teachers)
  - Concurrent uploads: 5-10 videos simultaneous processing
  - Concurrent RTSP cameras: up to 8 active feeds per deployment
  - Video retention: 30 days of analysis data; after 30 days, videos archived and embeddings moved to cold storage

**CLARIFICATION NOTES**: 5 clarification questions resolved 2026-05-01 (see spec.md § Clarifications):
1. ✅ File Storage Strategy → Local Filesystem (`/data/videos/`)
2. ✅ Observability Strategy → Comprehensive (frame-level logs, metrics to Prometheus)
3. ✅ Reliability SLA → Strict (5min timeout, 2 retries, exponential backoff)
4. ✅ Partial Failure Handling → Continue with Partial Results (job status: completed_partial)
5. ✅ Auth & Authz → Role-Based with Global Visibility (all authenticated staff see all jobs)

## Constitution Check

*GATE: Verify compliance with project constitution before proceeding. This is BLOCKING.*

> ✅ **Supreme Directive Gate**: PASS
> This plan acknowledges and commits to:
> 1. Commit immediately after every modification (create/update/fix/delete).
> 2. Create/update all affected .md documentation and docs/ mirror files.
> 3. Include ALL mandatory diagram types (flowchart, sequence, class, state, ER).
> 4. Provide detailed explanations for every diagram (intro + walkthrough).
> 5. Cross-link all .md files via relative Markdown links.
> Enforcement: Agents MUST verify 5-point checklist after each modification before proceeding.

> ✅ **Test-in-Loop Gate**: PASS
> This plan enforces TDD:
> 1. Write unit tests (embedding generation, color assignment, similarity scoring).
> 2. Write integration tests (tracker + detector, re-ID pipeline).
> 3. Write system tests (upload + processing + playback with overlay control).
> 4. Use real video data from Raw Data/ folders; no mocks-only tests.
> 5. Achieve >= 80% code coverage before declaring feature complete.
> Enforcement: No implementation code is written until tests exist and fail.

## Project Structure

### Documentation (this feature)

```text
specs/004-video-upload-inference-tab/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command) — COMPLETED
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
│   ├── upload.md        # REST API for video file upload
│   ├── status.md        # WebSocket + polling for job status
│   └── overlay.md       # REST PATCH for model visibility toggles
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

```text
backend/
├── apps/
│   ├── video_analysis/          # NEW: Video upload & job management
│   │   ├── models.py            # VideoAnalysisJob, FrameEmbedding, TrackingMetadata
│   │   ├── views.py             # REST endpoints for upload, status, visibility toggle
│   │   ├── serializers.py       # DRF serializers for API contracts
│   │   └── tasks.py             # Celery tasks for video processing, embedding generation
│   └── tracking/                # NEW: Tracking & re-ID logic
│       ├── tracker.py           # Ultralytics tracker wrapper, ByteTrack/BoT-SORT selection
│       ├── embeddings.py        # Embedding generation, Redis/DB storage
│       ├── reid.py              # Re-ID via similarity scoring
│       └── rendering.py         # Bounding box rendering with colors & IDs
├── tests/
│   ├── unit/
│   │   ├── test_embeddings.py
│   │   ├── test_reid.py
│   │   ├── test_rendering.py
│   │   └── test_tracking.py
│   ├── integration/
│   │   ├── test_tracker_detector.py
│   │   └── test_pipeline.py
│   └── system/
│       └── test_upload_to_playback.py
└── docs/
    └── video_analysis_pipeline.md  # Architecture and data flow

frontend/
├── src/
│   ├── components/
│   │   ├── VideoUploadTab.tsx       # NEW: Upload form & job list
│   │   ├── VideoPlayer.tsx          # NEW: Playback with overlay controls
│   │   └── ModelVisibilityToggles.tsx # NEW: Model selector UI
│   ├── pages/
│   │   └── VideoAnalysisPage.tsx    # NEW: Container for upload tab & player
│   ├── services/
│   │   └── videoAnalysisAPI.ts      # REST + WebSocket client
│   ├── stores/
│   │   └── videoAnalysisStore.ts    # Zustand: job state, visibility settings
│   ├── types/
│   │   └── videoAnalysis.ts         # TypeScript types for job, frame, tracking data
│   └── tests/
│       ├── integration/
│       │   └── upload-and-playback.spec.ts
│       └── unit/
│           └── videoAnalysisStore.spec.ts
```

**Structure Decision**: 
This feature extends the existing Django + React architecture. Backend adds two new Django apps: `video_analysis` (REST endpoints, camera registry, Celery tasks) and `tracking` (algorithm implementations, re-ID logic). Frontend adds a unified analytics surface with upload playback, RTSP live monitoring, and shared model visibility toggles. All documentation mirrors source structure per constitution §docs/. Test structure follows backend/frontend separation with unit → integration → system progression.

## Complexity Tracking

> No constitution violations detected. All complexity (multi-algorithm tracking, embedding generation, re-ID) is well-justified by feature requirements and does not introduce architectural debt.

---

## Next Steps (Phase 1 Deliverables)

1. **data-model.md**: Define all entities (Camera, VideoAnalysisJob, Frame, DetectionPacket, StudentTrack, FrameEmbedding, BoundingBox) with relationships.
2. **contracts/upload.md**: REST POST contract for file upload and `GET /videos/{id}/results` playback results.
3. **contracts/status.md**: WebSocket contract and polling endpoint for job status, progress, and error tracking.
4. **contracts/overlay.md**: REST PATCH contract for toggling model visibility; response format.
5. **quickstart.md**: End-to-end developer guide: upload a video, monitor an RTSP feed, toggle overlays, and view synchronized playback.
6. **Update CLAUDE.md**: Add video_analysis and tracking as active technologies.
7. **Agent context update**: Run `.specify/scripts/powershell/update-agent-context.ps1` to sync plan with Copilot context.
8. **Re-evaluate Constitution Check**: Verify all diagrams, cross-links, and documentation standards are met.
