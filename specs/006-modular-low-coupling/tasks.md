# Tasks: Modular Design and Low Coupling Hardening

**Input**: Design documents from `/specs/006-modular-low-coupling/`
**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [quickstart.md](quickstart.md), [contracts/](contracts/)

**Tests**: Mandatory. Write tests first, verify every new test fails in the RED phase before implementation, and preserve 100% line and branch coverage for affected modules or record a time-boxed exception in both the evidence pack and `plan.md` Complexity Tracking. Inference, tracking, video-processing, and overlay tests must use real model weights and real raw test data.

**Organization**: Tasks are grouped by user story so each story can be implemented and validated independently after the foundational phase.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel because it touches different files and does not depend on an incomplete task
- **[Story]**: Maps to a user story from [spec.md](spec.md)
- Every task includes exact file paths
- Every source, test, or script file task names its required documentation counterpart in `docs/`

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Capture the current baseline and create shared evidence locations before any refactor work begins.

- [x] T001 Create the evidence directory index for this feature in specs/006-modular-low-coupling/evidence/evidence-pack.md
- [x] T002 [P] Capture current backend unit, integration, contract, and system baseline command results in specs/006-modular-low-coupling/evidence/baseline/backend-before.md
- [x] T003 [P] Capture current frontend unit, e2e, build, lint, and type-check baseline command results in specs/006-modular-low-coupling/evidence/baseline/frontend-before.md
- [x] T004 [P] Inventory real model weights and real raw/live/offline media test assets in specs/006-modular-low-coupling/evidence/real-data-assets.md
- [x] T005 [P] Create the initial 100% line and branch coverage exception register in specs/006-modular-low-coupling/evidence/coverage-exceptions.md and document that every approved exception must also update specs/006-modular-low-coupling/plan.md Complexity Tracking with owner, expiry, and removal plan
- [x] T006 [P] Document official Ultralytics Predict and Track reference links for inference and tracking validation in specs/006-modular-low-coupling/evidence/ultralytics-reference.md
- [x] T007 Record the pre-refactor full delivered baseline checklist in specs/006-modular-low-coupling/evidence/baseline/full-delivered-baseline.md

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Establish governance artifacts and automated gates required before user story implementation.

**Critical**: No user story work can begin until this phase is complete.

- [x] T008 Create the module ownership and boundary map for all backend, frontend, runtime, and deployment boundaries in docs/architecture/module-boundary-map.md
- [x] T009 Create the coupling-risk register with severity, owner, mitigation, verification, and exception eligibility fields in docs/architecture/coupling-risk-register.md
- [x] T010 Create the compatibility contract index for frontend-backend, backend-inference, live/offline, and deployment contracts in docs/architecture/compatibility-contracts.md
- [x] T011 Create the runtime scenario matrix for live-stream, offline-video, and non-video-dashboard flows in docs/architecture/runtime-scenario-matrix.md
- [x] T012 Create the documentation diagram coverage register for existing and incoming Markdown files in docs/architecture/documentation-diagram-coverage.md
- [x] T013 [P] Add contract tests for module boundary records in backend/tests/contract/test_module_boundary_contracts.py and docs/backend/tests/contract/test_module_boundary_contracts.md
- [x] T014 [P] Add contract tests for coupling-risk and refactor-exception records in backend/tests/contract/test_coupling_risk_register.py and docs/backend/tests/contract/test_coupling_risk_register.md
- [x] T015 [P] Add contract tests for runtime scenario evidence records in backend/tests/contract/test_runtime_scenario_contracts.py and docs/backend/tests/contract/test_runtime_scenario_contracts.md
- [x] T016 [P] Add documentation coverage tests for Mermaid diagram classes and cross-links in backend/tests/unit/docs/test_documentation_diagram_coverage.py and docs/backend/tests/unit/docs/test_documentation_diagram_coverage.md
- [x] T017 [P] Add frontend contract tests for typed API and store boundary assumptions in frontend/tests/unit/api/modular_contracts.test.ts and docs/frontend/tests/unit/api/modular_contracts.test.md
- [x] T018 Add the shared boundary record schema and validation helpers in backend/core/boundaries.py and docs/backend/core/boundaries.md
- [x] T019 Add the shared evidence record schema and validation helpers in backend/core/evidence.py and docs/backend/core/evidence.md
- [x] T020 Add the CI boundary verification script in scripts/ci/verify_module_boundaries.py and docs/scripts/ci/verify_module_boundaries.md
- [x] T021 Add the CI documentation diagram verification script in scripts/ci/verify_docs_diagrams.py and docs/scripts/ci/verify_docs_diagrams.md
- [x] T022 Update the release gate to include boundary, coupling, evidence, real-data, and diagram checks in scripts/ci/verify_release_gate.py and docs/scripts/ci/verify_release_gate.md
- [x] T023 Run every new foundational test from T013-T017 and record RED failure evidence before implementation changes in specs/006-modular-low-coupling/evidence/foundation-red-test-status.md
- [x] T024 Run the pre-existing foundational baseline checks separately and record pass/fail status in specs/006-modular-low-coupling/evidence/foundation-existing-baseline-status.md

