# CLAUDE.md — Long-Term Memory for 014-yoloe-scene-srvl

**Last updated:** 2026-06-07
**Branch**: `014-yoloe-scene-srvl`
**Production server**: `0.tcp.eu.ngrok.io:27681`, user `bamby`, `/home/bamby/grad_project`
**GPU**: RTX 5090, CUDA 12.8, TRT 10.16.1.11
**Constraints**: No Docker, no sudo, PostgreSQL only (never SQLite)

---

## Completed Tasks: 58 / 120

### Phase 1 (T001–T007) ✅ DONE
Scene package scaffold: `__init__.py`, `config.py`, `live_guard.py`, Django app integration, settings stubs.

### Phase 2 (T008–T032) ✅ DONE
- **T011**: 11 scene models in `models.py`
- **T012**: Migration `0016_scene_segmentation_srvl.py`
- **T014**: `scene/artifacts.py`
- **T016**: `scene/access.py`
- **T018**: `scene/live_guard.py`
- **T020**: `pipeline/model_registry.py` + SceneModelDescriptor
- **T021**: `pipeline/services/model_route_service.py` + scene_segmentation route
- **T023**: `serializers_scene.py` (10 serializers)
- **T024**: `urls_scene.py` (9 routes + T068 frame-mismatch)
- **T026**: `scene/ws_events.py`
- **T028**: `scene/telemetry.py`
- **T030/T031**: `docs/entity/cycles/cycle_014_yoloe_scene_srvl.md` extended
- **T032**: `tools/prod/README.md` appended

### Phase 3 (T033–T054) ✅ DONE
- **T041**: `scene/prompts.py`
- **T042**: `scene/export_manifest.py`
- **T043**: `scene/normalizer.py` (NormalizedObject, YOLOEResultNormalizer)
- **T044**: `scene/masks.py` (rle_zstd codec)
- **T045**: `scene/persistence.py` (persist_scene_frame)
- **T046**: `scene/non_roi.py`
- **T047**: `scene/contradictions.py`
- **T048**: `scene_lane.py` → `run_scene_frame_lane` (gated offline scene lane)
- **T049**: `views_scene.py` (all read-only API views)
- **T050**: `ws_broadcast.py` + `enqueue_scene_event`
- **T051–T054**: Production export/verify scripts

### Phase 4 (T055–T068) ✅ DONE
- **T055**: `tests/unit/video_analysis/test_scene_people_counts.py`
- **T056**: `tests/unit/video_analysis/test_scene_recovery_lookup.py`
- **T057**: `tests/unit/video_analysis/test_scene_recovery_assignment.py`
- **T058**: `tests/integration/video_analysis/test_scene_recovery_candidates.py`
- **T059**: `tests/integration/video_analysis/test_scene_detector_assist.py`
- **T060**: `tests/integration/video_analysis/test_scene_recovery_append_only.py`
- **T061**: `scene/people_counts.py`
- **T062**: `scene/recovery_lookup.py`
- **T063**: `scene/recovery_assignment.py`
- **T064**: `scene/recovery_candidates.py`
- **T065**: `scene/recovery_timeline.py`
- **T066**: `scene/detector_assist.py`
- **T067**: `scene_lane.py` mismatch emission + recovery candidates
- **T068**: `views_scene.py` frame_number filter + SceneFrameMismatchView; `urls_scene.py` updated

---

## Next: Phase 5 (T069–T088) — US3 SRVL

### Pending
- **T069**: `tests/unit/video_analysis/test_srvl_math.py` — CPU-reference parity tests
- **T070**: `tests/unit/video_analysis/test_srvl_vectorized_backend.py`
- **T071**: `tests/unit/video_analysis/test_srvl_artifacts.py` — NPZ digest tests
- **T072**: `scene/srvl_inputs.py`
- **T073**: `scene/srvl_modes.py`
- **T074**: `scene/srvl.py` — SRVL compute engine
- **T075**: `scene/srvl_maps.py` — heatmap rendering
- **T076**: `scene/visualization_queue.py`
- **T077**: `scene/render_artifacts.py`
- **T078–T088**: Frontend types + components

---

## Architecture Rules
- **YOLOE_SCENE_ENABLED=0** and **SRVL_ENABLED=0** are defaults
- **offline-only**: live_guard blocks live streams
- **append-only evidence**: no mutation of prior rows
- **one-to-one assignment**: deterministic, ambiguous left unresolved
- **idempotency keys**: SHA-256 of `scene:{job_id}:{frame_number}`
- **RLE-Zstd codec**: `rle_zstd` / `RLZS` header, version 1
- **PostgreSQL only**

## Key File Paths
- `backend/apps/video_analysis/scene/` — all scene modules
- `backend/apps/video_analysis/scene_lane.py` — offline lane entry
- `backend/apps/video_analysis/models.py` — 11 scene models at bottom
- `backend/apps/video_analysis/migrations/0016_scene_segmentation_srvl.py`
- `backend/apps/video_analysis/serializers_scene.py`
- `backend/apps/video_analysis/views_scene.py`
- `backend/apps/video_analysis/urls_scene.py`
- `backend/apps/video_analysis/ws_broadcast.py`
- `backend/apps/pipeline/model_registry.py`
- `backend/config/urls.py`
- `tools/prod/` — export/verify scripts
- `specs/014-yoloe-scene-srvl/tasks.md` — 58/120 [x]
