# Research: Video Upload & Tracking Integration

**Feature**: 004-video-upload-inference-tab  
**Date**: 2026-04-30  
**Status**: Research Phase 0 output  

## Executive Summary

This research documents technical decisions for integrating multi-algorithm tracking, embedding-based re-identification, and test-first implementation into the video upload feature. The feature is upload-first; RTSP/live-stream language is treated as shared pipeline reuse rather than separate acceptance criteria. It applies tracking only to the first detection layer (student/teacher), manages model output visualization, and achieves 80%+ test coverage using real video data.

---

## R1: Tracking Algorithm Selection

### Decision: Use Ultralytics Built-In Trackers with ByteTrack as Default

**Selected Approach**: Use the trackers that Ultralytics documents as built-in today: **ByteTrack** and **BoT-SORT**. ByteTrack is the lighter default path; BoT-SORT is the ReID-enabled option. Custom third-party trackers such as StrongSORT are not part of the built-in Ultralytics track page and should only be treated as future/custom integration work.

**Rationale**:
- The Ultralytics track docs explicitly list **ByteTrack** and **BoT-SORT** as available trackers.
- The default tracker in the docs is **BoT-SORT**, while ByteTrack is selected by passing `tracker="bytetrack.yaml"`.
- `with_reid` is documented as supported only by **BoT-SORT**, which makes it the tracker to use when appearance-based matching is needed.
- A custom YAML tracker file can be passed through `tracker="custom_tracker.yaml"` if tracker parameters need to be tuned without changing code.

**Alternatives Considered**:
- Single fixed tracker: Rejected because the feature needs a documented default plus a second supported option.
- StrongSORT as a built-in option: Rejected because it is not listed in the Ultralytics track docs page used for this research.

**Implementation Details**:
```python
from ultralytics import YOLO

model = YOLO("yolo26n.pt")

# Default tracker (BoT-SORT)
results = model.track(source="video.mp4", persist=True)

# Explicit ByteTrack selection
results = model.track(source="video.mp4", persist=True, tracker="bytetrack.yaml")

# Custom tracker YAML derived from an Ultralytics template
results = model.track(source="video.mp4", persist=True, tracker="custom_tracker.yaml")
```

**Docs Note**: The Ultralytics page also shows the longer frame loop pattern with `persist=True` and track history plotting, plus multithreaded tracking for multiple streams.

---

## R2: Tracking Scope—First Layer of Pyramid Only

### Decision: Apply Tracking to Detection Layer; Skip Cropped Behavioral Layers

**Selected Approach**: Track students in the full-frame detection layer (pyramid layer 0). Cropped images from layers 1+ are behavior-specific and do not retain scene context necessary for tracking.

**Rationale**:
- Detection layer operates on full video frames with spatial continuity.
- Cropped behavioral layers (standing/sitting, up/down, etc.) operate on isolated student ROIs without inter-frame reference.
- Tracking IDs and positions must be maintained at full-frame level to support re-identification and boundary box rendering.

- **Teachers are DETECTED (stored in Detection records) but NOT TRACKED**: The tracker pipeline is configured to focus on student-class detections. Teachers appear in detection output but are excluded from StudentTrack creation. This supports audit visibility while keeping tracking focused on student behavior.
**Data Flow**:
1. Full frame → YOLO detector (produces detections for students & teachers)
2. Apply tracker → Assign tracking IDs to student detections only
3. Crop student ROIs → Feed to behavior classifiers (standing/sitting, etc.)
4. Store tracking metadata → Use for re-identification and UI rendering

---

## R3: Tracking ID Persistence and Re-entry Handling

### Decision: Embedding-Based Re-Identification with Periodic Refresh

**Selected Approach**:
- Generate embeddings for each tracked student every **10 seconds**.
- Store embeddings in **Redis** (cache layer) with TTL for fast access; persist to PostgreSQL for history.
- Apply **cosine similarity** measure to detect re-entry: if a student exits frame for < 5 minutes and returns, match to existing ID via embedding similarity (threshold: 0.85).
- If similarity < 0.85, assign new tracking ID.