**Checkpoint**: Foundation ready. User story phases may start after T008-T024 are complete.

---

## Phase 3: User Story 1 - Preserve Existing User Workflows During Refactor (Priority: P1) MVP

**Goal**: Preserve live monitoring, offline video, auth, exams, rooms/cameras, sessions, anomalies, exports, health, settings, and dashboard navigation while modular boundaries are introduced.

**Independent Test**: Run the current live monitoring workflow, offline video workflow, and non-video dashboard workflows before and after modularization and compare user-visible outcomes, statuses, overlays, tracking IDs, reports, and navigation state.

### Tests for User Story 1 (Mandatory)

- [x] T025 [P] [US1] Add full delivered backend baseline regression tests in backend/tests/integration/test_full_delivered_baseline.py and docs/backend/tests/integration/test_full_delivered_baseline.md
- [x] T026 [P] [US1] Add live-stream real-data equivalence system tests with real model weights and raw/live media in backend/tests/system/test_live_stream_modular_equivalence.py and docs/backend/tests/system/test_live_stream_modular_equivalence.md
- [x] T027 [P] [US1] Add offline-video real-data equivalence system tests with real model weights and raw video data in backend/tests/system/test_offline_video_modular_equivalence.py and docs/backend/tests/system/test_offline_video_modular_equivalence.md
- [x] T028 [P] [US1] Add frontend full baseline e2e tests for login, dashboard, live camera, recordings, anomalies, health, settings, and exports in frontend/tests/e2e/full-baseline.spec.ts and docs/frontend/tests/e2e/full-baseline.spec.md
- [x] T029 [P] [US1] Add frontend performance regression tests for dashboard navigation and stream state propagation in frontend/tests/integration/modular-performance.test.ts and docs/frontend/tests/integration/modular-performance.test.md

### Implementation for User Story 1

- [x] T030 [US1] Add reusable workflow baseline helpers for backend regression comparisons in backend/tests/utils/workflow_baseline.py and docs/backend/tests/utils/workflow_baseline.md
- [x] T031 [US1] Preserve live camera and session behavior while routing through documented services in backend/apps/cameras/services.py, docs/backend/apps/cameras/services.md, backend/apps/sessions/services.py, and docs/backend/apps/sessions/services.md
- [x] T032 [US1] Preserve detection and tracking overlays while routing through documented service outputs in backend/apps/detections/services/detection_service.py, docs/backend/apps/detections/services/detection_service.md, backend/apps/tracking/services/tracking_service.py, and docs/backend/apps/tracking/services/tracking_service.md
- [x] T033 [US1] Preserve offline video job status, inference results, and playback overlay behavior in backend/apps/video_analysis/tasks.py, docs/backend/apps/video_analysis/tasks.md, backend/apps/video_analysis/services/inference_orchestrator.py, and docs/backend/apps/video_analysis/services/inference_orchestrator.md
- [x] T034 [US1] Preserve anomaly, recording, export, and health workflow behavior through documented services in backend/apps/anomalies/services.py, docs/backend/apps/anomalies/services.md, backend/apps/recordings/services.py, docs/backend/apps/recordings/services.md, backend/apps/exports/services.py, docs/backend/apps/exports/services.md, backend/apps/health/services.py, and docs/backend/apps/health/services.md
- [x] T035 [US1] Preserve frontend API and store state transitions for baseline workflows in frontend/src/api/client.ts, docs/frontend/src/api/client.md, frontend/src/stores/cameraStore.ts, docs/frontend/src/stores/cameraStore.md, frontend/src/stores/uploadStore.ts, docs/frontend/src/stores/uploadStore.md, frontend/src/stores/anomalyStore.ts, docs/frontend/src/stores/anomalyStore.md, frontend/src/stores/healthStore.ts, and docs/frontend/src/stores/healthStore.md
- [x] T036 [US1] Preserve live, offline, and dashboard page interactions after API/store boundary changes in frontend/src/pages/CameraFeedPage.tsx, docs/frontend/src/pages/CameraFeedPage.md, frontend/src/pages/VideoAnalysisPage.tsx, docs/frontend/src/pages/VideoAnalysisPage.md, frontend/src/pages/RecordingDetailPage.tsx, docs/frontend/src/pages/RecordingDetailPage.md, frontend/src/pages/DashboardPage.tsx, docs/frontend/src/pages/DashboardPage.md, frontend/src/pages/HealthPage.tsx, and docs/frontend/src/pages/HealthPage.md
- [x] T037 [US1] Update baseline workflow documentation with Mermaid live/offline/non-video flow diagrams in docs/architecture/runtime-scenario-matrix.md
- [x] T038 [US1] Record post-refactor live/offline/non-video baseline evidence in specs/006-modular-low-coupling/evidence/baseline/us1-after.md

