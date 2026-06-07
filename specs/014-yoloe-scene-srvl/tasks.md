# Tasks: YOLOE Scene Segmentation And SRVL

**Input**: Design documents from `/specs/014-yoloe-scene-srvl/`
**Prerequisites**: `plan.md`, `spec.md`, `research.md`, `data-model.md`, `contracts/`, `quickstart.md`

**Tests and Evidence**: This feature touches production inference, artifacts,
identity-scoped recovery evidence, frontend rendering, PostgreSQL persistence,
Redis lookup evidence, and benchmark claims. Tests are mandatory for every user
story. Local Windows checks are contract evidence only; production acceptance
requires native Linux RTX 5090 execution, PostgreSQL, Redis where recovery is
enabled, active Triton route evidence, no Docker, no `sudo`, raw benchmark
artifacts, generated figures from the same raw artifacts, and rollback proof.

**Organization**: Tasks are grouped by user story so each story is independently
implementable and testable after the shared foundation is complete.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel because it touches different files or only reads
  established contracts.
- **[Story]**: User-story phases use `[US1]`, `[US2]`, `[US3]`, or `[US4]`.
- Every task names exact target paths and cites covered FR/SC identifiers.

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Prepare feature scaffolding, dependency declarations, configuration
contract locations, and baseline documentation before shared model/runtime work.

- [x] T001 Create scene package scaffold in `backend/apps/video_analysis/scene/__init__.py` for YOLOE/SRVL modules. (FR-001)
- [x] T002 [P] Add scene env contract placeholders and default disabled flags in `.env.example`. (FR-001, FR-032, FR-033, SC-018, SC-022)
- [x] T003 [P] Add backend dependency declarations for YOLOE export, RLE-Zstd masks, and NPZ artifact handling in `backend/pyproject.toml`. (FR-002, FR-007)
- [x] T004 [P] Add frontend renderer dependency candidate entry for PixiJS benchmarking in `frontend/package.json`. (FR-022, SC-010)
- [x] T005 [P] Add feature evidence root and generated-artifact ignore exceptions in `.gitignore` for CI-required manifests under `specs/014-yoloe-scene-srvl/` and `docs/figures/`. (FR-027, FR-028, SC-017, SC-018)
- [x] T006 [P] Create implementation evidence stub in `docs/entity/cycles/cycle_014_yoloe_scene_srvl.md` with `Streaming compatibility: offline-only`. (FR-028, SC-012, SC-013, SC-018)
- [x] T007 [P] Add scene module documentation stub in `docs/entity/modules/apps.video_analysis.scene.md`. (FR-027, FR-028)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Shared configuration, schema, route, artifact, access, telemetry,
and governance foundations that must exist before any user story can close.

**Critical**: No user-story completion claim is valid until this phase is done.

