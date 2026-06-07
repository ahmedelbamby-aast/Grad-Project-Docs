# CLAUDE.md — Long-Term Memory for 014-yoloe-scene-srvl

**Last updated:** 2026-06-07 (session 3)
**Branch**: `014-yoloe-scene-srvl`
**Production server**: `0.tcp.eu.ngrok.io:27681`, user `bamby`, `/home/bamby/grad_project`
**GPU**: RTX 5090, CUDA 12.8, TRT 10.16.1.11
**Constraints**: No Docker, no sudo, PostgreSQL only (never SQLite)

---

## ✅ ALL 120 / 120 TASKS COMPLETE

`specs/014-yoloe-scene-srvl/tasks.md` — 120/120 `[x]`

Feature status: `staged_local_only` — disabled-by-default
(`YOLOE_SCENE_ENABLED=0`, `SRVL_ENABLED=0`)

---

## Phase Summary

### Phase 1 (T001–T007) ✅ DONE
Scene package scaffold: `__init__.py`, `config.py`, `live_guard.py`, Django app integration, settings stubs.

### Phase 2 (T008–T032) ✅ DONE
11 scene models, migration `0016_scene_segmentation_srvl.py`, `serializers_scene.py`, `urls_scene.py`, `ws_events.py`, `telemetry.py`, cycle doc, tools README.

### Phase 3 (T033–T054) ✅ DONE
`scene_lane.py` → `run_scene_frame_lane`, all scene sub-modules (`prompts.py`, `normalizer.py`, `masks.py`, `persistence.py`, `non_roi.py`, `contradictions.py`), production export/verify scripts.

### Phase 4 (T055–T068) ✅ DONE
US2 people-count mismatch: `people_counts.py`, `recovery_lookup.py`, `recovery_assignment.py`, `recovery_candidates.py`, `recovery_timeline.py`, `detector_assist.py`. All test files (T055–T060). `views_scene.py` frame filter + `SceneFrameMismatchView`.

### Phase 5 (T069–T088) ✅ DONE
US3 SRVL:
- `scene/srvl_inputs.py` (MAX_OBJECTS=128, SRVLInputObject, prepare_srvl_inputs)
- `scene/srvl.py` (compute_srvl, vectorised O(N²) NumPy)
- `scene/srvl_modes.py` (full, top_k, thresholded, heatmap_only)
- `scene/srvl_maps.py` (heatmap, direction_map, correlation_map)
- `scene/visualization_queue.py` (VisualizationQueue, latest-frame-wins)
- `scene/render_artifacts.py` (atomic NPZ write, SHA-256, PNG snapshot)
- `scene_lane.py` `_run_srvl_lane` appended
- Frontend: `types/videoAnalysis.ts`, `SceneMapRenderer.tsx`, `sceneMetrics.ts`
- Frontend: `OverlayCanvas.tsx` scene badges, `BoundingBoxCanvas.tsx` truth-state
- Test files T069–T076

### Phase 6 (T089–T108) ✅ DONE
US4 production readiness:
- `tools/prod/prod_run_yoloe_scene_benchmark.sh` + `.ps1` (T098, T099)
- `tools/prod/prod_collect_yoloe_scene_metrics.py` (T100)
- `tools/prod/prod_generate_yoloe_scene_figures.py` (T101)
- `tools/prod/prod_benchmark_scene_renderers.sh` + `.ps1` (T102, T103)
- `tools/prod/prod_rollback_yoloe_scene.sh` + `.ps1` (T104, T105)
- `docs/production_inference_benchmark.md` updated (T106)
- `docs/entity/cycles/cycle_014_yoloe_scene_srvl.md` updated (T107)
- `.github/workflows/yoloe-scene-srvl.yml` (T108)
- Test files: T089–T097 (shell syntax, PS syntax, metrics, figures, rollback, reconciliation, evidence, contract)