**Checkpoint**: User Story 1 is independently testable and is the MVP completion gate.

---

## Phase 4: User Story 2 - Define Clear Module Ownership and Boundaries (Priority: P1)

**Goal**: Give each major capability a documented owner, responsibilities, public inputs, public outputs, consumers, dependencies, forbidden dependencies, and failure behavior.

**Independent Test**: Review the module boundary map and verify every current capability has one primary owner and documented consumer-facing contracts.

### Tests for User Story 2 (Mandatory)

- [x] T039 [P] [US2] Add backend boundary declaration tests for accounts, audit, exams, cameras, sessions, detections, pipeline, tracking, video_analysis, anomalies, recordings, exports, and health in backend/tests/unit/test_backend_boundary_declarations.py and docs/backend/tests/unit/test_backend_boundary_declarations.md
- [x] T040 [P] [US2] Add frontend boundary declaration tests for API clients, auth state, live monitoring UI, offline video UI, anomaly UI, recording/export UI, health/settings UI, and shared components in frontend/tests/unit/boundaries/frontend-boundaries.test.ts and docs/frontend/tests/unit/boundaries/frontend-boundaries.test.md
- [x] T041 [P] [US2] Add dependency direction tests that fail on forbidden cross-boundary imports in backend/tests/contract/test_dependency_direction.py and docs/backend/tests/contract/test_dependency_direction.md
- [x] T042 [P] [US2] Add documentation alignment tests that compare boundary declarations to docs/architecture/module-boundary-map.md in backend/tests/integration/test_boundary_docs_alignment.py and docs/backend/tests/integration/test_boundary_docs_alignment.md

### Implementation for User Story 2

- [x] T043 [P] [US2] Add boundary declarations for accounts, audit, and exams in backend/apps/accounts/boundary.py, docs/backend/apps/accounts/boundary.md, backend/apps/audit/boundary.py, docs/backend/apps/audit/boundary.md, backend/apps/exams/boundary.py, and docs/backend/apps/exams/boundary.md
- [x] T044 [P] [US2] Add boundary declarations for cameras, sessions, and detections in backend/apps/cameras/boundary.py, docs/backend/apps/cameras/boundary.md, backend/apps/sessions/boundary.py, docs/backend/apps/sessions/boundary.md, backend/apps/detections/boundary.py, and docs/backend/apps/detections/boundary.md
- [x] T045 [P] [US2] Add boundary declarations for pipeline, tracking, and video_analysis in backend/apps/pipeline/boundary.py, docs/backend/apps/pipeline/boundary.md, backend/apps/tracking/boundary.py, docs/backend/apps/tracking/boundary.md, backend/apps/video_analysis/boundary.py, and docs/backend/apps/video_analysis/boundary.md
- [x] T046 [P] [US2] Add boundary declarations for anomalies, recordings, exports, and health in backend/apps/anomalies/boundary.py, docs/backend/apps/anomalies/boundary.md, backend/apps/recordings/boundary.py, docs/backend/apps/recordings/boundary.md, backend/apps/exports/boundary.py, docs/backend/apps/exports/boundary.md, backend/apps/health/boundary.py, and docs/backend/apps/health/boundary.md
- [x] T047 [P] [US2] Add frontend boundary declarations and public contract types in frontend/src/types/boundaries.ts and docs/frontend/src/types/boundaries.md
- [x] T048 [US2] Publish all backend and frontend boundary records in docs/architecture/module-boundary-map.md
- [x] T049 [US2] Publish allowed and forbidden dependency directions in docs/architecture/compatibility-contracts.md
- [x] T050 [US2] Update module README files with purpose, responsibilities, public inputs, public outputs, consumers, dependencies, and failure behavior in backend/apps/cameras/README.md, backend/apps/sessions/README.md, backend/apps/pipeline/README.md, backend/apps/tracking/README.md, backend/apps/video_analysis/README.md, backend/apps/anomalies/README.md, and backend/apps/health/README.md
- [x] T051 [US2] Record boundary review evidence and reviewer notes in specs/006-modular-low-coupling/evidence/us2-boundary-review.md