- [x] T008 Define typed scene configuration in `backend/apps/video_analysis/scene/config.py` and parse all YOLOE/SRVL/recovery/render/artifact keys through settings. (FR-001, FR-031, FR-032, FR-033, SC-022)
- [x] T009 [P] Add configuration tests for declared scene env keys and forbidden hardcoded operational values in `backend/tests/unit/video_analysis/test_scene_config.py`. (FR-032, FR-033, SC-018, SC-022)
- [x] T010 Add scene runtime settings wiring in `backend/config/settings/base.py`. (FR-032, FR-033, SC-022)
- [x] T011 Define Django models for prompt profiles, export manifests, frame summaries, object observations, non-ROI masks, contradiction events, mismatch events, recovery candidates, SRVL frames, visualization artifacts, artifact manifests, access grants, and runtime configuration in `backend/apps/video_analysis/models.py`. (FR-004, FR-007, FR-009, FR-010, FR-014, FR-018, FR-021, FR-024, FR-030, FR-031, SC-003, SC-004, SC-005, SC-017)
- [x] T012 Add PostgreSQL migration for scene models and indexes in `backend/apps/video_analysis/migrations/00XX_scene_segmentation_srvl.py`. (FR-007, FR-028, SC-017, SC-018)
- [x] T013 [P] Add model and migration tests for PostgreSQL constraints, idempotency keys, and append-only evidence in `backend/tests/unit/video_analysis/test_scene_models.py`. (FR-007, FR-009, FR-014, FR-016, FR-028, SC-007, SC-017, SC-019)
- [x] T014 Implement scene artifact manifest and digest helpers in `backend/apps/video_analysis/scene/artifacts.py`. (FR-004, FR-007, FR-024, FR-026, SC-003, SC-004, SC-013, SC-017)
- [x] T015 [P] Add artifact manifest schema tests in `backend/tests/unit/video_analysis/test_scene_artifacts.py`. (FR-007, FR-024, SC-004, SC-013, SC-017)
- [x] T016 Implement scene access-control policy helpers in `backend/apps/video_analysis/scene/access.py`. (FR-030, SC-015)
- [x] T017 [P] Add scene artifact authorization tests in `backend/tests/contract/video_analysis/test_scene_artifact_access.py`. (FR-030, SC-015)
- [x] T018 Implement live-profile rejection and disabled-for-live status helpers in `backend/apps/video_analysis/scene/live_guard.py`. (FR-001, FR-028, SC-018)
- [x] T019 [P] Add live-disabled contract tests in `backend/tests/integration/video_analysis/test_scene_live_disabled.py`. (FR-001, FR-028, SC-018)
- [x] T020 Register scene route metadata and prompt manifest validation hooks in `backend/apps/pipeline/model_registry.py`. (FR-002, FR-003, FR-004, FR-005, SC-002)
- [x] T021 Wire scene model-route validation into `backend/apps/pipeline/services/model_route_service.py`. (FR-003, FR-004, FR-005, SC-002)
- [x] T022 [P] Add model-route and manifest compatibility tests in `backend/tests/unit/pipeline/test_yoloe_scene_model_route.py`. (FR-002, FR-003, FR-004, FR-005, SC-002)
- [x] T023 Define API serializers with explicit fields in `backend/apps/video_analysis/serializers_scene.py`. (FR-006, FR-007, FR-023, FR-030, FR-031, SC-014, SC-015, SC-021)
- [x] T024 Define API routes for scene summary, frame scene data, scene artifacts, and scene-map video in `backend/apps/video_analysis/urls_scene.py`. (FR-023, FR-030, SC-014, SC-015)
- [x] T025 [P] Add API schema contract tests from `contracts/api-scene-artifacts.md` in `backend/tests/contract/video_analysis/test_scene_api_contract.py`. (FR-023, FR-030, SC-014, SC-015)
- [x] T026 Define WebSocket payload builders for scene events in `backend/apps/video_analysis/scene/ws_events.py`. (FR-020, FR-023, FR-024, FR-031, SC-011, SC-014)
- [x] T027 [P] Add WebSocket contract tests from `contracts/websocket-scene-events.md` in `backend/tests/contract/video_analysis/test_scene_ws_contract.py`. (FR-020, FR-023, FR-024, SC-011, SC-014)
- [x] T028 Define scene telemetry and reconciliation record helpers in `backend/apps/video_analysis/scene/telemetry.py`. (FR-024, FR-026, FR-028, SC-012, SC-016, SC-017)
- [x] T029 [P] Add telemetry/unavailable metric tests in `backend/tests/unit/video_analysis/test_scene_telemetry.py`. (FR-024, FR-026, SC-012, SC-016, SC-017)
- [x] T030 Document Figure Planner ownership, required plots, raw inputs, unavailable metric policy, and Markdown embed targets in `docs/entity/cycles/cycle_014_yoloe_scene_srvl.md`. (FR-026, FR-028, SC-012, SC-013)
- [x] T031 Document Figure Implementer ownership, generator changes, tests, manifest/digest output paths, and workflow updates in `docs/entity/cycles/cycle_014_yoloe_scene_srvl.md`. (FR-025, FR-027, FR-028, SC-013)
- [x] T032 Add no-sudo production preflight assertions for scene helpers in `tools/prod/README.md`. (FR-025, FR-028, SC-018)

**Checkpoint**: Foundation ready. User stories can proceed without changing the
shared schema and contract rules.

---

## Phase 3: User Story 1 - Segment Classroom Scene Evidence (Priority: P1)

**Goal**: Prompt-locked YOLOE scene segmentation produces traceable scene frame
summaries, object observations, masks, non-ROI guard evidence, and disabled-path
parity for offline video jobs.

**Independent Test**: Replay a classroom video with scene segmentation enabled
and verify selected scene frames produce prompt-profile identity, object
summaries, confidence evidence, recoverable masks, non-ROI masks, contradiction
events, artifact lineage, and disabled-path parity.

### Validation and Evidence for User Story 1

