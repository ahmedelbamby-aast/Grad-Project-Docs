# Tasks: Student Behavior Detection Pyramid

**Input**: Design documents from `/specs/003-student-behavior-pyramid/`
**Prerequisites**: plan.md (required), spec.md (required), research.md, data-model.md, contracts/

**Tests**: Tests are MANDATORY per the Test-in-Loop methodology (Constitution §II). ALL tests (unit, integration, system) MUST be written BEFORE implementation for every user story. Tests MUST be verified to fail (RED) before any implementation code is written.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Backend**: `backend/apps/pipeline/` (primary module), `backend/tests/` (tests)
- **Frontend**: `frontend/src/` (existing — extend only if needed)
- **Docs**: `docs/backend/apps/pipeline/` (per-file documentation)

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Create pipeline configuration, exception hierarchy, base class typing, and test directory structure. These are prerequisites for all user stories.

- [X] T001 Create test directory structure for pipeline unit tests in backend/tests/unit/pipeline/__init__.py
- [X] T002 [P] Create Pydantic v2 pipeline configuration with model paths, thresholds, device selection, and in-memory data structures (FrameMeta, PersonDetectionResult, CroppedStudentImage, BehaviorPrediction, MergedBehaviorRecord) in backend/apps/pipeline/config.py
- [X] T003 [P] Create pipeline-specific exception hierarchy (ModelLoadError, InferenceError, CropError, MergeError, ConfigurationError) in backend/apps/pipeline/exceptions.py
- [X] T004 [P] Extend BasePyramidLayer ABC with proper type hints, lazy model loading interface, and device configuration in backend/apps/pipeline/base.py
- [X] T005 [P] Add PYRAMID_* environment variables to .env.example for model paths, device, confidence thresholds, worker count, and inference timeout
- [X] T006 Add PIPELINE configuration loading to Django settings in backend/config/settings/base.py

**Checkpoint**: Pipeline infrastructure ready — user story implementation can begin

---

## Phase 2: User Story 1 — Detect and Isolate Students in a Video Frame (Priority: P1) 🎯 MVP

**Goal**: Accept a video frame, detect all persons, classify each as student or teacher, and return only student detections with bounding box coordinates. Teacher detections are discarded.

**Independent Test**: Provide a classroom image with both students and a teacher; verify the system returns bounding regions for students only and excludes the teacher.

### Tests for User Story 1 (MANDATORY — Test-in-Loop) 🔴

> **MANDATORY: Write ALL tests FIRST, verify they FAIL (RED) before implementation (Constitution §II)**

- [X] T007 [P] [US1] Unit test for PipelineConfig validation (valid/invalid model paths, thresholds, device) in backend/tests/unit/pipeline/test_config.py
- [X] T008 [P] [US1] Unit test for PersonDetector — mock YOLO model, verify detection output structure, student/teacher classification, confidence filtering, and empty frame handling in backend/tests/unit/pipeline/test_detector.py
- [X] T009 [P] [US1] Unit test for TrackerService — verify tracking ID assignment, unknown fallback, and ID format in backend/tests/unit/pipeline/test_tracker.py
- [X] T010 [P] [US1] Integration test for detection → filtering flow — mock YOLO model, provide frame with mixed students/teachers, verify only student detections pass through in backend/tests/integration/test_pipeline_flow.py

### Implementation for User Story 1

- [X] T011 [US1] Implement PersonDetector with lazy YOLO model loading, person detection, student/teacher classification, teacher filtering, and confidence thresholds in backend/apps/pipeline/detector.py
- [X] T012 [US1] Extend TrackerService with robust tracking ID assignment using detection metadata in backend/apps/pipeline/tracker.py
- [X] T013 [US1] Verify all US1 tests pass (GREEN) and refactor PersonDetector and TrackerService for code quality (constitution §I)

**Checkpoint**: PersonDetector correctly identifies students vs teachers. Can be tested independently by providing a frame and checking that only student bounding boxes are returned.

---

## Phase 3: User Story 2 — Crop Individual Students from Frame (Priority: P1)