**Rationale**:
- **Continuous ID assignment**: Native Ultralytics tracking may lose ID after occlusion/exit; embeddings provide recovery mechanism.
- **10-second cadence**: Balances computational cost with re-identification accuracy.
- **Redis + PostgreSQL**: Redis for latency-sensitive queries (real-time playback), PostgreSQL for audit trail and batch re-identification jobs.
- **Cosine similarity (0.85 threshold)**: Industry standard for re-id; threshold tuned to minimize false matches while tolerating appearance variation.

**No New Embeddings After 10s**: If 10 seconds pass with no new detections, student is marked as **exited**. Tracker will not reassign ID to that student until re-entry is confirmed via embedding similarity.

---

## R4: Bounding Box Rendering—ID, Color, and Visibility Control

### Decision: Per-Student Color Palette with Model Selectors

**Selected Approach**:
- Assign each student a **unique, consistent color** across all frames and **all 5 models** (hash of student ID mod color palette size).
- Each student can have up to **5 bounding boxes overlaid in the same frame** (one per model: person detection, sitting/standing, attention tracking, hand raising, + future models).
- **ALL 5 boxes for the same student MUST render with identical color** to maintain visual clarity and student identification.
- Render **tracking ID as text overlay** on bounding boxes.
- Provide **model selector toggles** in UI to show/hide each model's detections independently.
- Support **all-off except one** scenario.

**Multi-Box Color Consistency Details**:
- When a student has detections from multiple models in the same frame, all bounding boxes use the same color.
- Example: Student #5 (assigned red color) → person detection box (red), sitting/standing box (red), attention box (red), hand-raise box (red).
- If user toggles a model off, that box disappears but does NOT change the color assignment; if toggled back on, the same red color reappears.
- Color assignment is **deterministic** and **student-scoped**: Same student ID always gets same color across all frames, jobs, and model outputs.

**Rationale**:
- **Consistent color**: Improves human-in-the-loop verification; users instantly recognize individuals by color across all overlays.
- **Multi-box clarity**: Without color consistency, 5 overlapping boxes would create visual confusion. Matching colors signal "same student".
- **ID overlay**: Auditable; users can verify tracking consistency across models.
- **Model toggles**: Support comparison of YOLO outputs (person detection) vs. behavioral classifiers (standing/sitting, etc.).
- **Independent control**: Allows users to isolate individual model outputs for debugging or analysis.

**UI Behavior**:
- Toggle buttons for each model (e.g., "Person Detection", "Sitting/Standing", "Attention Level", "Hand Raising").
- Default: All models enabled.
- Interaction: User clicks toggle → front-end sends command → back-end re-renders frame → front-end updates playback.
- Latency target: < 500ms from toggle to visual update.
- All visible boxes for the same student maintain their assigned color.

**Color Palette**:
- Minimum 30-color palette to support up to 30 concurrent students (classroom scale).
- Palette maximizes perceptual contrast via HSL-based distribution: fixed saturation (100%) and lightness (50%) with evenly-spaced hues. For 30 colors: hues at 0°, 12°, 24°, ..., 348° (360/30 = 12° spacing). This ensures each color is visually distinct from all others, critical for dense classrooms with many simultaneous students.
- **Unique per uploaded video**: No two students detected anywhere in the entire uploaded video share the same color. This constraint is enforced at database level via `UNIQUE(job_id, color_hex)` to guarantee unambiguous visual identification regardless of frame composition or temporal position.
- Color assignment algorithm: Deterministic hash of (job_id, tracking_id) mod palette size, ensuring reproducibility and consistency across reruns.

---

## R5: Video Upload Reuses Shared Pipeline Components

### Decision: Upload-First Scope with Shared Processing Components

**Selected Approach**:
- Uploaded .mp4 jobs reuse the same core processing components as the broader analysis pipeline, but this feature's acceptance criteria remain limited to uploaded videos.
- No distinct code paths are required inside this feature for upload vs. live; use common frame-processing utilities where the codebase already shares them.
- Video files are read frame-by-frame at the file's native FPS; tracking and inference apply identically within the upload flow.