**Checkpoint**: User Story 2 is independently testable through boundary declarations, docs alignment, and dependency direction tests.

---

## Phase 5: User Story 3 - Change One Module Without Cascading Regressions (Priority: P2)

**Goal**: Make representative module changes local to stable contracts, including inference provider changes, tracking rule changes, and frontend data source changes.

**Independent Test**: Swap or simulate a representative inference provider, tracking rule, and dashboard data source through documented contracts and prove unrelated workflows remain unchanged.

### Tests for User Story 3 (Mandatory)

- [x] T052 [P] [US3] Add inference provider swap contract tests with real model route expectations in backend/tests/integration/test_modular_inference_contract.py and docs/backend/tests/integration/test_modular_inference_contract.md
- [x] T053 [P] [US3] Add tracking rule isolation tests that prove unrelated modules keep stable inputs and outputs in backend/tests/unit/tracking/test_modular_contract_isolation.py and docs/backend/tests/unit/tracking/test_modular_contract_isolation.md
- [x] T054 [P] [US3] Add frontend API contract tests for dashboard data source swaps in frontend/tests/unit/api/backend-contracts.test.ts and docs/frontend/tests/unit/api/backend-contracts.test.md
- [x] T055 [P] [US3] Add failure and degraded-state contract tests for unavailable module behavior in backend/tests/integration/test_modular_failure_boundaries.py and docs/backend/tests/integration/test_modular_failure_boundaries.md
- [x] T056 [P] [US3] Add native Linux deployment contract tests proving production paths do not require Docker in backend/tests/contract/test_native_linux_deployment_contract.py and docs/backend/tests/contract/test_native_linux_deployment_contract.md

### Implementation for User Story 3

- [x] T057 [US3] Add stable inference contract interfaces and schema exports in backend/apps/pipeline/contracts.py, docs/backend/apps/pipeline/contracts.md, backend/apps/pipeline/schemas/triton.py, and docs/backend/apps/pipeline/schemas/triton.md
- [x] T058 [US3] Refactor inference client factory and Triton client wiring to depend on the stable contract in backend/apps/pipeline/services/base_inference_client.py, docs/backend/apps/pipeline/services/base_inference_client.md, backend/apps/pipeline/services/inference_client_factory.py, docs/backend/apps/pipeline/services/inference_client_factory.md, backend/apps/pipeline/services/triton_client.py, and docs/backend/apps/pipeline/services/triton_client.md
- [x] T059 [US3] Refactor offline orchestration to consume pipeline contracts instead of pipeline internals in backend/apps/video_analysis/services/inference_orchestrator.py and docs/backend/apps/video_analysis/services/inference_orchestrator.md
- [x] T060 [US3] Refactor tracking service and rendering outputs behind a stable tracking contract in backend/apps/tracking/services/tracking_service.py, docs/backend/apps/tracking/services/tracking_service.md, backend/apps/tracking/tracker.py, docs/backend/apps/tracking/tracker.md, backend/apps/tracking/rendering.py, and docs/backend/apps/tracking/rendering.md
- [x] T061 [US3] Refactor detection and anomaly consumers to use documented detection and tracking outputs in backend/apps/detections/views.py, docs/backend/apps/detections/views.md, backend/apps/detections/consumers.py, docs/backend/apps/detections/consumers.md, backend/apps/anomalies/consumers.py, docs/backend/apps/anomalies/consumers.md, backend/apps/anomalies/services.py, and docs/backend/apps/anomalies/services.md
- [x] T062 [US3] Add frontend API response contract types and data-source adapters in frontend/src/types/api.ts, docs/frontend/src/types/api.md, frontend/src/api/detections.ts, docs/frontend/src/api/detections.md, frontend/src/api/videoAnalysis.ts, docs/frontend/src/api/videoAnalysis.md, frontend/src/api/dashboard.ts, and docs/frontend/src/api/dashboard.md
- [x] T063 [US3] Separate Docker development topology from native Linux production topology in docker-compose.dev.yml, infra/systemd/triton-server.service, docs/backend/architecture/deployment-topology.md, and docs/backend/architecture/triton-operations.md
- [x] T064 [US3] Record medium/low temporary coupling exceptions and prove no high-risk exceptions remain in docs/architecture/coupling-risk-register.md
- [x] T065 [US3] Record representative module-swap evidence and unrelated-workflow regression evidence in specs/006-modular-low-coupling/evidence/us3-contract-isolation.md