- [x] T033 [P] [US1] Add prompt-locked export unit tests for `model.set_classes(...)` before export in `backend/tests/unit/video_analysis/test_yoloe_scene_export.py`. (FR-002, FR-003, FR-004, SC-002)
- [x] T034 [P] [US1] Add stale prompt, missing manifest, wrong checkpoint, and prompt-after-export rejection tests in `backend/tests/unit/video_analysis/test_yoloe_scene_export_validation.py`. (FR-002, FR-003, FR-004, FR-005, SC-002)
- [x] T035 [P] [US1] Add YOLOE normalizer tests preserving boxes, class IDs, names, confidences, masks, shape metadata, raw refs, and unavailable reasons in `backend/tests/unit/video_analysis/test_yoloe_scene_normalizer.py`. (FR-006, FR-031, SC-004, SC-021)
- [x] T036 [P] [US1] Add RLE-Zstd mask and raw-output artifact round-trip tests in `backend/tests/unit/video_analysis/test_scene_mask_artifacts.py`. (FR-006, FR-007, SC-004, SC-021)
- [x] T037 [P] [US1] Add offline disabled-path parity tests in `backend/tests/integration/video_analysis/test_scene_disabled_offline_parity.py`. (FR-001, FR-029, SC-001)
- [x] T038 [P] [US1] Add selected-frame scene lane integration tests in `backend/tests/integration/video_analysis/test_scene_lane_offline.py`. (FR-001, FR-006, FR-007, FR-008, FR-024, SC-003, SC-004)
- [x] T039 [P] [US1] Add non-ROI append-only contradiction tests in `backend/tests/integration/video_analysis/test_scene_non_roi_guard.py`. (FR-008, FR-009, FR-016, FR-031, SC-007)
- [x] T040 [P] [US1] Add artifact authenticity tests for scene summaries, masks, traces, and manifest digests in `backend/tests/integration/video_analysis/test_scene_evidence_lineage.py`. (FR-004, FR-007, FR-024, FR-028, SC-003, SC-004, SC-017)

### Implementation for User Story 1

- [x] T041 [US1] Implement prompt profile resolution and digesting in `backend/apps/video_analysis/scene/prompts.py`. (FR-003, FR-004, FR-008, FR-032, FR-033, SC-002, SC-022)
- [x] T042 [US1] Implement YOLOE export manifest writer and validator in `backend/apps/video_analysis/scene/export_manifest.py`. (FR-002, FR-003, FR-004, FR-005, SC-002)
- [x] T043 [US1] Implement YOLOE runtime result normalizer in `backend/apps/video_analysis/scene/normalizer.py`. (FR-006, FR-031, SC-004, SC-021)
- [x] T044 [US1] Implement mask encoding and unavailable-output recording in `backend/apps/video_analysis/scene/masks.py`. (FR-006, FR-007, FR-024, FR-031, SC-004, SC-021)
- [x] T045 [US1] Implement scene frame persistence service with idempotency keys in `backend/apps/video_analysis/scene/persistence.py`. (FR-007, FR-024, FR-028, SC-003, SC-004, SC-017, SC-019)
- [x] T046 [US1] Implement non-ROI union mask builder in `backend/apps/video_analysis/scene/non_roi.py`. (FR-008, FR-009, FR-031, SC-007)
- [x] T047 [US1] Implement append-only contradiction event service in `backend/apps/video_analysis/scene/contradictions.py`. (FR-009, FR-016, FR-031, SC-007)
- [x] T048 [US1] Wire gated scene lane into offline frame processing in `backend/apps/video_analysis/tasks.py`. (FR-001, FR-006, FR-007, FR-008, FR-024, SC-001, SC-003, SC-004)
- [x] T049 [US1] Wire scene summary and frame-scene API views in `backend/apps/video_analysis/views_scene.py`. (FR-023, FR-030, FR-031, SC-014, SC-015)
- [x] T050 [US1] Wire compact scene WebSocket broadcasts in `backend/apps/video_analysis/ws_broadcast.py`. (FR-020, FR-023, FR-024, SC-011, SC-014)
- [x] T051 [US1] Create prompt-locked export helper shell script in `tools/prod/prod_export_yoloe_scene_model.sh`. (FR-002, FR-003, FR-004, FR-025, SC-002)
- [x] T052 [US1] Create prompt-locked export helper PowerShell contract script in `tools/prod/prod_export_yoloe_scene_model.ps1`. (FR-002, FR-003, FR-004, FR-025, SC-002)
- [x] T053 [US1] Create export verification helper in `tools/prod/prod_verify_yoloe_scene_export.sh`. (FR-004, FR-005, FR-025, SC-002)
- [x] T054 [US1] Create export verification PowerShell contract helper in `tools/prod/prod_verify_yoloe_scene_export.ps1`. (FR-004, FR-005, FR-025, SC-002)

**Checkpoint**: US1 can be validated independently with scene segmentation
enabled and disabled on the offline profile.

---

## Phase 4: User Story 2 - Reconcile Missing People (Priority: P2)

**Goal**: YOLOE person-count mismatches are classified with Redis-first
appearance evidence, deterministic one-to-one assignment, candidate-only
recovery evidence, unresolved ambiguity, and no retroactive mutation.

**Independent Test**: Feed known mismatch frames and verify Redis-first lookup,
one-to-one assignment, ambiguous unresolved handling, candidate-only default,
and append-only recovery evidence.

### Validation and Evidence for User Story 2