**Goal**: Given detected student bounding boxes and the original frame, crop each student into an individual image ready for behavior classification. Handle edge-of-frame boundaries without distortion.

**Independent Test**: Provide a frame and a set of known student coordinates; verify the system produces one correctly-sized cropped image per student.

### Tests for User Story 2 (MANDATORY — Test-in-Loop) 🔴

> **MANDATORY: Write ALL tests FIRST, verify they FAIL (RED) before implementation (Constitution §II)**

- [X] T014 [P] [US2] Unit test for FrameCropper — verify correct cropping from numpy array, boundary clamping for edge cases, empty bbox handling, and overlapping student regions in backend/tests/unit/pipeline/test_cropper.py

### Implementation for User Story 2

- [X] T015 [US2] Implement FrameCropper with numpy array slicing, boundary clamping (clamp x1/y1/x2/y2 to frame dimensions), and per-student CroppedStudentImage output in backend/apps/pipeline/cropper.py
- [X] T016 [US2] Verify all US2 tests pass (GREEN) and refactor FrameCropper for code quality

**Checkpoint**: FrameCropper produces one correctly bounded crop per detected student. Can be tested independently with a synthetic frame and predefined bounding boxes.

---

## Phase 4: User Story 3 — Classify Student Behaviors Across Multiple Models (Priority: P1)

**Goal**: Send each cropped student image simultaneously to four binary behavior classification models (posture, horizontal gaze, depth gaze, vertical gaze). Each model returns its classification label and confidence score.

**Independent Test**: Provide a single cropped student image to each model independently; verify each returns one of its two class labels with a confidence score.

### Tests for User Story 3 (MANDATORY — Test-in-Loop) 🔴

> **MANDATORY: Write ALL tests FIRST, verify they FAIL (RED) before implementation (Constitution §II)**

- [X] T017 [P] [US3] Unit test for PostureLayer — mock YOLO classifier, verify standing/sitting classification, confidence output, and model failure handling in backend/tests/unit/pipeline/test_posture.py
- [X] T018 [P] [US3] Unit test for HorizontalGazeLayer — mock YOLO classifier, verify looking_left/looking_right classification in backend/tests/unit/pipeline/test_horizontal_gaze.py
- [X] T019 [P] [US3] Unit test for DepthGazeLayer — mock YOLO classifier, verify looking_forward/looking_backward classification in backend/tests/unit/pipeline/test_depth_gaze.py
- [X] T020 [P] [US3] Unit test for VerticalGazeLayer — mock YOLO classifier, verify looking_up/looking_down classification in backend/tests/unit/pipeline/test_vertical_gaze.py

### Implementation for User Story 3

- [X] T021 [P] [US3] Rewrite PostureLayer with real YOLO classification inference, lazy model loading via double-checked locking, and BehaviorPrediction output in backend/apps/pipeline/layers/posture.py
- [X] T022 [P] [US3] Rewrite HorizontalGazeLayer with real YOLO classification inference and lazy model loading in backend/apps/pipeline/layers/horizontal_gaze.py
- [X] T023 [P] [US3] Rewrite DepthGazeLayer with real YOLO classification inference and lazy model loading in backend/apps/pipeline/layers/depth_gaze.py
- [X] T024 [P] [US3] Rewrite VerticalGazeLayer with real YOLO classification inference and lazy model loading in backend/apps/pipeline/layers/vertical_gaze.py
- [X] T025 [US3] Update layer registry exports in backend/apps/pipeline/layers/__init__.py
- [X] T026 [US3] Verify all US3 tests pass (GREEN) and refactor all four layers for code quality

**Checkpoint**: Each behavior layer independently classifies a cropped student image. Can be tested by providing a single crop to each model and checking the binary output.

---

## Phase 5: User Story 4 — Merge All Predictions for Each Student (Priority: P2)

**Goal**: Collect predictions from all four behavior models for each student in a frame, merge them into a single unified behavior record per student, and handle partial results when models fail. Feed merged results through the rule engine and persist to database.

**Independent Test**: Provide mock prediction outputs from the four models for a set of students; verify the merger produces one consolidated record per student with all four behavior dimensions and correct completeness flags.

