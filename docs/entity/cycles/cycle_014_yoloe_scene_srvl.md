# Cycle 014 YOLOE Scene Segmentation And SRVL

**Last updated:** 2026-06-07
**Entity kind:** `cycle`
**Status:** `staged_local_only`

Disabled-by-default YOLOE scene-segmentation lane and Spatial Relationship
Vectorization Layer (SRVL) for offline classroom video inference.

Streaming compatibility: `offline-only`

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
| File | `backend/apps/video_analysis/models.py` |
| File | `backend/apps/video_analysis/tasks.py` |
| File | `backend/apps/video_analysis/serializers_scene.py` |
| File | `backend/apps/video_analysis/urls_scene.py` |
| File | `backend/apps/video_analysis/views_scene.py` |
| File | `backend/apps/video_analysis/ws_broadcast.py` |
| File | `backend/apps/pipeline/model_registry.py` |
| File | `backend/apps/pipeline/services/model_route_service.py` |
| File | `backend/config/settings/base.py` |
| File | `frontend/src/types/videoAnalysis.ts` |
| File | `frontend/src/components/VideoPlayer/OverlayCanvas.tsx` |
| File | `frontend/src/components/camera/BoundingBoxCanvas.tsx` |
| File | `frontend/src/components/scene/SceneMapRenderer.tsx` |
| File | `frontend/src/services/sceneMetrics.ts` |
| File | `tools/prod/prod_export_yoloe_scene_model.sh` |
| File | `tools/prod/prod_export_yoloe_scene_model.ps1` |
| File | `tools/prod/prod_verify_yoloe_scene_export.sh` |
| File | `tools/prod/prod_verify_yoloe_scene_export.ps1` |
| File | `tools/prod/prod_run_yoloe_scene_benchmark.sh` |
| File | `tools/prod/prod_run_yoloe_scene_benchmark.ps1` |
| File | `tools/prod/prod_collect_yoloe_scene_metrics.py` |
| File | `tools/prod/prod_generate_yoloe_scene_figures.py` |
| File | `tools/prod/prod_benchmark_scene_renderers.sh` |
| File | `tools/prod/prod_benchmark_scene_renderers.ps1` |
| File | `tools/prod/prod_rollback_yoloe_scene.sh` |
| File | `tools/prod/prod_rollback_yoloe_scene.ps1` |
| Workflow | `.github/workflows/yoloe-scene-srvl.yml` |
| Doc | `specs/014-yoloe-scene-srvl/plan.md` |
| Doc | `specs/014-yoloe-scene-srvl/tasks.md` |
| Doc | `specs/014-yoloe-scene-srvl/spec.md` |
| Doc | `specs/014-yoloe-scene-srvl/research.md` |

## 1. Purpose and scope

Streaming compatibility: `offline-only`

Cycle 014 adds:

- A prompt-locked YOLOE-26s-seg scene segmentation lane gated by
  `YOLOE_SCENE_ENABLED=0` (default disabled).
- A non-ROI classroom guard that flags append-only contradiction events when
  downstream detections overlap non-ROI regions (tables, chairs, walls, etc.).
- A people-count mismatch recovery classifier gated by
  `YOLOE_SCENE_MISMATCH_RECOVERY=0` (default disabled).
- A Spatial Relationship Vectorization Layer (SRVL) gated by
  `SRVL_ENABLED=0` (default disabled) that computes pairwise distance, angle,
  vector, heatmap, direction, and correlation artifacts.
- A PixiJS-backed frontend scene-map renderer candidate (T022 benchmark
  required before acceptance).
- Full production helper scripts, metrics collectors, figure generators, and
  rollback proof.

V1 live use is disabled. Any live enablement requires a separate plan with
latency, bounded-queue, and append-only evidence.