**Rationale**:
- Simplifies backend logic; shared components mean fewer bugs and easier maintenance.
- Keeps the feature scoped to uploaded videos without blocking future live-session reuse.
- Re-uses existing RTSP pipeline infrastructure (Django Channels, WebSocket handlers, Celery tasks) where already available.

**Implementation**:
- Upload → Store in temp directory → Create Celery task → Process frames → Emit results via WebSocket for job progress and store results for playback.
- File I/O vs. network I/O abstracted by common frame reader interface.

---

## R6: Test-First Methodology with Real Video Data

### Decision: TDD with 80% Coverage Minimum; Real Video Validation

**Selected Approach**:
- **Test order**: Unit (models, utility functions) → Integration (tracker + detector) → System (end-to-end upload + analysis).
- **Real video data**: Use videos from `Raw Data/` folders (Arguing Students, Diverse Classrooms, etc.).
- **Test categories**:
  - Unit: Embedding generation, similarity scoring, color assignment.
  - Integration: Tracker persistence, re-ID logic, frame processing pipeline.
  - System: Upload → processing → playback with overlay control.
- **Coverage target**: >= 80% across backend + tracking logic.
- **No skipped tests**: All tests must pass with real assertions; no mock-only tests.

**Test Checklist** (write-before-code order):
1. ✓ Embedding generation produces fixed-size vectors for student crops.
2. ✓ Cosine similarity correctly identifies same student across frames.
3. ✓ Re-ID logic reassigns ID when similarity < 0.85.
4. ✓ Color assignment is deterministic (same student ID → same color).
5. ✓ ByteTrack tracker assigns IDs consistently within a video.
6. ✓ Tracking ID persists through 3-second occlusion.
7. ✓ Tracking ID resets after 5-minute absence.
8. ✓ Bounding box overlay renders with correct ID, color, and edges.
9. ✓ Model toggle command removes/adds overlays without video stutter.
10. ✓ Upload endpoint accepts .mp4, rejects unsupported formats.
11. ✓ Processing job completes within SLA (e.g., 5 min for 2 min video).
12. ✓ End-to-end: upload → process → playback with toggles works.

**Real Video Validation**: All tests above run against actual videos from Raw Data; minimum 3 videos per test scenario.

---

## R7: Frontend-Backend Integration

### Decision: WebSocket for Real-Time Status; REST for Bulk Operations

**Selected Approach**:
- **Upload**: REST POST to `/api/video-analysis/jobs/` with .mp4 file.
- **Job status**: WebSocket connection or polling endpoint for real-time updates.
- **Frame rendering**: Django REST Framework endpoint retrieves pre-computed and stored BoundingBox records, applies visibility filter from Redis cache, and returns filtered bounding boxes on-demand.
- **Overlay control**: REST PATCH to toggle model visibility (stored in Redis cache); subsequent frame fetches apply the current visibility filter.

**Rationale**:
- **REST for file upload**: Standard pattern; supports resumable uploads if needed.
- **WebSocket for real-time**: Low-latency job status updates and toggle responses; matches existing Session Section architecture.
- **Pre-computed overlays**: BoundingBox records are generated during processing (not on-demand) and stored; visibility filtering is applied at retrieval time from Redis cache, ensuring low-latency responses.

**Data Contract**:
- `POST /api/video-analysis/jobs/`: Submit .mp4 file → returns `{ job_id, status: 'queued' }`.
- `GET /api/video-analysis/jobs/{job_id}/frames/`: Fetch frame N with current overlay settings → returns image + metadata.
- `PATCH /api/video-analysis/jobs/{job_id}/visibility/`: Toggle models → `{ 'person_detection': true, 'behavior_model_1': false }`.
- **WebSocket**: `job_status_updated` events emit `{ job_id, status, progress, error }`.

---

## R8: Edge Case Handling

### Decision: Robust Error Recovery and State Management

**Identified Edge Cases**:

1. **Student leaves and returns**: Handled by embedding re-ID (R3).
2. **No detections in frame**: Bounding box overlays are empty; UI shows "No detections in frame N".
3. **Upload interrupted**: Celery task marked FAILED; user can retry from UI.
4. **Model output unavailable**: e.g., behavioral classifier didn't run. Gracefully skip overlay for that model; log error.
5. **Tracking ID conflict**: Two students with high embedding similarity. Accept first match; log confidence score for audit.
6. **All models toggled off**: UI shows raw video frame; no overlays.
7. **Very long video (>1 hour)**: Processing split into chunks; results streamed to DB; UI can view completed chunks while encoding continues.

**Error Messages** (User-Facing):
- "Upload failed: unsupported format. Supported: .mp4, .avi, .mov"
- "Processing failed: video corrupted or unreadable."
- "Your video will be ready in ~5 minutes. Current: 40% complete."

---

## R9: Technology Stack Alignment

### Decision: Leverage Existing Stack

**Stack Dependencies** (from CLAUDE.md):
- **Backend**: Python 3.13, Django 5.1.5, DRF 3.15.2, Celery 5.4.0, ultralytics 8.3.61 (supports YOLOv12 and documented track mode with ByteTrack and BoT-SORT), Pydantic 2.10.6, Redis, PostgreSQL.
- **Frontend**: React 19.2, Zustand 5, Vite 7.3, TypeScript ~5.9.
- **ASGI**: Daphne (WebSocket support already in place).

**Tracking Library**: 
- Ultralytics framework (included) supports ByteTrack and BoT-SORT with any configured YOLO model.
- No additional dependencies required for core tracking.

**Embeddings**:
- Configured YOLO model provides embeddings via `.get_embedding()` on detections.
- Fallback: Use `torch.nn.functional.cosine_similarity` for similarity scoring.

**No new major dependencies required**.

---

## R10: Deployment and Scaling Considerations

### Decision: Celery Task Queue + Redis Caching

**Selected Approach**:
- Upload jobs queued as Celery tasks; worker pool processes sequentially or in parallel (depending on hardware).
- Results cached in Redis (embeddings, frame overlays, job status).
- PostgreSQL maintains authoritative job history and audit trail.

**Rationale**:
- Existing infrastructure (Redis, Celery) already in production.
- Scaling: Add more Celery workers to increase throughput.
- Reliability: Celery retry logic for transient failures.

---

## Summary Table

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| **Tracking Algorithm** | ByteTrack default, BoT-SORT ReID-enabled | Native Ultralytics support with documented tracker YAMLs and custom tuning via YAML |
| **Tracking Scope** | Detection layer only (layer 0); students only | Cropped layers lose spatial context; teachers out of scope |
| **Re-ID Mechanism** | Embeddings + cosine similarity (0.85 threshold); refresh every 10s | Handles occlusion/exit; Redis + PostgreSQL hybrid storage |
| **Box Rendering** | Per-student color + ID overlay; independent model toggles | Improves usability; supports debugging |
| **Video Input** | Upload-first pipeline with shared processing components | Simplifies backend while keeping feature scope bounded |
| **Testing** | TDD, real video data, 80% coverage minimum | Catches edge cases; validates on production scenarios |
| **Frontend-Backend** | REST for uploads, WebSocket for real-time updates | Standard patterns; low-latency for toggles |
| **Edge Cases** | Explicit handling for all identified scenarios | Production-ready robustness |
| **Tech Stack** | Ultralytics 8.3.61 (supports YOLOv12 and future YOLO variants), Python 3.13, React 19.2, existing infra | No new dependencies; flexible model selection; leverages existing expertise |
| **Tech Stack** | Ultralytics 8.3.61 (supports YOLOv12 and future YOLO variants), Python 3.13, React 19.2, Prometheus/StatsD (observability per Q2), existing infra | No new Python/JS dependencies; Prometheus/StatsD already deployed; flexible model selection; leverages existing expertise |

---

## Next Steps (Phase 1)

1. **Data Model** (data-model.md): Define `VideoAnalysisJob`, `FrameEmbedding`, `TrackingMetadata` entities.
2. **Contracts** (contracts/): API contracts for upload, status, visibility toggle.
3. **Quickstart** (quickstart.md): Step-by-step end-to-end flow for developers.
4. **Tasks** (speckit.tasks): Generate implementation tasks in dependency order.