- [x] T055 [P] [US2] Add person-count comparison tests in `backend/tests/unit/video_analysis/test_scene_people_counts.py`. (FR-010, SC-005)
- [x] T056 [P] [US2] Add Redis-first and DB-fallback lookup tests in `backend/tests/unit/video_analysis/test_scene_recovery_lookup.py`. (FR-011, FR-026, SC-005)
- [x] T057 [P] [US2] Add deterministic one-to-one assignment and unresolved ambiguity tests in `backend/tests/unit/video_analysis/test_scene_recovery_assignment.py`. (FR-012, FR-013, SC-006)
- [x] T058 [P] [US2] Add append-only missing-candidate persistence tests in `backend/tests/integration/video_analysis/test_scene_recovery_candidates.py`. (FR-014, FR-015, FR-016, SC-005, SC-006, SC-019)
- [x] T059 [P] [US2] Add detector-assist recurrence tests in `backend/tests/integration/video_analysis/test_scene_detector_assist.py`. (FR-017, SC-005)
- [x] T060 [P] [US2] Add recovery no-retroactive-mutation tests in `backend/tests/integration/video_analysis/test_scene_recovery_append_only.py`. (FR-014, FR-015, FR-016, SC-006, SC-007, SC-019)

### Implementation for User Story 2

- [x] T061 [US2] Implement YOLOE versus student-plus-teacher count comparison in `backend/apps/video_analysis/scene/people_counts.py`. (FR-010, SC-005)
- [x] T062 [US2] Implement Redis-first appearance and configured slower fallback lookup in `backend/apps/video_analysis/scene/recovery_lookup.py`. (FR-011, FR-026, SC-005)
- [x] T063 [US2] Implement deterministic one-to-one recovery assignment in `backend/apps/video_analysis/scene/recovery_assignment.py`. (FR-012, FR-013, SC-006)
- [x] T064 [US2] Implement recovery candidate scoring, thresholds, reason codes, and candidate-only default in `backend/apps/video_analysis/scene/recovery_candidates.py`. (FR-012, FR-014, FR-015, FR-031, FR-032, SC-005, SC-006, SC-022)
- [x] T065 [US2] Implement original-frame-time synchronization and append-only lineage for recovery candidates in `backend/apps/video_analysis/scene/recovery_timeline.py`. (FR-014, FR-016, SC-005, SC-019)
- [x] T066 [US2] Implement bounded detector-assist evidence for following frames in `backend/apps/video_analysis/scene/detector_assist.py`. (FR-017, SC-005)
- [x] T067 [US2] Wire mismatch event emission into offline scene lane in `backend/apps/video_analysis/tasks.py`. (FR-010, FR-011, FR-012, FR-014, FR-024, SC-005)
- [x] T068 [US2] Extend scene API and WebSocket payloads with mismatch/recovery state in `backend/apps/video_analysis/views_scene.py`. (FR-023, FR-024, SC-005, SC-014)

**Checkpoint**: US2 can be validated independently with fixture mismatch frames
and candidate-only recovery enabled.

---

## Phase 5: User Story 3 - Visualize Spatial Relationships (Priority: P3)

**Goal**: SRVL computes bounded distance/vector/map artifacts and the frontend
renders scene maps without blocking analytics or playback.

**Independent Test**: Run SRVL on representative object counts and verify
matrix correctness, artifact digests, frontend FPS, dropped visualization
counters, and source frame-count preservation.

### Validation and Evidence for User Story 3

- [x] T069 [P] [US3] Add SRVL CPU-reference parity tests for zero, one, normal, invalid-coordinate, duplicate-ID, 128-object, and above-128-object cases in `backend/tests/unit/video_analysis/test_srvl_math.py`. (FR-018, FR-019, SC-008)
- [x] T070 [P] [US3] Add SRVL GPU vectorization and no Python pair-loop regression tests in `backend/tests/unit/video_analysis/test_srvl_vectorized_backend.py`. (FR-018, FR-019, SC-008, SC-009)
- [x] T071 [P] [US3] Add SRVL artifact manifest and NPZ digest tests in `backend/tests/unit/video_analysis/test_srvl_artifacts.py`. (FR-007, FR-018, FR-019, FR-024, SC-008, SC-017)
- [x] T072 [P] [US3] Add visualization queue and latest-frame-wins tests in `backend/tests/unit/video_analysis/test_scene_visualization_queue.py`. (FR-020, SC-011)
- [x] T073 [P] [US3] Add rendered MP4 source-frame-count preservation tests in `backend/tests/integration/video_analysis/test_scene_map_video.py`. (FR-021, SC-010, SC-011)
- [x] T074 [P] [US3] Add frontend optional-scene-field type and compatibility tests in `frontend/tests/sceneTypes.test.ts`. (FR-023, SC-014)
- [x] T075 [P] [US3] Add PixiJS renderer benchmark harness tests in `frontend/tests/sceneRendererBenchmark.test.ts`. (FR-020, FR-021, FR-022, SC-010, SC-011)
- [x] T076 [P] [US3] Add frontend Playwright scene-map fallback and absent-field checks in `frontend/tests/e2e/scene-map.spec.ts`. (FR-020, FR-021, FR-023, SC-010, SC-014)