## 2. Position in the system

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'primaryTextColor': '#EDE9FE', 'primaryBorderColor': '#A78BFA', 'lineColor': '#A78BFA', 'secondaryColor': '#1E3A8A', 'tertiaryColor': '#5B21B6', 'fontSize': '14px'}}}%%
flowchart LR
    F["Decoded frame"] --> Y["Prompt-locked<br/>YOLOE-Seg<br/>(YOLOE_SCENE_ENABLED)"]
    Y --> M["Masks + boxes<br/>+ scores"]
    M --> G["Non-ROI guard<br/>append-only flag"]
    M --> R["People mismatch<br/>Redis-first check"]
    M --> S["SRVL GPU<br/>(SRVL_ENABLED)"]
    S --> A["NPZ artifacts<br/>+ digest"]
    S --> V["Viz queue<br/>latest-wins"]
    V --> UI["PixiJS<br/>scene map"]
    A --> FS[("Job artifacts")]
    G --> DB[("PostgreSQL<br/>events")]
    R --> DB
    classDef prod fill:#5B21B6,stroke:#A78BFA,color:#EDE9FE;
    classDef store fill:#1E3A8A,stroke:#60A5FA,color:#DBEAFE;
    classDef warn fill:#F59E0B,stroke:#FCD34D,color:#1F2937;
    class Y,S,UI prod;
    class FS,DB store;
    class G,R warn;