**Checkpoint**: User Story 3 is independently testable through contract-swap, failure-boundary, and deployment-boundary evidence.

---

## Phase 6: User Story 4 - Explain the Modular System Clearly (Priority: P2)

**Goal**: Make the modular design traceable from documentation alone using Mermaid diagrams, detailed explanations, cross-links, and coupling-risk records.

**Independent Test**: A reviewer can trace live stream processing, offline video processing, frontend-backend interactions, backend-inference interactions, and deployment boundaries from documentation without reading source code.

### Tests for User Story 4 (Mandatory)

- [x] T066 [P] [US4] Expand Mermaid syntax, explanation, and cross-link validation in backend/tests/unit/docs/test_docs_link_and_mermaid_validation.py and docs/backend/tests/unit/docs/test_docs_link_and_mermaid_validation.md
- [x] T067 [P] [US4] Add documentation diagram contract alignment tests for code structure, system interaction, cross interaction, state, and ER/class diagrams in backend/tests/integration/test_docs_contract_alignment.py and docs/backend/tests/integration/test_docs_contract_alignment.md
- [x] T068 [P] [US4] Add evidence-pack diagram coverage tests in backend/tests/contract/test_documentation_diagram_contract.py and docs/backend/tests/contract/test_documentation_diagram_contract.md

### Implementation for User Story 4

- [x] T069 [US4] Create the modular system overview with code structure, system interaction, and cross interaction Mermaid diagrams in docs/architecture/modular-system-overview.md
- [x] T070 [US4] Update the root architecture guide with modular boundary diagrams, explanations, and related document links in docs/ARCHITECTURE.md
- [x] T071 [US4] Update backend data-flow and runtime docs with live/offline cross-interaction diagrams and explanations in docs/backend/architecture/data-flow.md and docs/backend/architecture/observability-runbook.md
- [x] T072 [US4] Update backend deployment docs with Docker development, Triton development, and native Linux production diagrams in docs/backend/architecture/deployment-topology.md and docs/backend/architecture/triton-operations.md
- [x] T073 [US4] Update frontend API, store, and page documentation with code structure and frontend-backend interaction diagrams in docs/frontend/src/api/README.md, docs/frontend/src/App.md, and docs/frontend/src/main.md
- [x] T074 [US4] Update tracking and video analysis documentation with Ultralytics-aligned prediction/tracking diagrams and explanations in docs/backend/apps/tracking/README.md and docs/backend/apps/video_analysis/README.md
- [x] T075 [US4] Update feature documentation diagram coverage evidence in docs/architecture/documentation-diagram-coverage.md and specs/006-modular-low-coupling/evidence/us4-diagram-review.md
- [x] T076 [US4] Update documentation index cross-links so all new modularity, runtime, deployment, and evidence documents are discoverable in docs/INDEX.md

**Checkpoint**: User Story 4 is independently testable through documentation alignment tests and reviewer traceability.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Final verification, coverage closure, performance evidence, review gates, and release readiness.

