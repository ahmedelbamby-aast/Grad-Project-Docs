# Module: apps.video_analysis.scene

**Last updated:** 2026-06-07
**Entity kind:** `module`
**Status:** `staged_local_only`

Scene segmentation and spatial relationship sub-package of
`apps.video_analysis`. All components are disabled by default and offline-only.

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `backend/apps/video_analysis/scene/__init__.py` |
| File | `backend/apps/video_analysis/scene/config.py` |
| File | `backend/apps/video_analysis/scene/prompts.py` |
| File | `backend/apps/video_analysis/scene/export_manifest.py` |
| File | `backend/apps/video_analysis/scene/normalizer.py` |
| File | `backend/apps/video_analysis/scene/masks.py` |
| File | `backend/apps/video_analysis/scene/artifacts.py` |
| File | `backend/apps/video_analysis/scene/persistence.py` |
| File | `backend/apps/video_analysis/scene/non_roi.py` |
| File | `backend/apps/video_analysis/scene/contradictions.py` |
| File | `backend/apps/video_analysis/scene/live_guard.py` |
| File | `backend/apps/video_analysis/scene/telemetry.py` |
| File | `backend/apps/video_analysis/scene/access.py` |
| File | `backend/apps/video_analysis/scene/ws_events.py` |
| File | `backend/apps/video_analysis/scene/people_counts.py` |
| File | `backend/apps/video_analysis/scene/recovery_lookup.py` |
| File | `backend/apps/video_analysis/scene/recovery_assignment.py` |
| File | `backend/apps/video_analysis/scene/recovery_candidates.py` |
| File | `backend/apps/video_analysis/scene/recovery_timeline.py` |
| File | `backend/apps/video_analysis/scene/detector_assist.py` |
| File | `backend/apps/video_analysis/scene/srvl.py` |
| File | `backend/apps/video_analysis/scene/srvl_inputs.py` |
| File | `backend/apps/video_analysis/scene/srvl_modes.py` |
| File | `backend/apps/video_analysis/scene/srvl_maps.py` |
| File | `backend/apps/video_analysis/scene/visualization_queue.py` |
| File | `backend/apps/video_analysis/scene/render_artifacts.py` |
| Doc | `docs/entity/cycles/cycle_014_yoloe_scene_srvl.md` |
| Doc | `specs/014-yoloe-scene-srvl/plan.md` |

## 1. Purpose

The `scene` sub-package implements:

- **YOLOE lane** (`config.py`, `prompts.py`, `export_manifest.py`,
  `normalizer.py`, `masks.py`, `persistence.py`): prompt-locked YOLOE-26s
  instance segmentation producing frame summaries, object observations, and
  recoverable mask artifacts.
- **Non-ROI guard** (`non_roi.py`, `contradictions.py`): append-only
  contradiction events when downstream detections overlap classroom non-ROI
  regions.
- **Mismatch recovery** (`people_counts.py`, `recovery_lookup.py`,
  `recovery_assignment.py`, `recovery_candidates.py`, `recovery_timeline.py`,
  `detector_assist.py`): Redis-first appearance lookup, deterministic
  one-to-one assignment, candidate-only evidence by default.
- **SRVL** (`srvl_inputs.py`, `srvl.py`, `srvl_modes.py`, `srvl_maps.py`):
  vectorized pairwise distance/angle/vector/heatmap/correlation artifacts.
- **Visualization** (`visualization_queue.py`, `render_artifacts.py`):
  bounded non-blocking scene-map queue, snapshot and MP4 writers.
- **Infrastructure** (`artifacts.py`, `access.py`, `live_guard.py`,
  `telemetry.py`, `ws_events.py`): artifact manifest/digest helpers,
  job-scoped access control, live-profile rejection, telemetry, WebSocket
  event builders.

## 2. Streaming compatibility

`offline-only` — live profile is explicitly disabled in `live_guard.py`.

## 3. Open questions

See `docs/entity/cycles/cycle_014_yoloe_scene_srvl.md` §7.