### Implementation for User Story 3

- [x] T077 [US3] Implement SRVL coordinate normalization and object ordering in `backend/apps/video_analysis/scene/srvl_inputs.py`. (FR-018, FR-019, FR-024, SC-008)
- [x] T078 [US3] Implement vectorized SRVL pairwise distance, angle, magnitude-angle vector, and optional Cartesian outputs in `backend/apps/video_analysis/scene/srvl.py`. (FR-018, FR-019, SC-008, SC-009)
- [x] T079 [US3] Implement SRVL mode selection for full, top-k, thresholded, and heatmap-only outputs in `backend/apps/video_analysis/scene/srvl_modes.py`. (FR-019, FR-032, SC-008, SC-022)
- [x] T080 [US3] Implement SRVL heatmap, direction map, and correlation map artifacts in `backend/apps/video_analysis/scene/srvl_maps.py`. (FR-018, FR-019, FR-024, SC-008)
- [x] T081 [US3] Implement non-blocking visualization queue and latest-frame-wins counters in `backend/apps/video_analysis/scene/visualization_queue.py`. (FR-020, FR-024, SC-011)
- [x] T082 [US3] Implement scene-map MP4 and snapshot artifact writer in `backend/apps/video_analysis/scene/render_artifacts.py`. (FR-021, FR-024, SC-010, SC-011)
- [x] T083 [US3] Wire SRVL and visualization artifacts into offline scene lane in `backend/apps/video_analysis/tasks.py`. (FR-018, FR-019, FR-020, FR-021, FR-024, SC-008, SC-009, SC-011)
- [x] T084 [US3] Add frontend scene payload types in `frontend/src/types/videoAnalysis.ts`. (FR-023, SC-014)
- [x] T085 [US3] Add scene overlay controls and optional absent-field handling in `frontend/src/components/VideoPlayer/OverlayCanvas.tsx`. (FR-020, FR-021, FR-023, SC-010, SC-014)
- [x] T086 [US3] Add camera bounding-box scene truth-state overlays in `frontend/src/components/camera/BoundingBoxCanvas.tsx`. (FR-020, FR-021, FR-023, FR-031, SC-010, SC-014)
- [x] T087 [US3] Implement PixiJS scene-map renderer candidate in `frontend/src/components/scene/SceneMapRenderer.tsx`. (FR-020, FR-021, FR-022, SC-010, SC-011)
- [x] T088 [US3] Implement frontend renderer metrics collection in `frontend/src/services/sceneMetrics.ts`. (FR-020, FR-022, FR-026, SC-010, SC-011, SC-012)

**Checkpoint**: US3 can be validated independently with synthetic scene payloads
and representative object-count fixtures.

---

## Phase 6: User Story 4 - Prove Production Readiness (Priority: P4)

**Goal**: Evidence helpers, benchmark collectors, figures, workflows,
documentation, and rollback proof prevent acceptance from mocks, placeholders,
or unsupported local runs.

**Independent Test**: Review the evidence package and verify every accepted
claim resolves to tracked files or linked production artifacts with manifest
digests, production job lineage, generated figures, unavailable-metric reasons,
and rollback proof.

### Validation and Evidence for User Story 4

- [x] T089 [P] [US4] Add shell syntax tests for scene production helpers in `backend/tests/unit/pipeline/test_scene_prod_helper_shell_syntax.py`. (FR-025, FR-027, SC-018)
- [x] T090 [P] [US4] Add PowerShell parser tests for scene production helpers in `backend/tests/unit/pipeline/test_scene_prod_helper_powershell_syntax.py`. (FR-025, FR-027, SC-018)
- [x] T091 [P] [US4] Add metrics collector unit tests for throughput, latency, model calls, GPU, CPU, PostgreSQL, Redis, frontend, artifact, correctness, and unavailable reasons in `backend/tests/unit/pipeline/test_prod_collect_yoloe_scene_metrics.py`. (FR-026, SC-012)
- [x] T092 [P] [US4] Add figure generator tests proving input digests, missing-metric policy, and produced images in `backend/tests/unit/pipeline/test_prod_generate_yoloe_scene_figures.py`. (FR-025, FR-026, FR-027, FR-028, SC-013)
- [x] T093 [P] [US4] Add renderer benchmark collector tests in `frontend/tests/sceneRendererMetrics.test.ts`. (FR-022, FR-026, SC-010, SC-012)
- [x] T094 [P] [US4] Add rollback proof tests for disabled scene/SRVL flags and offline parity in `backend/tests/system/test_yoloe_scene_rollback.py`. (FR-001, FR-029, SC-001)
- [x] T095 [P] [US4] Add runtime reconciliation system test for task, queue, database, Redis, artifact, telemetry, frontend, and rendered media states in `backend/tests/system/test_yoloe_scene_reconciliation.py`. (FR-024, FR-026, FR-028, SC-016)
- [x] T096 [P] [US4] Add artifact access and immutable evidence system test in `backend/tests/system/test_yoloe_scene_evidence_package.py`. (FR-027, FR-028, FR-030, SC-015, SC-017, SC-018)
- [x] T097 [P] [US4] Add production benchmark command dry-run contract test in `backend/tests/contract/test_yoloe_scene_production_helper_contract.py`. (FR-025, FR-026, SC-012, SC-013)