- [x] T077 [P] Run backend unit, integration, contract, and system tests with coverage and record results in specs/006-modular-low-coupling/evidence/final/backend-test-results.md
- [x] T078 [P] Run frontend test, e2e, build, lint, type-check, and coverage commands and record results in specs/006-modular-low-coupling/evidence/final/frontend-test-results.md
- [x] T079 [P] Run live-stream real-data validation with real model weights and raw/live media and record results in specs/006-modular-low-coupling/evidence/final/live-stream-real-data.md
- [x] T080 [P] Run offline-video real-data validation with real model weights and raw video data and record results in specs/006-modular-low-coupling/evidence/final/offline-video-real-data.md
- [x] T081 [P] Generate 100% line and branch coverage reports or approved exception links in specs/006-modular-low-coupling/evidence/final/coverage-report.md
- [x] T082 [P] Update specs/006-modular-low-coupling/plan.md Complexity Tracking for every approved coverage exception with owner, expiry, removal plan, and rationale before completion
- [x] T083 [P] Verify high-risk coupling is resolved and medium/low exceptions are owner-expiry-plan-test complete in specs/006-modular-low-coupling/evidence/final/coupling-risk-signoff.md
- [x] T084 [P] Verify all Mermaid diagrams render, include explanations, and have valid cross-links in specs/006-modular-low-coupling/evidence/final/diagram-signoff.md
- [x] T085 [P] Verify no production Linux requirement depends on Docker and record native-service evidence in specs/006-modular-low-coupling/evidence/final/native-linux-signoff.md
- [x] T086 [P] Verify live-stream frame/result propagation, offline processing throughput, dashboard navigation, and health/export workflows stay within the plan's 10% performance regression limit in specs/006-modular-low-coupling/evidence/final/performance-thresholds.md
- [x] T087 [P] Validate specs/006-modular-low-coupling/checklists/modularity.md against final boundary, coupling, baseline, diagram, and evidence results in specs/006-modular-low-coupling/evidence/final/modularity-checklist-signoff.md
- [x] T088 [P] Run the quickstart validation workflow and record command output in specs/006-modular-low-coupling/evidence/final/quickstart-validation.md
- [x] T089 [P] Verify every touched source, test, and script file has a module-level docstring or header comment and record results in specs/006-modular-low-coupling/evidence/final/source-header-signoff.md
- [x] T090 [P] Run pre-commit evidence for backend lint, backend format, backend type-check where configured, frontend lint, frontend format, frontend type-check, script formatting, and docs formatting in specs/006-modular-low-coupling/evidence/final/pre-commit-checks.md
- [x] T091 Create and open the pull request for branch 006-modular-low-coupling, record the PR URL, target branch, task IDs, FR/SC traceability, evidence links, and constitution gate checklist in specs/006-modular-low-coupling/evidence/final/pull-request.md
- [x] T092 Record Windows and Linux CI evidence for the pull request and every required push check in specs/006-modular-low-coupling/evidence/final/cross-platform-ci.md
- [x] T093 Record automated review approval evidence or blocking findings from CodeRabbit or equivalent in specs/006-modular-low-coupling/evidence/final/automated-review.md
- [x] T094 Record required human review approval for architecture, security, constitution, and deployment-sensitive changes in specs/006-modular-low-coupling/evidence/final/human-review.md
- [x] T095 Record final reviewer sign-off and implementation completion summary in specs/006-modular-low-coupling/evidence/final/reviewer-signoff.md

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 Setup**: No dependencies; start immediately.
- **Phase 2 Foundational**: Depends on Phase 1; blocks all user story implementation.
- **Phase 3 User Story 1**: Depends on Phase 2; MVP and primary behavior-preservation gate.
- **Phase 4 User Story 2**: Depends on Phase 2; can run alongside US1 after foundation if ownership is coordinated.
- **Phase 5 User Story 3**: Depends on Phase 2 and benefits from US2 boundary declarations, but contract tests can be prepared in parallel.
- **Phase 6 User Story 4**: Depends on Phase 2 and should track docs for any completed story.
- **Phase 7 Polish**: Depends on all selected user stories being complete.

### User Story Dependencies

- **US1 (P1)**: Starts after Phase 2; no dependency on other stories.
- **US2 (P1)**: Starts after Phase 2; no dependency on other stories.
- **US3 (P2)**: Starts after Phase 2; implementation should align with US2 boundary declarations where available.
- **US4 (P2)**: Starts after Phase 2; documentation can proceed incrementally as US1-US3 change code and contracts.

