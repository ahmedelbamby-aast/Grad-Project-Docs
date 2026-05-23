# Tasks: Parallel Pose Inference

**Input**: Design documents from `/specs/009-parallel-pose-inference/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: Tests are MANDATORY per the Test-in-Loop methodology (Constitution §II). ALL tests (unit, integration, system) MUST be written BEFORE implementation for every user story. Tests MUST be verified to fail (RED) before any implementation code is written. Affected modules and features MUST reach 100% line and branch coverage. Inference, prediction, tracking, video-processing, and overlay tests MUST use real model weights and real raw test data; mocks MUST NOT replace model weights, raw media, trackers, prediction outputs, or system wiring.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- Backend: `backend/apps/`, `backend/tests/`
- Frontend: `frontend/src/`, `frontend/tests/`
- Docs mirror: `docs/backend/`, `docs/frontend/`

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Prepare runtime profiles, tooling, and test scaffolding for hybrid telemetry and stress-lane validation.

### Baseline Environment Matrix

| Lane | Runtime Target | Device Target | Telemetry Mode | Profile Mode |
|------|----------------|---------------|----------------|--------------|
| NVIDIA Stress | triton | nvidia:gpu | mandatory-telemetry-v1 | baseline/optimized |
| Intel Stress | openvino | intel:gpu | mandatory-telemetry-v1 | baseline/optimized |
| Hybrid Assignment | triton+openvino | nvidia:gpu + intel:gpu | mandatory-telemetry-v1 | baseline/optimized |

- [X] T001 Create feature task tracker and baseline environment matrix in `specs/009-parallel-pose-inference/tasks.md`
- [X] T002 [P] Add validation profile env template for full-model Triton/OpenVINO lanes in `backend/.env.validation.example`
- [X] T003 [P] Add benchmark mode configuration schema for stress lanes and telemetry mode IDs in `backend/apps/pipeline/config.py`
- [X] T004 [P] Add test data manifest for required raw media and model artifacts in `backend/tests/fixtures/validation_manifest.json`
- [X] T005 [P] Add helper script for baseline-vs-optimized lane invocation in `backend/scripts/run_hybrid_validation.ps1`
- [X] T006 [P] Add helper script for PresentMon capture invocation in `backend/scripts/run_presentmon_capture.ps1`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core data model, buffering, runtime assignment, and evidence persistence needed by all stories.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [X] T007 Create migration for run/evidence/telemetry tables including `BufferPressureEvent` in `backend/apps/pipeline/migrations/`
- [X] T008 Implement data models for HybridValidationRun, assignments, capture sessions, normalized records, and buffer pressure events in `backend/apps/pipeline/models.py`
- [X] T009 [P] Implement strict serializer schemas for telemetry and validation entities in `backend/apps/pipeline/serializers.py`
- [X] T010 Implement bounded tiered buffering configuration (`buffer.ram.*`, `buffer.spill.*`, `buffer.overflow_policy`) in `backend/apps/pipeline/config.py`
- [X] T011 Implement RAM queue + disk spill queue primitives with ordering guarantees in `backend/apps/pipeline/buffering.py`
- [X] T012 Implement deterministic overflow policy handlers (`block`, `drop_oldest`, `drop_newest`) in `backend/apps/pipeline/buffering.py`
- [X] T013 [P] Implement queue/spill/overflow telemetry event writer in `backend/apps/pipeline/runtime_ingestion.py`
- [X] T014 Implement model-identity routing and dedicated worker contract enforcement in `backend/apps/pipeline/runtime_router.py`
- [X] T015 [P] Implement shared run-correlation manifest writer in `backend/apps/pipeline/evidence.py`
- [X] T016 Add foundational integration wiring tests for model routing, queue bounds, spill behavior, and persistence in `backend/tests/integration/test_pipeline_foundation.py`

**Checkpoint**: Foundation ready - user story implementation can now begin.

---

## Priority Override

MCP server and governed telemetry endpoint work (`US6`, tasks `T062`-`T072`) is the highest implementation priority after Phase 2 and must be executed before other user stories.

---

## Phase 3: User Story 1 - Live Dual Inference (Priority: P1) 🎯 MVP

**Goal**: Run object + pose pipelines concurrently in live mode with delayed-live timeline, degraded-state visibility, and tiered overload handling.

**Independent Test**: Start a representative live session and confirm synchronized object/pose artifacts per delayed-live timeline, degraded-state visibility, and consistent state through temporary pipeline failures.

### Tests for User Story 1 (MANDATORY — Test-in-Loop) 🔴

- [X] T017 [P] [US1] Add unit tests for delayed-live scheduler and frame identity propagation in `backend/tests/unit/test_live_scheduler.py`
- [X] T018 [P] [US1] Add unit tests for fallback region handling and degraded pose markers in `backend/tests/unit/test_pose_fallback.py`
- [X] T019 [P] [US1] Add integration tests for concurrent live object+pose workers in `backend/tests/integration/test_live_parallel_pipeline.py`
- [X] T020 [P] [US1] Add system test for live dual inference on representative stream data in `backend/tests/system/test_live_dual_inference.py`
- [X] T021 [P] [US1] Add frontend store/component tests for live state transitions (pending/buffering/running/delayed/degraded) in `frontend/tests/unit/components/camera/CameraFeedLiveState.test.tsx`

### Implementation for User Story 1

- [X] T022 [US1] Implement live frame timeline coordinator and shared frame IDs in `backend/apps/pipeline/live_coordinator.py`
- [X] T023 [P] [US1] Implement parallel live dispatch for object and pose workers in `backend/apps/pipeline/tasks.py`
- [X] T024 [US1] Integrate delayed-live buffer with tiered overload buffering in `backend/apps/pipeline/live_buffer.py`
- [X] T025 [US1] Implement degraded-state propagation and reason codes in `backend/apps/pipeline/state_machine.py`
- [X] T026 [P] [US1] Expose live job/session state fields via API serializers in `backend/apps/video_analysis/serializers.py`
- [X] T027 [P] [US1] Update live monitoring UI state rendering and warnings in `frontend/src/pages/CameraFeedPage.tsx`
- [X] T028 [US1] Add live pipeline operational logs and reconciliation counters in `backend/apps/pipeline/services.py`

**Checkpoint**: User Story 1 is fully functional and independently testable.

---

## Phase 4: User Story 2 - Offline Dual Artifact Generation (Priority: P1)

**Goal**: Produce complete, reconciled offline artifacts and pass heavy-workload full-model lanes for Triton and OpenVINO.

**Independent Test**: Run representative offline video through full-model NVIDIA lane and full-model Intel lane, each with baseline-vs-optimized comparison and complete artifact reconciliation.

### Tests for User Story 2 (MANDATORY — Test-in-Loop) 🔴

- [X] T029 [P] [US2] Add unit tests for offline frame reconciliation and pose-gap thresholds in `backend/tests/unit/test_offline_reconciliation.py`
- [X] T030 [P] [US2] Add integration tests for full-model lane assignment from `.env` in `backend/tests/integration/test_full_model_lane_assignment.py`
- [X] T031 [P] [US2] Add integration tests for artifact manifest integrity and atomic status labeling in `backend/tests/integration/test_artifact_manifest_integrity.py`
- [X] T032 [P] [US2] Add system test for full-model NVIDIA stress lane in `backend/tests/system/test_offline_stress_nvidia_triton.py`
- [X] T033 [P] [US2] Add system test for full-model Intel stress lane in `backend/tests/system/test_offline_stress_intel_openvino.py`

### Implementation for User Story 2

- [X] T034 [US2] Implement `.env` selected-model expansion and strict lane binding in `backend/apps/pipeline/runtime_router.py`
- [X] T035 [US2] Implement offline stage execution ledger and stage verdict persistence in `backend/apps/pipeline/services.py`
- [X] T036 [P] [US2] Implement frame-level object/pose/association reconciliation with threshold gates in `backend/apps/pipeline/reconciliation.py`
- [X] T037 [P] [US2] Implement artifact manifest hashing/readability validation in `backend/apps/pipeline/evidence.py`
- [X] T038 [US2] Enforce offline failure conditions for pose missing ratio and continuous gap in `backend/apps/pipeline/validation.py`
- [X] T039 [P] [US2] Persist baseline-vs-optimized lane comparison linkage in `backend/apps/pipeline/models.py`

**Checkpoint**: User Story 2 produces complete offline artifacts and full-model stress-lane evidence.

---

## Phase 5: User Story 3 - Accelerator Throughput Optimization (Priority: P2)

**Goal**: Benchmark and select safe high-throughput live/offline profiles with correctness gates.

**Independent Test**: Execute benchmark suite on supported profiles and verify accepted profile improves throughput/latency while preserving correctness and completeness.

### Tests for User Story 3 (MANDATORY — Test-in-Loop) 🔴

- [X] T040 [P] [US3] Add unit tests for profile selection and rejection logic in `backend/tests/unit/test_profile_selection.py`
- [X] T041 [P] [US3] Add integration tests for per-model worker backlog and failure isolation reporting in `backend/tests/integration/test_worker_isolation_metrics.py`
- [X] T042 [P] [US3] Add performance test harness for baseline vs optimized parity checks in `backend/tests/performance/test_hybrid_profile_benchmarks.py`
- [X] T043 [P] [US3] Add resilience test for profile fallback on correctness gate failure in `backend/tests/resilience/test_profile_fallback.py`

### Implementation for User Story 3

- [X] T044 [US3] Implement optimization profile registry with separate live/offline defaults in `backend/apps/pipeline/profiles.py`
- [X] T045 [P] [US3] Implement benchmark metrics collector for decode/preprocess/inference/postprocess/artifact-write/queue-wait in `backend/apps/pipeline/benchmark.py`
- [X] T046 [US3] Implement profile acceptance gate evaluator with quality and completeness thresholds in `backend/apps/pipeline/validation.py`
- [X] T047 [P] [US3] Implement per-model worker health/backlog/timeout/error rollups in `backend/apps/pipeline/runtime_ingestion.py`

**Checkpoint**: User Story 3 can choose and justify optimized profiles with measured evidence.

---

## Phase 6: User Story 4 - Grayscale Optimization Decision (Priority: P2)

**Goal**: Evaluate grayscale as a candidate optimization with strict throughput + quality parity gates.

**Independent Test**: Compare RGB vs grayscale runs on same input and accept grayscale only if all quality/parity gates pass and throughput target is met.

### Tests for User Story 4 (MANDATORY — Test-in-Loop) 🔴

- [X] T048 [P] [US4] Add unit tests for grayscale acceptance threshold logic in `backend/tests/unit/test_grayscale_gates.py`
- [X] T049 [P] [US4] Add integration tests for color mode provenance in run artifacts and summaries in `backend/tests/integration/test_color_mode_provenance.py`
- [X] T050 [P] [US4] Add performance tests for RGB-vs-grayscale stage cost breakdown in `backend/tests/performance/test_grayscale_stage_costs.py`

### Implementation for User Story 4

- [X] T051 [US4] Implement color-mode execution path flags and artifact tagging in `backend/apps/pipeline/services.py`
- [X] T052 [P] [US4] Implement grayscale benchmark report decomposition by stage in `backend/apps/pipeline/benchmark.py`
- [X] T053 [US4] Implement grayscale acceptance/rejection persistence and explanation fields in `backend/apps/pipeline/models.py`

**Checkpoint**: User Story 4 provides evidence-based grayscale decisioning with traceable outcomes.

---

## Phase 7: User Story 5 - Mature Failure Handling and Evidence (Priority: P3)

**Goal**: Harden failure behavior across decode/inference/storage/overload scenarios with clear summaries and safe degradation.

**Independent Test**: Run failure-injection matrix and confirm bounded behavior, accurate artifacts, and actionable final summaries.

### Tests for User Story 5 (MANDATORY — Test-in-Loop) 🔴

- [X] T054 [P] [US5] Add unit tests for overflow policy behavior (`block`, `drop_oldest`, `drop_newest`) in `backend/tests/unit/test_overflow_policies.py`
- [X] T055 [P] [US5] Add integration tests for disk spill failure modes (disk-full, permission, timeout, corruption) in `backend/tests/integration/test_spill_failure_modes.py`
- [X] T056 [P] [US5] Add resilience tests for retry isolation and duplicate-artifact prevention in `backend/tests/resilience/test_retry_isolation.py`
- [X] T057 [P] [US5] Add system test for artifact storage failure and reconciliation summary accuracy in `backend/tests/system/test_artifact_failure_summary.py`

### Implementation for User Story 5

- [X] T058 [US5] Implement spill queue recovery/drain orchestration with ordering guarantees in `backend/apps/pipeline/buffering.py`
- [X] T059 [P] [US5] Implement failure-injection hooks for pipeline stages in `backend/apps/pipeline/failure_injection.py`
- [X] T060 [US5] Implement final run summary reconciliation fields for buffered/spilled/dropped/delayed frames in `backend/apps/pipeline/reconciliation.py`
- [X] T061 [P] [US5] Implement operator-facing actionable error mapping in `backend/apps/video_analysis/views.py`

**Checkpoint**: User Story 5 provides robust failure handling and auditable evidence consistency.

---

## Phase 8: User Story 6 - Governed Telemetry Access (Priority: P3)

**Goal**: Deliver mandatory telemetry endpoints and mandatory read-only MCP service over normalized evidence with RBAC, bounds, redaction, and audit.

**Independent Test**: Validate dashboard REST/WebSocket telemetry and MCP resources/tools from same normalized store, including authorized reads, denied prohibited actions, and failure isolation.

### Tests for User Story 6 (MANDATORY — Test-in-Loop) 🔴

- [X] T062 [P] [US6] Add contract tests for dashboard telemetry REST endpoints in `backend/tests/contract/test_dashboard_telemetry_api.py`
- [X] T063 [P] [US6] Add contract tests for telemetry WebSocket stream schema and auth in `backend/tests/contract/test_dashboard_telemetry_ws.py`
- [X] T064 [P] [US6] Add contract tests for MCP resources/tools, bounds, and denied operations in `backend/tests/contract/test_mcp_telemetry_contract.py`
- [X] T065 [P] [US6] Add integration tests for MCP audit events and redaction behavior in `backend/tests/integration/test_mcp_audit_redaction.py`
- [X] T066 [P] [US6] Add resilience tests for MCP outage isolation from inference/dashboard delivery in `backend/tests/resilience/test_mcp_isolation.py`

### Implementation for User Story 6

- [X] T067 [US6] Implement normalized telemetry projection query service in `backend/apps/pipeline/runtime_ingestion.py`
- [X] T068 [P] [US6] Implement dashboard telemetry REST endpoints in `backend/apps/video_analysis/views.py`
- [X] T069 [P] [US6] Implement dashboard telemetry WebSocket consumer in `backend/apps/video_analysis/consumers.py`
- [X] T070 [US6] Implement mandatory read-only MCP telemetry server resources/tools in `backend/apps/telemetry_mcp/server.py`
- [X] T071 [P] [US6] Implement MCP RBAC, bounds enforcement, and redaction in `backend/apps/telemetry_mcp/security.py`
- [X] T072 [P] [US6] Implement MCP access audit persistence in `backend/apps/telemetry_mcp/audit.py`

**Checkpoint**: User Story 6 provides governed telemetry access with mandatory dashboard + MCP surfaces.

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: Final validation, coverage closure, docs parity, and readiness proof.

- [ ] T073 [P] Run full backend suite for affected scopes with xdist and collect coverage reports in `backend/tests/`
- [ ] T074 [P] Run frontend telemetry UI and integration tests in `frontend/tests/`
- [ ] T075 Run end-to-end quickstart validation workflow and capture evidence package in `specs/009-parallel-pose-inference/quickstart.md`
- [ ] T076 [P] Update backend source docs mirrors for touched files in `docs/backend/`
- [ ] T077 [P] Update frontend source docs mirrors for touched files in `docs/frontend/`
- [ ] T078 [P] Update affected module `README.md` files for new runtime/buffering/telemetry contracts
- [ ] T079 Validate 100% line and branch coverage gates for changed modules and patch remaining gaps in `backend/tests/` and `frontend/tests/`
- [ ] T080 Produce final completion evidence index linking hybrid run + stress lanes + telemetry + fidelity artifacts in `backend/data/videos/{job_id}/run_manifest.json`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: No dependencies.
- **Phase 2 (Foundational)**: Depends on Phase 1 completion; blocks all user stories.
- **Phases 3-8 (User Stories)**: Depend on Phase 2 completion.
- **Phase 9 (Polish)**: Depends on completion of desired user stories.

### User Story Dependencies

- **US6 (Priority Override)**: Starts first after foundational work and establishes mandatory governed telemetry surfaces.
- **US1 (P1)**: Starts after foundational work and after US6 contract/API baseline is in place.
- **US2 (P1)**: Starts after foundational work and after US6 telemetry projection baseline is in place.
- **US3 (P2)**: Depends on US1/US2 runtime evidence paths being present.
- **US4 (P2)**: Depends on US3 benchmark and validation scaffolding.
- **US5 (P3)**: Depends on foundational buffering and reconciliation primitives.
- **US6 (P3 label in spec, execution priority override in plan)**: Core MCP/dashboard contracts execute first; remaining hardening can continue in parallel with later stories.

### Within Each User Story

- Write tests first and verify RED before implementation.
- Data/persistence updates before service logic.
- Service logic before API/UI or MCP surface exposure.
- Story must pass independent test before marking complete.

### Parallel Opportunities

- Setup tasks marked [P] can run concurrently.
- Foundational [P] tasks can run concurrently after schema direction is set.
- In each story, test tasks marked [P] can be authored in parallel.
- API/consumer and MCP security/audit tasks in US6 can run in parallel.

---

## Parallel Example: User Story 6

```bash
Task: "Add contract tests for dashboard telemetry REST endpoints in backend/tests/contract/test_dashboard_telemetry_api.py"
Task: "Add contract tests for telemetry WebSocket stream schema and auth in backend/tests/contract/test_dashboard_telemetry_ws.py"
Task: "Add contract tests for MCP resources/tools, bounds, and denied operations in backend/tests/contract/test_mcp_telemetry_contract.py"
```

---

## Implementation Strategy

### MVP First (Priority Override: User Story 6)

1. Complete Setup + Foundational phases.
2. Deliver User Story 6 mandatory governed telemetry access surfaces (dashboard REST/WebSocket + read-only MCP + RBAC/audit).
3. Validate independently with contract/security/isolation tests.

### Incremental Delivery

1. Complete US6 first (highest priority override).
2. Add US1 and US2 for live/offline dual-pipeline behavior and full-model stress-lane evidence.
3. Add US3 and US4 for optimization profile selection and grayscale gating.
4. Add US5 for hardened failure behavior.

### Parallel Team Strategy

1. Team A: Foundational buffering + runtime routing.
2. Team B: Offline evidence + benchmarks.
3. Team C: Dashboard + MCP contracts and security/audit.

---

## Notes

- [P] tasks are safe parallel candidates with disjoint primary files.
- Keep commits atomic per logical task group and preserve evidence traceability.
- Ensure every touched source file has corresponding `docs/` markdown updates and module README updates.
- Do not mark feature complete before hybrid assignment run, two full-model stress lanes, and required telemetry/fidelity gates pass.