### Implementation for User Story 4

- [x] T098 [US4] Implement production benchmark runner in `tools/prod/prod_run_yoloe_scene_benchmark.sh`. (FR-025, FR-026, FR-028, SC-012)
- [x] T099 [US4] Implement production benchmark PowerShell contract helper in `tools/prod/prod_run_yoloe_scene_benchmark.ps1`. (FR-025, FR-026, SC-012)
- [x] T100 [US4] Implement production metrics collector in `tools/prod/prod_collect_yoloe_scene_metrics.py`. (FR-025, FR-026, SC-012, SC-016, SC-017)
- [x] T101 [US4] Implement benchmark figure generator in `tools/prod/prod_generate_yoloe_scene_figures.py`. (FR-025, FR-026, FR-027, FR-028, SC-013)
- [x] T102 [US4] Implement renderer benchmark helper in `tools/prod/prod_benchmark_scene_renderers.sh`. (FR-022, FR-025, FR-026, SC-010, SC-012)
- [x] T103 [US4] Implement renderer benchmark PowerShell contract helper in `tools/prod/prod_benchmark_scene_renderers.ps1`. (FR-022, FR-025, SC-010)
- [x] T104 [US4] Implement rollback helper in `tools/prod/prod_rollback_yoloe_scene.sh`. (FR-001, FR-025, FR-029, SC-001)
- [x] T105 [US4] Implement rollback PowerShell contract helper in `tools/prod/prod_rollback_yoloe_scene.ps1`. (FR-001, FR-025, FR-029, SC-001)
- [x] T106 [US4] Update production benchmark history with decision table and evidence links in `docs/production_inference_benchmark.md`. (FR-026, FR-027, FR-028, FR-029, SC-012, SC-013, SC-016, SC-017)
- [x] T107 [US4] Update feature cycle evidence with production job lineage, rollback proof, figure links, and unavailable metric reasons in `docs/entity/cycles/cycle_014_yoloe_scene_srvl.md`. (FR-026, FR-027, FR-028, FR-029, SC-012, SC-013, SC-016, SC-017, SC-018)
- [x] T108 [US4] Add CI workflow or existing workflow updates for scene contracts, docs, figures, shell syntax, and frontend checks in `.github/workflows/yoloe-scene-srvl.yml`. (FR-027, FR-028, SC-013, SC-018)

**Checkpoint**: US4 can be validated by reviewing a complete evidence package
without relying on local-only, mock-only, or placeholder-only claims.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Cross-story validation, documentation, reproducibility, and final
acceptance gates.

- [x] T109 [P] Update quickstart reproduction commands and evidence expectations in `specs/014-yoloe-scene-srvl/quickstart.md`. (FR-025, FR-027, FR-028, SC-012, SC-013, SC-017, SC-018)
- [x] T110 [P] Update source plan references and remaining gap notes in `docs/yoloe_scene_segmentation_plan.md`. (FR-027, FR-028, SC-018)
- [x] T111 [P] Run and record Mermaid verification for feature docs in `specs/014-yoloe-scene-srvl/quickstart.md`. (FR-027, FR-028, SC-018)
- [x] T112 [P] Run and record documentation reading-order verification in `docs/production_inference_benchmark.md`. (FR-027, FR-028, SC-018)
- [x] T113 [P] Run and record backend PostgreSQL tests for scene unit, integration, contract, and system suites in `docs/entity/cycles/cycle_014_yoloe_scene_srvl.md`. (FR-028, SC-001, SC-002, SC-003, SC-004, SC-005, SC-006, SC-007, SC-008, SC-014, SC-015, SC-016, SC-017, SC-018, SC-019, SC-021, SC-022)
- [x] T114 [P] Run and record frontend type, unit, benchmark, and Playwright checks in `docs/entity/cycles/cycle_014_yoloe_scene_srvl.md`. (FR-020, FR-021, FR-022, FR-023, SC-010, SC-011, SC-014)
- [x] T115 [P] Run and record shell, PowerShell, and Python helper checks in `docs/entity/cycles/cycle_014_yoloe_scene_srvl.md`. (FR-025, FR-027, SC-012, SC-013, SC-018)
- [x] T116 Produce final requirements-to-evidence manifest in `specs/014-yoloe-scene-srvl/evidence_manifest.md`. (FR-027, FR-028, SC-017, SC-018)
- [x] T117 Confirm no hidden xfails, unsupported live enablement, SQLite fallback, untracked CI-required files, mock acceptance artifacts, or local-only production claims in `specs/014-yoloe-scene-srvl/evidence_manifest.md`. (FR-028, SC-018)
- [x] T118 Confirm any async job, vector persistence, or orchestration changes declare stage deadlines, reconciler behavior, idempotency keys, error-ratio thresholds, vector payload guards, and re-run tests in `specs/014-yoloe-scene-srvl/evidence_manifest.md`. (FR-028, SC-019)
- [x] T119 Confirm no worker/thread/concurrency candidate is accepted without governed production benchmark evidence in `specs/014-yoloe-scene-srvl/evidence_manifest.md`. (FR-028, SC-020)
- [x] T120 Run final rollback and disabled-path proof, then link command output in `docs/entity/cycles/cycle_014_yoloe_scene_srvl.md`. (FR-001, FR-029, SC-001)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies.
- **Foundational (Phase 2)**: Depends on setup; blocks all user stories.
- **US1 (Phase 3)**: Depends on foundation.
- **US2 (Phase 4)**: Depends on foundation and can use US1 scene summaries when
  integrated, but its matching/recovery logic remains independently testable.