### Phase 7 (T109–T120) ✅ DONE
Polish & cross-cutting:
- `specs/014-yoloe-scene-srvl/quickstart.md` updated with reproduction commands (T109, T111)
- `docs/yoloe_scene_segmentation_plan.md` gap notes updated (T110)
- `docs/production_inference_benchmark.md` reading-order note (T112)
- `docs/entity/cycles/cycle_014_yoloe_scene_srvl.md` test suite records + rollback proof section (T113–T115, T120)
- `specs/014-yoloe-scene-srvl/evidence_manifest.md` produced (T116–T119)
- `frontend/tests/sceneRendererMetrics.test.ts` (T093)

---

## Architecture Rules
- **YOLOE_SCENE_ENABLED=0** and **SRVL_ENABLED=0** are defaults (never flip in CI)
- **offline-only**: `live_guard.assert_offline` blocks live streams
- **append-only evidence**: no mutation of prior contradiction/recovery/mismatch rows
- **one-to-one assignment**: deterministic, ambiguous left unresolved
- **idempotency keys**: SHA-256 of `scene:{job_id}:{frame_number}`
- **RLE-Zstd codec**: `rle_zstd` / `RLZS` header, version 1
- **PostgreSQL only** — never SQLite
- **MAX_OBJECTS=128** for dense SRVL matrices
- **Atomic NPZ writes**: `tempfile.mkstemp` + `os.replace`
- **No Python pair-loop** in SRVL hot path
- **Latest-frame-wins** in VisualizationQueue

## Key File Paths
- `backend/apps/video_analysis/scene/` — all scene sub-modules
- `backend/apps/video_analysis/scene_lane.py` — offline lane entry + SRVL wiring
- `backend/apps/video_analysis/models.py` — 11 scene models
- `backend/apps/video_analysis/migrations/0016_scene_segmentation_srvl.py`
- `backend/apps/video_analysis/serializers_scene.py`
- `backend/apps/video_analysis/views_scene.py`
- `backend/apps/video_analysis/urls_scene.py`
- `backend/apps/video_analysis/ws_broadcast.py`
- `backend/apps/pipeline/model_registry.py`
- `frontend/src/types/videoAnalysis.ts` — scene TS types
- `frontend/src/components/scene/SceneMapRenderer.tsx`
- `frontend/src/services/sceneMetrics.ts`
- `tools/prod/` — all production helper scripts
- `specs/014-yoloe-scene-srvl/evidence_manifest.md`
- `.github/workflows/yoloe-scene-srvl.yml`

## Figure-Generation Pipeline (post-120 enhancement) ✅ DONE

### Overview
All production benchmark scripts now gate on figure generation — the job cannot
be marked complete until all 5 figures + MANIFEST.json are written.

### Files modified / created
- `tools/prod/prod_generate_yoloe_scene_figures.py` (rewritten, ~505 lines)
  - Always emits every figure — data-driven or placeholder PNG if unavailable
  - `_placeholder_chart(title, reason, out)` — red "UNAVAILABLE" PNG
  - `generate_db_summary_chart`, `generate_artifact_sizes_chart`, `generate_gpu_chart`
  - `generate_box_throughput_chart` — grouped bars (5 models × 4 metrics)
  - `generate_throughput_summary_chart` — horizontal bar
  - `_write_figure_manifest(out_dir, metrics)` — MANIFEST.json with SHA-256
  - `_generate_all(metrics, out_dir)` → True when job terminal
  - `--poll-interval N` + `--collect-command CMD` real-time polling loop
  - `--max-polls N` safety cap; partial figures watermarked "IN PROGRESS"
- `tools/prod/prod_collect_yoloe_scene_metrics.py` (~387 lines)
  - `collect_box_throughput_metrics(processed_frames, elapsed_s)` added
  - `_collect_job_elapsed(job_id)` helper
  - `--generate-figures` + `--figures-dir` flags
- `tools/prod/prod_collect_benchmark_metrics.py` (~1871 lines)
  - `_generate_figures_from_metrics(metrics, figures_dir)` — non-fatal subprocess
  - `--figures-dir DIR` wired to end of `main()`