### Within Each User Story

- Tests must be written first and verified RED before implementation tasks.
- Existing-suite baseline evidence must be recorded separately from RED evidence for new tests.
- Boundary schemas and declarations must precede dependency enforcement.
- Contract interfaces must precede consumers that depend on them.
- Every touched source, test, and script file must have a module-level docstring or header comment before its task is complete.
- Documentation diagrams must be updated with the code or contract change they explain.
- Evidence files must be updated before a story checkpoint is considered complete.
- Pre-commit, pull request URL, Windows/Linux CI, automated review, and human review evidence must be recorded before final completion.

## Parallel Opportunities

- Setup evidence tasks T002-T006 can run in parallel after T001.
- Foundational contract and documentation tests T013-T017 can run in parallel after T008-T012 are drafted.
- US1 tests T025-T029 can run in parallel before implementation.
- US2 boundary declaration tasks T043-T047 can run in parallel because each owns different files.
- US3 test tasks T052-T056 can run in parallel before contract implementation.
- US4 test tasks T066-T068 can run in parallel before documentation updates.
- Final evidence tasks T077-T090 can run in parallel after all story checkpoints pass; T091-T095 depend on the final evidence tasks.

## Parallel Example: User Story 1

```text
Task: T025 Add full delivered backend baseline regression tests in backend/tests/integration/test_full_delivered_baseline.py
Task: T026 Add live-stream real-data equivalence system tests in backend/tests/system/test_live_stream_modular_equivalence.py
Task: T027 Add offline-video real-data equivalence system tests in backend/tests/system/test_offline_video_modular_equivalence.py
Task: T028 Add frontend full baseline e2e tests in frontend/tests/e2e/full-baseline.spec.ts
```

## Parallel Example: User Story 2

```text
Task: T043 Add accounts, audit, and exams boundary declarations
Task: T044 Add cameras, sessions, and detections boundary declarations
Task: T045 Add pipeline, tracking, and video_analysis boundary declarations
Task: T046 Add anomalies, recordings, exports, and health boundary declarations
Task: T047 Add frontend boundary declarations
```

## Parallel Example: User Story 3

```text
Task: T052 Add inference provider swap contract tests
Task: T053 Add tracking rule isolation tests
Task: T054 Add frontend API contract tests
Task: T055 Add failure and degraded-state contract tests
Task: T056 Add native Linux deployment contract tests
```

## Parallel Example: User Story 4

```text
Task: T066 Expand Mermaid validation
Task: T067 Add documentation diagram contract alignment tests
Task: T068 Add evidence-pack diagram coverage tests
Task: T069 Create modular system overview documentation
```

## Implementation Strategy

### MVP First

1. Complete Phase 1 setup evidence.
2. Complete Phase 2 foundational governance, tests, and CI gates.
3. Complete Phase 3 User Story 1.
4. Stop and validate live stream, offline video, and non-video dashboard baseline preservation before deeper refactoring.

### Incremental Delivery

1. Deliver US1 to prove behavior preservation.
2. Deliver US2 to make ownership and boundaries explicit.
3. Deliver US3 to prove low-coupling contract isolation.
4. Deliver US4 to make the final modular design reviewable from docs.
5. Complete Phase 7 evidence, pre-commit checks, PR opening, Windows/Linux CI, automated review, human review, and reviewer sign-off.

### Team Strategy

After Phase 2, one developer can own US1 baseline preservation, one can own US2 boundary declarations, one can own US3 contract isolation, and one can own US4 documentation diagrams. Coordinate shared files named in each task before editing them.

## Notes

- Commit after each task or logical group.
- Do not keep high-risk coupling as an exception.
- Medium/low temporary coupling requires owner, expiry, removal plan, regression coverage, and reviewer approval.
- Coverage exceptions require both evidence links and `plan.md` Complexity Tracking entries.
- Keep Docker-specific requirements in development/test docs only; production Linux documentation must use native service assumptions.
- Use official Ultralytics Predict and Track docs as the primary reference for inference and tracking behavior.
- Every touched source, test, and script file must include a module-level docstring or header comment.
- Every source file change must include the corresponding documentation update and related Mermaid diagrams in the same task and commit.
- Every completed implementation task must be traceable in an opened pull request with URL, target branch, Windows/Linux CI evidence, automated review evidence, and required human approval.