- **US3 (Phase 5)**: Depends on foundation and can use US1 object summaries
  when integrated, but SRVL math and renderer contracts remain independently
  testable.
- **US4 (Phase 6)**: Depends on foundation; final production evidence depends
  on the selected implemented story scope.
- **Polish (Phase 7)**: Depends on the desired implemented story scope.

### User Story Dependencies

- **US1 (P1)**: MVP and first independently useful increment.
- **US2 (P2)**: May run after foundation using fixture scene objects; full
  integration depends on US1 outputs.
- **US3 (P3)**: May run after foundation using representative scene payloads;
  full integration depends on US1 outputs.
- **US4 (P4)**: Evidence infrastructure can run in parallel with implementation;
  acceptance evidence requires the implemented scope.

### Parallel Opportunities

- Setup tasks T002-T007 can run in parallel.
- Foundational tests T009, T013, T015, T017, T019, T022, T025, T027, and T029
  can run in parallel after their target interfaces are defined.
- US1 validation tasks T033-T040 can be drafted in parallel.
- US2 validation tasks T055-T060 can be drafted in parallel.
- US3 validation tasks T069-T076 can be drafted in parallel.
- US4 validation tasks T089-T097 can be drafted in parallel.
- Cross-cutting validation tasks T109-T115 can run in parallel once docs and
  helpers exist.

---

## Parallel Example: User Story 1

```bash
# Draft US1 validation in parallel:
Task: "T033 prompt-locked export unit tests"
Task: "T035 YOLOE normalizer preservation tests"
Task: "T036 mask artifact round-trip tests"
Task: "T039 non-ROI append-only contradiction tests"

# Implement independent US1 modules in parallel:
Task: "T041 prompt profile resolution"
Task: "T043 runtime normalizer"
Task: "T046 non-ROI union mask builder"
```

## Parallel Example: User Story 3