- `tools/prod/prod_run_yoloe_scene_benchmark.sh` (rewritten, 5-step pipeline)
  - Step 1: validate scene model export
  - Step 2: run management command
  - Step 3: collect metrics (`prod_collect_benchmark_metrics.py --figures-dir`)
  - Step 4: generate scene figures (direct + scene-metrics pass)
  - Step 5: verify MANIFEST.json exists and has ≥1 figure — exits 1 if missing
  - `--figures-dir DIR`, `--replay-key KEY`, `--dry-run` CLI args
- `backend/tests/unit/pipeline/test_prod_generate_yoloe_scene_figures.py` (updated)
  - 3 tests updated to match "always-emit placeholder" contract

### Key design decisions
- **Always emit**: every chart function always writes a PNG (placeholder if unavailable).
- **Real-time**: poll loop via `--poll-interval` regenerates figures on every tick.
- **Non-fatal**: figure failures print WARNING but never abort metrics collection.
- **MANIFEST.json gate**: `prod_run_yoloe_scene_benchmark.sh` refuses exit 0
  unless MANIFEST.json exists with ≥1 figure.

### CI fix (commit 51f095f failure)
- Root cause: `test_prod_generate_yoloe_scene_figures.py` expected NO png for
  unavailable data, but rewritten script always writes placeholder png.
- Fix: updated 3 test assertions to `assert (path).exists()`.
- All 27 backend CI tests pass locally (shell syntax × 12, figures × 7, collect × 8).

---

## Box Throughput Metrics Feature (post-120 enhancement) ✅ DONE

Added `generated_boxes per frame / per ms / per second` metrics for **all 7 supported models**.

### Files modified / created
- `tools/prod/prod_collect_benchmark_metrics.py`
  - `ALL_SUPPORTED_MODELS` (7 models: 5 bbox, 1 keypoints, 1 segmentation)
  - `BBOX_PRODUCING_MODELS` (5-tuple of bbox model names)
  - `_compute_box_throughput_per_model()` — returns per-model dict with:
    `boxes_total`, `boxes_per_frame`, `boxes_per_second`, `boxes_per_ms`,
    `boxes_per_model_call`, `boxes_per_call_ms`, `call_count`, `mean_rtt_ms`,
    `p95_rtt_ms`, `calls_per_s`, `unavailable_reason`
  - Wired into `_collect_db_metrics()` return as `box_throughput` key
  - `_build_throughput_layers()` — per-model `boxes_{model}` rows (boxes/s)
  - `_write_markdown()` — "Per-Model Box Throughput" table section
  - `_comparison_rows()` — boxes/frame, /s, /ms, /call-ms comparison rows
- `tools/prod/prod_watch_benchmark_metrics.sh`
  - `BBOX_PRODUCING_MODELS` constant added after `LOG_DIR`
  - `_throughput_rows()` extended with `bbox_by_model` param — emits one
    `boxes/{model_name}` row per bbox model (count, rate/s, per_frame)
  - Call-site updated to pass `{row[0]: row[1] for row in bbox_rows}`
- `backend/tests/unit/pipeline/test_box_throughput_metrics.py` ← **new**
  - 38 unit tests covering: normal accuracy, zero frames/elapsed,
    non-bbox unavailable reasons, no model calls, zero mean_rtt_ms,
    all 5 bbox models simultaneously, mixed partial unavailability

### Unavailable-reason policy
- `processed_frames_zero` — no frames processed
- `elapsed_s_unavailable` — wall time missing or zero
- `no_bbox_rows` — bbox model produced 0 boxes (model disabled or not run)
- `no_model_calls` — no TelemetryModelCall rows
- `mean_rtt_ms_zero` — rtt is zero; per-call-ms undefined
- `output_kind_is_{kind}` — rtmpose_model (keypoints), yoloe_scene_seg (segmentation)

---

## Pending (acceptance gates — not implementation tasks)
- SC-010: PixiJS renderer benchmark ≥30 FPS on production RTX 5090
- SC-009: SRVL p95 ≤ 10 ms on production hardware
- Production `combined.mp4` benchmark run with YOLOE_SCENE_ENABLED=1
- Link rollback command output to cycle_014 §T120 slot