### Tests for User Story 4 (MANDATORY — Test-in-Loop) 🔴

> **MANDATORY: Write ALL tests FIRST, verify they FAIL (RED) before implementation (Constitution §II)**

- [X] T027 [P] [US4] Unit test for PredictionMerger — verify merging of complete results, partial results (1-3 models fail), all-fail scenario, and completeness flag logic in backend/tests/unit/pipeline/test_merger.py
- [X] T028 [P] [US4] Unit test for extended RuleEngine — verify anomaly evaluation with all 4 predictions, partial predictions, and edge case gaze patterns in backend/tests/unit/pipeline/test_rule_engine.py
- [X] T029 [P] [US4] Unit test for rewritten PipelineService — verify full pyramid orchestration (detect → filter → crop → classify in parallel → merge → rule engine), mock all models, test empty frame and partial failure scenarios in backend/tests/unit/pipeline/test_services.py
- [X] T030 [P] [US4] Integration test for full pipeline flow — mock YOLO models, verify Detection and PyramidPrediction database records are created correctly for multi-student frame in backend/tests/integration/test_pipeline_flow.py (extend existing file)
- [X] T031 [US4] System test for end-to-end frame processing — mock YOLO models, verify frame input produces database records and WebSocket message broadcast in backend/tests/system/test_frame_e2e.py

### Implementation for User Story 4

- [X] T032 [US4] Implement PredictionMerger with ThreadPoolExecutor-based parallel layer execution, per-model timeout, partial result collection, completeness tracking, and MergedBehaviorRecord output in backend/apps/pipeline/merger.py
- [X] T033 [US4] Extend RuleEngine with evaluation of all 4 behavior dimensions, partial prediction handling, and configurable anomaly rules in backend/apps/pipeline/rule_engine.py
- [X] T034 [US4] Rewrite PipelineService.process_frame() to orchestrate full pyramid: PersonDetector → filter teachers → FrameCropper → parallel behavior layers via PredictionMerger → RuleEngine → persist Detection + PyramidPrediction records → broadcast WebSocket events in backend/apps/pipeline/services.py
- [X] T035 [US4] Verify all US4 tests pass (GREEN) and refactor merger, rule engine, and services for code quality

**Checkpoint**: Full pyramid processes a frame end-to-end. Merged records include all 4 behavior dimensions with completeness flags. Partial failures are handled gracefully per FR-014.

---

## Phase 6: User Story 5 — Model Inventory, Export, Benchmarking, and Deployment Selection (Priority: P1)

**Goal**: Scan root `models/`, detect available artifacts, export missing runtime formats from `.pt` in background jobs, benchmark each model-format on required runtime targets using `Raw Data/` replay streams, persist results, generate explainability and figure artifacts, and compute a deployment matrix.

**Independent Test**: Place mixed model artifacts in `models/`, run the model lifecycle pipeline, and verify: (1) inventory reflects available formats, (2) missing formats are exported from `.pt` when possible, (3) benchmark results are persisted, (4) deployment matrix recommendations are produced, and (5) Triton-path validation results are logged.

### Tests for User Story 5 (MANDATORY — Test-in-Loop) 🔴

> **MANDATORY: Write ALL tests FIRST, verify they FAIL (RED) before implementation (Constitution §II)**

- [X] T036 [P] [US5] Unit test for model inventory scanner extension detection (`.pt`, `.onnx`, `.engine`/`.trt`, OpenVINO `.xml`+`.bin`) in backend/tests/unit/pipeline/test_model_lifecycle/test_inventory.py
- [X] T037 [P] [US5] Unit test for runtime capability snapshot (OS/CPU/GPU/CUDA/TensorRT/OpenVINO detection) in backend/tests/unit/pipeline/test_model_lifecycle/test_capabilities.py
- [X] T038 [P] [US5] Unit test for export decision policy (missing artifacts + `.pt` fallback + no-source error state) in backend/tests/unit/pipeline/test_model_lifecycle/test_export_orchestrator.py
- [X] T039 [P] [US5] Unit test for benchmark scoring and deployment matrix ranking in backend/tests/unit/pipeline/test_model_lifecycle/test_deployment_matrix.py
- [X] T040 [P] [US5] Integration test for sequential scan -> export -> benchmark pipeline with persisted metrics in backend/tests/integration/test_model_lifecycle_flow.py
- [X] T041 [US5] System test for Dockerized benchmark flow including Triton validation path in backend/tests/system/test_model_ops_e2e.py