```bash
# Draft SRVL and frontend validations in parallel:
Task: "T069 SRVL CPU-reference parity tests"
Task: "T072 visualization queue tests"
Task: "T074 frontend optional-scene-field tests"
Task: "T076 Playwright absent-field checks"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1 setup.
2. Complete Phase 2 foundation.
3. Complete US1 prompt-locked export, scene lane, mask artifacts, non-ROI guard,
   disabled-path parity, and focused tests.
4. Stop and validate US1 independently with local contract tests and no
   production acceptance claim.
5. Run production evidence only when the code path and rollback proof are ready.

### Incremental Delivery

1. Ship US1 as the minimum evidence-producing scene lane.
2. Add US2 mismatch recovery as candidate-only, append-only evidence.
3. Add US3 SRVL and visualization without blocking analytics.
4. Complete US4 evidence, figures, docs, rollback, and production benchmark
   before any accepted-scope claim.

### Acceptance Strategy

1. Keep V1 live disabled unless a separate live-specific plan is approved.
2. Keep `YOLOE_SCENE_ENABLED=0`, `YOLOE_SCENE_MISMATCH_RECOVERY=0`,
   `YOLOE_SCENE_RECOVERY_REPASS=0`, and `SRVL_ENABLED=0` as rollback defaults.
3. Treat local Windows, smoke, synthetic, and component probe output as
   hypothesis or contract evidence only.
4. Accept production claims only from native Linux RTX 5090 `combined.mp4`
   evidence with raw metrics, generated figures, unavailable-metric reasons,
   and rollback proof.

---

## Requirements Coverage Index

| Requirement | Primary task coverage |
|---|---|
| FR-001 | T001, T002, T018, T019, T037, T048, T094, T104, T105, T120 |
| FR-002 | T003, T020, T022, T033, T034, T041, T042, T051, T052 |
| FR-003 | T020, T021, T022, T033, T034, T041, T042, T051, T052 |
| FR-004 | T011, T014, T020, T021, T022, T033, T034, T040, T042, T051, T052, T053, T054 |
| FR-005 | T020, T021, T022, T034, T042, T053, T054 |
| FR-006 | T023, T035, T036, T038, T043, T044, T048 |
| FR-007 | T011, T012, T013, T014, T015, T036, T038, T040, T044, T045, T071 |
| FR-008 | T041, T038, T039, T046, T048 |
| FR-009 | T011, T013, T039, T046, T047 |
| FR-010 | T011, T055, T061, T067 |
| FR-011 | T056, T062, T067 |
| FR-012 | T057, T063, T064, T067 |
| FR-013 | T057, T063 |
| FR-014 | T011, T013, T058, T060, T064, T065, T067 |
| FR-015 | T058, T060, T064, T065 |
| FR-016 | T013, T039, T047, T058, T060, T065 |
| FR-017 | T059, T066 |
| FR-018 | T011, T069, T070, T071, T077, T078, T080, T083 |
| FR-019 | T069, T070, T071, T077, T078, T079, T080, T083 |
| FR-020 | T026, T027, T050, T072, T075, T076, T081, T083, T085, T086, T087, T088, T114 |
| FR-021 | T073, T075, T076, T082, T085, T086, T087, T114 |
| FR-022 | T004, T075, T087, T088, T093, T102, T103, T114 |
| FR-023 | T023, T024, T025, T026, T027, T049, T050, T068, T074, T084, T085, T086 |
| FR-024 | T014, T026, T028, T029, T038, T040, T045, T050, T067, T068, T071, T077, T080, T082, T083, T095 |
| FR-025 | T032, T051, T052, T053, T054, T089, T090, T097, T098, T099, T100, T101, T102, T103, T104, T105, T115 |
| FR-026 | T028, T029, T032, T056, T088, T091, T092, T093, T095, T098, T100, T101, T102, T106, T107 |
| FR-027 | T005, T006, T007, T030, T031, T089, T090, T092, T106, T107, T108, T109, T110, T111, T112, T115, T116 |
| FR-028 | T005, T006, T013, T018, T019, T030, T031, T032, T040, T045, T094, T095, T096, T098, T101, T106, T107, T108, T109, T110, T111, T112, T113, T116, T117, T118, T119 |
| FR-029 | T037, T094, T104, T105, T106, T107, T120 |
| FR-030 | T016, T017, T023, T024, T025, T049, T096 |
| FR-031 | T008, T023, T035, T039, T043, T044, T046, T047, T064, T086 |
| FR-032 | T002, T008, T009, T010, T041, T064, T079 |
| FR-033 | T002, T008, T009, T010, T041 |

| Success Criterion | Primary task coverage |
|---|---|
| SC-001 | T037, T094, T104, T105, T120 |
| SC-002 | T020, T021, T022, T033, T034, T041, T042, T051, T052, T053, T054 |
| SC-003 | T011, T014, T038, T040, T045, T048 |
| SC-004 | T014, T015, T035, T036, T038, T040, T043, T044, T045, T048 |
| SC-005 | T055, T056, T058, T059, T061, T062, T064, T067, T068 |
| SC-006 | T057, T058, T060, T063, T064 |
| SC-007 | T013, T039, T046, T047, T060 |
| SC-008 | T069, T070, T071, T077, T078, T079, T080, T083 |
| SC-009 | T070, T078, T083 |
| SC-010 | T004, T073, T075, T076, T082, T085, T086, T087, T093, T102, T103, T114 |
| SC-011 | T026, T027, T050, T072, T073, T075, T081, T082, T083, T114 |
| SC-012 | T028, T029, T032, T088, T091, T095, T097, T098, T100, T102, T106, T107, T109, T115 |
| SC-013 | T014, T015, T030, T031, T092, T097, T101, T106, T107, T109, T115 |
| SC-014 | T023, T024, T025, T027, T049, T050, T068, T074, T084, T085, T086, T114 |
| SC-015 | T016, T017, T023, T024, T025, T049, T096 |
| SC-016 | T028, T029, T095, T100, T106, T107 |
| SC-017 | T005, T011, T012, T014, T015, T028, T029, T040, T045, T071, T096, T100, T106, T107, T116 |
| SC-018 | T002, T005, T006, T009, T012, T018, T019, T032, T089, T090, T094, T096, T108, T109, T110, T111, T112, T113, T115, T117 |
| SC-019 | T013, T045, T058, T060, T065, T113, T118 |
| SC-020 | T119 |
| SC-021 | T023, T035, T036, T043, T044 |
| SC-022 | T002, T008, T009, T010, T041, T064, T079 |