```

## 3. Configuration

All operational parameters are in `.env` / `backend/config/settings/base.py`.
See `.env.example` for the full list of `YOLOE_SCENE_*` and `SRVL_*` keys.

Master rollback: `YOLOE_SCENE_ENABLED=0` + `SRVL_ENABLED=0`.

## 4. Figure Planner and Figure Implementer

Per constitution v2.11.0 §7.1.1 and §12.6, every benchmark decision requires
exactly one Figure Planner and one Figure Implementer named here before any
benchmark decision claim.

### Figure Planner (T030)

**Role owner**: Codex 2026-06-07 spec repair session.

**Required plots** (must be generated from raw metrics artifacts, not mocked):

| Plot | Raw input | Embed target |
|---|---|---|
| YOLOE inference latency distribution (p50/p95/p99) | `scene_benchmark_raw.json` | §6 of this doc |
| SRVL compute time vs object count | `srvl_benchmark_raw.json` | §6 of this doc |
| Non-ROI contradiction rate per frame | `scene_benchmark_raw.json` | §6 of this doc |
| People mismatch rate per frame | `scene_benchmark_raw.json` | §6 of this doc |
| Scene renderer FPS (PixiJS vs pillow baseline) | `renderer_benchmark_raw.json` | §6 of this doc |

**Unavailable-metric policy**: any metric whose raw source file is absent or empty
must produce a figure placeholder with the legend "UNAVAILABLE — {reason}".  No
plot may be omitted without a recorded reason in the evidence manifest.

**Markdown embed targets**: all figures are embedded in §6 (Production benchmark
evidence) of this document as `![fig title](docs/figures/cycle_014/fig_name.png)`
with SHA-256 digest in the alt text.

### Figure Implementer (T031)

**Role owner**: Codex 2026-06-07 spec repair session.

Role-separation note: this repair was performed in a single-agent session, so a
separate Figure Implementer was not available. The planner evidence above
defines required plots, inputs, unavailable-metric handling, and Markdown embed
targets; the implementer evidence below is limited to generator code, tests,
manifest paths, and workflow coverage.

**Responsibilities**:
- Owns `tools/prod/prod_generate_yoloe_scene_figures.py` and its unit tests
  (`backend/tests/unit/pipeline/test_prod_generate_yoloe_scene_figures.py`).
- Writes a `docs/figures/cycle_014/MANIFEST.json` with `{figure_filename: sha256}` entries.
- All produced images are committed under `docs/figures/cycle_014/`.
- The CI workflow gate (`T108`) verifies figure digests against the manifest before
  accepting the benchmark evidence.

Both roles are named above. Benchmark decisions require completed production
evidence and role sign-off in §5.

## 5. Decision history

| Cycle state | Date | Notes |
|---|---|---|
| `staged_local_only` | 2026-06-07 | Local implementation and helper scaffolding complete. Production benchmark and rollback evidence still pending. |

## 6. Production benchmark evidence

*No production benchmark has been run yet. Acceptance requires a native Linux
RTX 5090 `combined.mp4` benchmark with baseline/candidate comparison table,
figures, rollback proof, and all metrics listed in SC-012.*

## 7. Open questions

- Production YOLOE cadence (run every N frames) to be measured before
  acceptance gate is set.
- PixiJS renderer benchmark (SC-010) required before frontend acceptance.
- SRVL p95 ≤ 10 ms gate (SC-009) must be confirmed on production hardware.

## 8. Implementation status (2026-06-07)

| Phase | Tasks | Status |
|---|---|---|
| Phase 1 — scaffold | T001–T007 | ✅ Complete |
| Phase 2 — foundation | T008–T032 | ✅ Complete |
| Phase 3 — YOLOE lane | T033–T054 | ✅ Complete |
| Phase 4 — US2 contradiction | T055–T068 | ✅ Complete |
| Phase 5 — SRVL pipeline | T069–T088 | ✅ Complete |
| Phase 6 — US4 production readiness | T089–T108 | Staged; T106-T107 pending production evidence |
| Phase 7 — polish | T109–T120 | Staged; T120 pending production rollback output |

Total: **117/120 tasks complete**. Local implementation and helper scaffolding
are staged. T106, T107, and T120 remain open until production benchmark,
figure, lineage, and rollback evidence are generated and linked.

### Production helper scripts added (T098–T108)

| Script | T# | Purpose |
|---|---|---|
| `prod_run_yoloe_scene_benchmark.sh` | T098 | Run benchmark, delegate to management command |
| `prod_run_yoloe_scene_benchmark.ps1` | T099 | PowerShell wrapper |
| `prod_collect_yoloe_scene_metrics.py` | T100 | Collect DB/GPU/CPU/Redis/artifact metrics |
| `prod_generate_yoloe_scene_figures.py` | T101 | Generate figures from metrics JSON |
| `prod_benchmark_scene_renderers.sh` | T102 | Run frontend renderer benchmark |
| `prod_benchmark_scene_renderers.ps1` | T103 | PowerShell renderer benchmark wrapper |
| `prod_rollback_yoloe_scene.sh` | T104 | Rollback: set flags to 0, optional SIGHUP |
| `prod_rollback_yoloe_scene.ps1` | T105 | PowerShell rollback wrapper |

### CI workflow

`.github/workflows/yoloe-scene-srvl.yml` (T108): runs on push to branch
`014-yoloe-scene-srvl` and on PRs targeting `main`. Runs backend tests, frontend
tests, migration check, and disabled-by-default env contract verification.

### Backend test suite (T113)

Local PostgreSQL-backed verification on 2026-06-07:

```text
.\.venv\Scripts\python.exe manage.py makemigrations --check --dry-run
# No changes detected

.\.venv\Scripts\python.exe manage.py check
# System check identified no issues (0 silenced).