### Implementation for User Story 5

- [X] T042 [US5] Implement model artifact scan and compatibility mapping in backend/apps/pipeline/model_lifecycle/inventory.py
- [X] T043 [US5] Implement runtime capability probing and snapshot persistence in backend/apps/pipeline/model_lifecycle/capabilities.py
- [X] T044 [US5] Implement background export orchestration (`.pt` -> ONNX/TensorRT/OpenVINO) with structured logs in backend/apps/pipeline/model_lifecycle/export_orchestrator.py and backend/apps/pipeline/model_lifecycle/export_worker.py
- [X] T045 [US5] Implement benchmark runner using replay streams from `Raw Data/` with latency/throughput/warmup/memory/error metrics in backend/apps/pipeline/model_lifecycle/benchmark_runner.py
- [X] T046 [US5] Implement deployment matrix scoring and recommendation persistence in backend/apps/pipeline/model_lifecycle/deployment_matrix.py
- [X] T047 [US5] Implement explainability artifact generation and benchmark figure generation linkage in backend/apps/pipeline/model_lifecycle/explainability.py and backend/apps/pipeline/model_lifecycle/visualizations.py
- [X] T048 [US5] Add internal APIs for inventory/export/benchmark/matrix status in backend/apps/pipeline/services.py and backend/apps/pipeline/models.py
- [X] T049 [US5] Add Docker runtime definitions and integration notes for CPU/GPU/TensorRT/OpenVINO/Triton execution in docker-compose.dev.yml and specs/003-student-behavior-pyramid/quickstart.md
- [X] T050 [US5] Implement Triton Inference Server adapter with config.pbtxt generation, model repository structure, and health check validation in backend/apps/pipeline/model_lifecycle/triton_adapter.py
- [X] T051 [US5] Verify all US5 tests pass (GREEN) and refactor model lifecycle modules for code quality

**Checkpoint**: Model lifecycle is operational and produces reproducible benchmark evidence with deployment recommendations.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Documentation, diagrams, cross-links, final quality sweep, and delivery sign-off per constitution.