.\.venv\Scripts\python.exe -m pytest tests\unit\video_analysis\test_srvl_math.py tests\unit\pipeline\test_prod_collect_yoloe_scene_metrics.py tests\unit\pipeline\test_prod_generate_yoloe_scene_figures.py tests\unit\pipeline\test_scene_prod_helper_powershell_syntax.py tests\unit\pipeline\test_scene_prod_helper_shell_syntax.py tests\system\test_yoloe_scene_evidence_package.py tests\system\test_yoloe_scene_reconciliation.py tests\contract\test_yoloe_scene_production_helper_contract.py -q --tb=short
# 99 passed, 3 skipped in 11.52s
```

| Suite | Status |
|---|---|
| Unit: SRVL math, vectorized, artifacts, visualization queue | ✅ Implemented |
| Integration: detector assist, append-only, scene map video | ✅ Implemented |
| System: rollback, reconciliation, evidence package | ✅ Implemented |
| Contract: production helpers | ✅ Implemented |

Actual passing run on production server: **PENDING**

### Frontend test suite (T114)

Local benchmark harness verification on 2026-06-07:

```text
npm.cmd test -- sceneRendererBenchmark.test.ts --run
# 1 test file passed, 8 tests passed
```

| Test | Status |
|---|---|
| `sceneTypes.test.ts` | ✅ Implemented |
| `sceneRendererMetrics.test.ts` | ✅ Implemented |
| `sceneRendererBenchmark.test.ts` | ✅ Implemented |
| `e2e/scene-map.spec.ts` | ✅ Implemented |

SC-010 PixiJS FPS gate: **PENDING** — requires production run

### Shell/PowerShell/Python helper checks (T115)

All scripts pass `bash -n` (syntax), contain `DRY_RUN`/`$DryRun` switch,
include no-sudo guard, and are covered by T089/T090/T097 automated tests.
See `specs/014-yoloe-scene-srvl/evidence_manifest.md §8` for full table.

### Rollback and disabled-path proof (T120)

Rollback scripts implemented: `prod_rollback_yoloe_scene.sh` and `.ps1`.
System test `test_yoloe_scene_rollback.py` verifies the disabled path.
Production rollback command output: **PENDING** — must be run and linked here.

---

## Cycle 015 — YOLOE-26L-seg Model Upgrade (2026-06-07)

**Decision:** Replace `yoloe-26s-seg` with `yoloe-26l-seg` (Large) as the scene
segmentation authority. Export path unchanged: prompt-lock 12 classes → ONNX →
**TensorRT FP16** engine → Triton `yoloe_scene_seg` (I/O contract identical:
`images[1,3,640,640]` → `output0[1,300,38]`, `output1[1,32,160,160]`).

**Artifacts (prod RTX 5090, CUDA 12.8, TRT 10.16.1.11):**
- Checkpoint `yoloe-26l-seg.pt` sha256 `a612d2d505f24e14d87ec82d688b823b6cb600646664f16125ce6c84ce360da9` (79,417,413 B)
- Deployed `model.plan` sha256 `a4357103c62f76948596a7c6210c5ddd658a3e431b30ed5917900845004cfb39` (59 MB)
- DB export manifest `7f93fd08-154b-4a40-aa02-99b664ed8506`
- 26s engine/config/checkpoint backed up at `/home/bamby/yoloe_26s_backup_20260607_185625/` (rollback)

**Full benchmark — combined.mp4 (4541 frames):**

| Metric | 26s baseline | 26L (FP16) | Δ |
|---|---|---|---|
| Objects detected | 88,363 | 111,792 | **+26.5%** |
| YOLOE inference p50 | 7.16 ms | 8.13 ms | +0.97 ms |
| YOLOE inference p95 | 11.66 ms | 9.93 ms | −1.73 ms |
| YOLOE inference p99 | 16.43 ms | 10.80 ms | −5.63 ms |
| End-to-end FPS | 15.29 | 8.51 | −44% |
| Frame latency p50 | 65.9 ms | 120.0 ms | +54 ms |

**Interpretation:** On the RTX 5090 the Large model's FP16 engine has inference
latency on par with 26s (p50 +1 ms; p95/p99 tails are *tighter*), while
detecting 26.5 % more objects. The end-to-end FPS drop is **postprocess**
scaling with object count + per-frame mask rendering — NOT model inference —
consistent with the latency analysis (postprocess = 66 % of compute).

**FP16 vs INT8 decision:** **Stay FP16.** Inference is not the bottleneck
(p95 already 9.93 ms); INT8 would risk the recall gain that motivated the Large
model for negligible latency benefit. INT8 is not warranted for this cycle.

**Follow-up (later phases):** postprocess GPU offload + batching + async dispatch
(latency-attack phase); real person/teacher embeddings → Redis + contradiction
arbitration (embedding cycle). Tracked in [[cycle014-arch-state-and-program]].