- [X] T052 [P] Create/update docs/backend/apps/pipeline/config.md with diagrams and cross-links
- [X] T053 [P] Create/update docs/backend/apps/pipeline/exceptions.md with diagrams and cross-links
- [X] T054 [P] Create/update docs/backend/apps/pipeline/base.md with diagrams and cross-links
- [X] T055 [P] Create/update docs/backend/apps/pipeline/detector.md with diagrams and cross-links
- [X] T056 [P] Create/update docs/backend/apps/pipeline/cropper.md with diagrams and cross-links
- [X] T057 [P] Create/update docs/backend/apps/pipeline/merger.md with diagrams and cross-links
- [X] T058 [P] Create/update docs/backend/apps/pipeline/model_lifecycle/inventory.md with diagrams and cross-links
- [X] T059 [P] Create/update docs/backend/apps/pipeline/model_lifecycle/capabilities.md with diagrams and cross-links
- [X] T060 [P] Create/update docs/backend/apps/pipeline/model_lifecycle/export_orchestrator.md with diagrams and cross-links
- [X] T061 [P] Create/update docs/backend/apps/pipeline/model_lifecycle/export_worker.md with diagrams and cross-links
- [X] T062 [P] Create/update docs/backend/apps/pipeline/model_lifecycle/benchmark_orchestrator.md with diagrams and cross-links
- [X] T063 [P] Create/update docs/backend/apps/pipeline/model_lifecycle/benchmark_runner.md with diagrams and cross-links
- [X] T064 [P] Create/update docs/backend/apps/pipeline/model_lifecycle/metrics.md with diagrams and cross-links
- [X] T065 [P] Create/update docs/backend/apps/pipeline/model_lifecycle/deployment_matrix.md with diagrams and cross-links
- [X] T066 [P] Create/update docs/backend/apps/pipeline/model_lifecycle/visualizations.md with diagrams and cross-links
- [X] T067 [P] Create/update docs/backend/apps/pipeline/model_lifecycle/explainability.md with diagrams and cross-links
- [X] T068 [P] Create/update docs/backend/apps/pipeline/model_lifecycle/triton_adapter.md with diagrams and cross-links
- [X] T069 [P] Create/update docs/backend/apps/pipeline/services.md with diagrams and cross-links
- [X] T070 [P] Create/update docs/backend/apps/pipeline/rule_engine.md with diagrams and cross-links
- [X] T071 [P] Create/update docs/backend/apps/pipeline/tracker.md with diagrams and cross-links
- [X] T072 [P] Create/update docs/backend/apps/pipeline/layers/posture.md with diagrams and cross-links
- [X] T073 [P] Create/update docs/backend/apps/pipeline/layers/horizontal_gaze.md with diagrams and cross-links
- [X] T074 [P] Create/update docs/backend/apps/pipeline/layers/depth_gaze.md with diagrams and cross-links
- [X] T075 [P] Create/update docs/backend/apps/pipeline/layers/vertical_gaze.md with diagrams and cross-links
- [X] T076 [P] Create/update docs/backend/apps/pipeline/layers/__init__.md with diagrams and cross-links
- [X] T077 [P] Create/update docs/backend/apps/pipeline/model_lifecycle/__init__.md with diagrams and cross-links
- [X] T078 [P] Create/update backend/apps/pipeline/README.md with module overview, public API, dependencies, and usage examples
- [X] T079 Verify ALL diagram types used in every docs/ .md file (constitution §Mandatory Diagram Types)
- [X] T080 Verify every diagram has introduction + detailed explanation (constitution §Diagram Explanation Requirements)
- [X] T081 Verify every .md file has Related Documents cross-links (constitution §.md File Interconnection)
- [X] T082 Validate all cross-reference links resolve to existing files — no broken links
- [X] T083 Run full test suite (unit + integration + system) and verify ≥80% line coverage
- [X] T084 Security review: no hardcoded secrets, model paths via env vars, PII-free logs, tracking IDs anonymized
- [X] T085 Run quickstart.md smoke test validation
- [X] T086 Delivery sign-off per constitution §VII (7-step protocol): lint, type-check, tests, security, spec compliance, documentation, sign-off summary

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately
- **US1 (Phase 2)**: Depends on Setup (Phase 1) completion — detector needs config.py, base.py, exceptions.py
- **US2 (Phase 3)**: Depends on Setup (Phase 1) completion — cropper needs config.py data structures. Independently testable (does NOT require US1 implementation).
- **US3 (Phase 4)**: Depends on Setup (Phase 1) completion — layers need base.py, config.py. Independently testable (does NOT require US1 or US2).
- **US4 (Phase 5)**: Depends on US1, US2, US3 — PipelineService orchestrates all upstream components. Merger integrates all layers.
- **US5 (Phase 6)**: Depends on Setup and US4 core pipeline outputs — model lifecycle benchmarking reuses pipeline and persistence foundations.
- **Polish (Phase 7)**: Depends on all user stories being complete

### User Story Dependencies

- **US1 (P1)**: After Setup → Independently testable with a frame input
- **US2 (P1)**: After Setup → Independently testable with frame + bbox coordinates
- **US3 (P1)**: After Setup → Independently testable with cropped student images
- **US4 (P2)**: After US1 + US2 + US3 → Integrates all components into full pipeline
- **US5 (P1)**: After US4 + Setup → Adds model inventory/export/benchmarking and deployment selection

### Within Each User Story

- Tests MUST be written and verified to FAIL before implementation
- Config/data structures before service classes
- Service classes before integration
- Core implementation before quality refactor
- Story complete before moving to next

### Parallel Opportunities

- **Setup**: T002, T003, T004, T005 can all run in parallel (different files)
- **US1 tests**: T007, T008, T009, T010 can all run in parallel
- **US2/US3 stories**: US2 and US3 can run in parallel after Setup (independent of each other)
- **US3 tests**: T017, T018, T019, T020 can all run in parallel (different test files)
- **US3 implementation**: T021, T022, T023, T024 can all run in parallel (different layer files)
- **US4 tests**: T027, T028, T029, T030 can all run in parallel
- **US5 tests**: T036, T037, T038, T039, T040 can all run in parallel
- **Polish docs**: T051–T069 can all run in parallel (different doc files)

---

## Parallel Example: User Story 3

```bash
# Launch all layer tests in parallel (Test-in-Loop RED phase):
Task T017: "Unit test for PostureLayer in backend/tests/unit/pipeline/test_posture.py"
Task T018: "Unit test for HorizontalGazeLayer in backend/tests/unit/pipeline/test_horizontal_gaze.py"
Task T019: "Unit test for DepthGazeLayer in backend/tests/unit/pipeline/test_depth_gaze.py"
Task T020: "Unit test for VerticalGazeLayer in backend/tests/unit/pipeline/test_vertical_gaze.py"

# Then launch all layer implementations in parallel (GREEN phase):
Task T021: "Rewrite PostureLayer in backend/apps/pipeline/layers/posture.py"
Task T022: "Rewrite HorizontalGazeLayer in backend/apps/pipeline/layers/horizontal_gaze.py"
Task T023: "Rewrite DepthGazeLayer in backend/apps/pipeline/layers/depth_gaze.py"
Task T024: "Rewrite VerticalGazeLayer in backend/apps/pipeline/layers/vertical_gaze.py"
```

---

## Parallel Example: User Stories 2 & 3 (Cross-Story Parallelism)

```bash
# After Setup completes, US2 and US3 can start simultaneously:

# Developer A: User Story 2
Task T014: "Unit test for FrameCropper"
Task T015: "Implement FrameCropper"

# Developer B: User Story 3
Task T017-T020: "Unit tests for all 4 behavior layers"
Task T021-T024: "Implement all 4 behavior layers"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: User Story 1 — PersonDetector + TrackerService
3. **STOP and VALIDATE**: Test with a classroom image — students detected, teacher excluded
4. Deploy/demo if ready — basic detection proves the pipeline foundation works

### Incremental Delivery

1. Complete Setup → Infrastructure ready
2. Add US1 (detect students) → Test independently → Demo (MVP!)
3. Add US2 (crop students) + US3 (classify behaviors) in parallel → Test independently → Demo
4. Add US4 (merge predictions) → Full pipeline functional → Test end-to-end → Demo
5. Add US5 (inventory/export/benchmark/matrix) → Production deployment evidence ready
6. Polish (docs, diagrams, sign-off) → Production ready

### Parallel Team Strategy

With multiple developers:

1. Team completes Setup together
2. Once Setup is done:
   - Developer A: User Story 1 (PersonDetector)
   - Developer B: User Story 2 (FrameCropper) + User Story 3 (all 4 behavior layers)
3. After US1, US2, US3 complete:
   - Any developer: User Story 4 (PredictionMerger + PipelineService orchestration)
4. After US4 complete:
   - Developer C: User Story 5 (model lifecycle + benchmarking + deployment matrix)
5. All developers: Polish phase (docs in parallel)

---

## Notes

- [P] tasks = different files, no dependencies on incomplete tasks
- [Story] label maps task to specific user story for traceability
- Each user story is independently testable with mocked upstream inputs
- Tests MUST fail before implementation (Test-in-Loop RED → GREEN → REFACTOR)
- Commit after each task or logical group (constitution Supreme Directive)
- Every source file change MUST include its corresponding docs/ .md file
- Module README.md MUST be created/updated after finishing work on a module
- Stop at any checkpoint to validate story independently
- YOLO models are NOT thread-safe — each layer uses its own model instance (research R-002, R-005)
- All model paths come from Pydantic config with PYRAMID_* env prefix (research R-006)
