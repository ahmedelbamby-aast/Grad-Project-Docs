# Tasks: Pose Behavior Pipeline Foundation

**Input**: Design documents from `/specs/007-pose-behavior-pipeline/`  
**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [quickstart.md](quickstart.md), [contracts/](contracts/)

**Tests**: Mandatory (Test-in-Loop). For each phase and user story, write tests first, verify RED failure, then implement. Inference/tracking/video tests must use real model artifacts and real raw media.

**Organization**: Tasks are grouped by user story to keep each story independently implementable and testable after foundational tasks complete.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Parallelizable (different files, no blocking dependency on incomplete tasks)
- **[Story]**: Story mapping label (`[US1]`, `[US2]`, `[US3]`, `[US4]`)
- Every task includes exact file path(s)

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Establish feature scaffolding, configuration surfaces, and baseline evidence locations.

- [X] T001 Create feature evidence workspace in `specs/007-pose-behavior-pipeline/evidence/README.md`
- [X] T002 [P] Add pose-behavior environment variable placeholders to `backend\.env.example` and `frontend\.env.example`
- [X] T003 [P] Add feature configuration defaults and validation stubs in `backend\apps\pipeline\config.py` and `docs\backend\apps\pipeline\config.md`
- [X] T004 [P] Add frontend runtime config constants for pose-behavior keys in `frontend\src\constants\index.ts` and `docs\frontend\src\constants\index.md`
- [X] T005 [P] Create model artifact manifest scaffold for RTMPose/OpenVINO/TensorRT lineage in `backend\apps\pipeline\model_lifecycle\rtmpose_manifest.py` and `docs\backend\apps\pipeline\model_lifecycle\rtmpose_manifest.md`
- [X] T006 [P] Capture pre-change backend baseline command outputs in `specs/007-pose-behavior-pipeline/evidence/baseline/backend-before.md`
- [X] T007 [P] Capture pre-change frontend baseline command outputs in `specs/007-pose-behavior-pipeline/evidence/baseline/frontend-before.md`
- [X] T008 Record available real raw media and model artifact paths for live/offline validation in `specs/007-pose-behavior-pipeline/evidence/real-data-assets.md`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Build shared contracts, persistence, and service boundaries required by all user stories.

**Critical**: No user story work starts until this phase is complete.

### Tests for Foundational Phase (MANDATORY) 🔴

- [X] T009 [P] Add contract tests for pose-behavior schema records in `backend\tests\contract\test_pose_behavior_schema_contract.py`
- [X] T010 [P] Add contract tests for model-serving route policy schema in `backend\tests\contract\test_model_route_policy_contract.py`
- [X] T011 [P] Add integration tests for tracking ID handoff into pose pipeline in `backend\tests\integration\test_tracking_pose_handoff.py`
- [X] T012 [P] Add frontend contract tests for behavior timeline payload typing in `frontend\tests\unit\api\behaviorTimeline.contract.test.ts`
- [X] T013 [P] Add system smoke tests for live/offline feature wiring skeleton in `backend\tests\system\test_pose_behavior_foundation_smoke.py`
- [X] T014 Record RED-phase results for T009-T013 in `specs/007-pose-behavior-pipeline/evidence/foundation-red.md`

### Implementation for Foundational Phase

- [X] T015 Create shared data entities for session analysis and track lifecycle in `backend\apps\pipeline\models.py` and `docs\backend\apps\pipeline\models.md`
- [X] T016 [P] Create behavior signal and fused frame entities in `backend\apps\detections\models.py` and `docs\backend\apps\detections\models.md`
- [X] T017 [P] Create cheating event and triage entities in `backend\apps\anomalies\models.py` and `docs\backend\apps\anomalies\models.md`
- [X] T018 [P] Create serving route policy and artifact version entities in `backend\apps\pipeline\model_lifecycle\models.py` and `docs\backend\apps\pipeline\model_lifecycle\models.md`
- [X] T019 Add migrations for T015-T018 in `backend\apps\pipeline\migrations\` and `backend\apps\anomalies\migrations\`
- [X] T020 Implement shared serializers/schemas for behavior and anomaly contracts in `backend\apps\pipeline\schemas\pose_behavior.py` and `backend\apps\anomalies\serializers.py`
- [X] T021 [P] Add API route stubs for behavior timeline and event triage in `backend\apps\anomalies\urls.py`, `backend\apps\anomalies\views.py`, and `docs\backend\apps\anomalies\views.md`
- [X] T022 [P] Add WebSocket event channel stubs for live behavior stream in `backend\apps\anomalies\consumers.py` and `backend\apps\anomalies\routing.py`
- [X] T023 Add frontend base types and API clients for behavior/event payloads in `frontend\src\types\behavior.ts`, `frontend\src\api\anomalies.ts`, and `docs\frontend\src\api\anomalies.md`
- [X] T024 Record foundational completion evidence and baseline schema snapshots in `specs/007-pose-behavior-pipeline/evidence/foundation-ready.md`

**Checkpoint**: Foundation complete. User stories can now proceed.

---

## Phase 3: User Story 1 - Detect Cheating Timelines in Live Classrooms (Priority: P1) 🎯 MVP

**Goal**: Deliver live per-student behavior extraction, fused signal timeline, and cheating event/group outputs with low-latency updates.

**Independent Test**: Run live classroom stream with real model artifacts and verify per-frame/per-student signals, fused timeline updates, cheating events with start/end, and group association.

### Tests for User Story 1 (MANDATORY) 🔴

- [X] T025 [P] [US1] Add unit tests for temporal stabilization pipeline in `backend\tests\unit\pipeline\test_temporal_stabilizer.py`
- [X] T026 [P] [US1] Add unit tests for head/hand/torso/motion feature extraction in `backend\tests\unit\pipeline\test_behavior_feature_extractors.py`
- [X] T027 [P] [US1] Add contract tests for live behavior signal payload in `backend\tests\contract\test_live_behavior_signal_contract.py`
- [X] T028 [P] [US1] Add system test for live stream inference-to-timeline flow using real artifacts in `backend\tests\system\test_live_pose_behavior_pipeline.py`
- [X] T029 [P] [US1] Add system test for live cheating pair/group event generation in `backend\tests\system\test_live_cheating_group_events.py`
- [X] T030 [P] [US1] Add frontend integration test for live timeline store updates in `frontend\tests\integration\liveBehaviorTimeline.test.tsx`
- [X] T031 [P] [US1] Add frontend e2e live monitoring timeline/event scenario in `frontend\tests\e2e\live-behavior-timeline.spec.ts`

### Implementation for User Story 1

- [X] T032 [US1] Implement RTMPose live inference adapter boundary in `backend\apps\pipeline\services\pose_runtime.py` and `docs\backend\apps\pipeline\services\pose_runtime.md`
- [X] T033 [P] [US1] Implement OneEuro + interpolation + confidence-aware stabilization in `backend\apps\pipeline\services\temporal_stabilizer.py` and `docs\backend\apps\pipeline\services\temporal_stabilizer.md`
- [X] T034 [P] [US1] Implement missing-keypoint recovery and outlier rejection in `backend\apps\pipeline\services\pose_recovery.py` and `docs\backend\apps\pipeline\services\pose_recovery.md`
- [X] T035 [P] [US1] Implement head/hand/torso/motion feature extractors in `backend\apps\pipeline\services\behavior_features.py` and `docs\backend\apps\pipeline\services\behavior_features.md`
- [X] T036 [US1] Implement interaction feature extraction for synchronized and pairwise behavior in `backend\apps\pipeline\services\interaction_features.py` and `docs\backend\apps\pipeline\services\interaction_features.md`
- [X] T037 [US1] Implement fused live signal frame builder combining pose and contrastive signals in `backend\apps\pipeline\services\signal_fusion.py` and `docs\backend\apps\pipeline\services\signal_fusion.md`
- [X] T038 [US1] Implement deterministic live anomaly rule evaluation and event lifecycle hooks in `backend\apps\anomalies\services\behavior_events.py` and `docs\backend\apps\anomalies\services\behavior_events.md`
- [X] T039 [US1] Integrate live pipeline outputs with existing tracking/ReID identity streams in `backend\apps\tracking\services\tracking_service.py` and `docs\backend\apps\tracking\services\tracking_service.md`
- [X] T040 [US1] Expose live timeline/event stream endpoints and socket payloads in `backend\apps\anomalies\views.py`, `backend\apps\anomalies\consumers.py`, and `docs\backend\apps\anomalies\consumers.md`
- [X] T041 [US1] Render live behavior timeline and cheating events in `frontend\src\pages\CameraFeedPage.tsx`, `frontend\src\stores\anomalyStore.ts`, and `docs\frontend\src\pages\CameraFeedPage.md`

**Checkpoint**: US1 is independently functional and is the MVP scope.

---

## Phase 4: User Story 2 - Analyze Uploaded/Raw Videos with Full Behavioral Outputs (Priority: P1)

**Goal**: Deliver offline batch processing that produces the same behavior/event contract as live mode.

**Independent Test**: Process real raw video and verify persisted per-frame/per-student behavior signals, cheating intervals, and group events are queryable and visible in review UI.

### Tests for User Story 2 (MANDATORY) 🔴

- [X] T042 [P] [US2] Add unit tests for offline temporal buffer rolling logic in `backend\tests\unit\video_analysis\test_temporal_buffer_window.py`
- [X] T043 [P] [US2] Add contract tests for offline behavior artifact payload in `backend\tests\contract\test_offline_behavior_artifact_contract.py`
- [X] T044 [P] [US2] Add integration tests for offline job orchestration with pose-behavior stages in `backend\tests\integration\test_offline_pose_behavior_orchestration.py`
- [X] T045 [P] [US2] Add system test for offline raw video end-to-end outputs in `backend\tests\system\test_offline_pose_behavior_pipeline.py`
- [X] T046 [P] [US2] Add frontend integration test for recording timeline hydration in `frontend\tests\integration\recordingBehaviorTimeline.test.tsx`
- [X] T047 [P] [US2] Add frontend e2e offline analysis review scenario in `frontend\tests\e2e\offline-behavior-review.spec.ts`

### Implementation for User Story 2

- [X] T048 [US2] Extend offline inference orchestrator with RTMPose behavior stages in `backend\apps\video_analysis\services\inference_orchestrator.py` and `docs\backend\apps\video_analysis\services.md`
- [X] T049 [P] [US2] Add offline job task wiring for pose-behavior steps in `backend\apps\video_analysis\tasks.py` and `docs\backend\apps\video_analysis\tasks.md`
- [X] T050 [P] [US2] Implement offline persistence for signals, fused frames, and events in `backend\apps\video_analysis\models.py` and `docs\backend\apps\video_analysis\models.md`
- [X] T051 [US2] Add offline result retrieval endpoints for behavior/event timelines in `backend\apps\video_analysis\views.py` and `backend\apps\video_analysis\urls.py`
- [X] T052 [US2] Add offline-live contract parity checks in `backend\apps\pipeline\services\contract_parity.py` and `docs\backend\apps\pipeline\services\contract_parity.md`
- [X] T053 [US2] Add recording playback timeline payload integration in `backend\apps\recordings\views.py` and `docs\backend\apps\recordings\views.md`
- [X] T054 [US2] Render offline behavior timeline and event groups in `frontend\src\pages\RecordingDetailPage.tsx` and `docs\frontend\src\pages\RecordingDetailPage.md`
- [X] T055 [P] [US2] Add offline behavior API client methods in `frontend\src\api\videoAnalysis.ts` and `docs\frontend\src\api\videoAnalysis.md`
- [X] T056 [US2] Persist offline investigation evidence snapshots in `specs/007-pose-behavior-pipeline/evidence/us2-offline-results.md`

**Checkpoint**: US2 independently validates offline full-contract behavior analysis.

---

## Phase 5: User Story 3 - Configure and Operate Pipeline Without Rebuilding Existing Features (Priority: P2)

**Goal**: Expose runtime controls via `.env`, enforce reuse of existing modules, and implement deterministic hybrid serving fallback.

**Independent Test**: Change configuration only, run live/offline flows, simulate primary serving failure, and verify output contract stability with reused tracking/ReID paths.

### Tests for User Story 3 (MANDATORY) 🔴

- [X] T057 [P] [US3] Add contract tests for `.env` pose-behavior config parsing in `backend\tests\contract\test_pose_behavior_env_contract.py`
- [X] T058 [P] [US3] Add integration tests for primary/secondary serving fallback policy in `backend\tests\integration\test_pose_serving_route_fallback.py`
- [X] T059 [P] [US3] Add integration tests proving tracking/ReID authority reuse without duplicate IDs in `backend\tests\integration\test_tracking_reid_authority_reuse.py`
- [X] T060 [P] [US3] Add system test for route failover under live load in `backend\tests\system\test_live_route_failover.py`
- [X] T061 [P] [US3] Add frontend integration test for config-driven state behavior in `frontend\tests\integration\settingsPoseBehaviorConfig.test.tsx`
- [X] T062 [P] [US3] Add contract test for retention and triage state permissions in `backend\tests\contract\test_behavior_retention_access_contract.py`

### Implementation for User Story 3

- [X] T063 [US3] Add full pose-behavior `.env` config loading and guardrails in `backend\apps\pipeline\config.py` and `docs\backend\apps\pipeline\config.md`
- [X] T064 [P] [US3] Add route policy service for OpenVINO/Triton primary-secondary selection in `backend\apps\pipeline\model_lifecycle\route_policy.py` and `docs\backend\apps\pipeline\model_lifecycle\route_policy.md`
- [X] T065 [P] [US3] Add route health and fallback observability hooks in `backend\apps\health\services.py` and `docs\backend\apps\health\services.md`
- [X] T066 [US3] Add deterministic timeout/retry/fallback execution wrapper in `backend\apps\pipeline\services\serving_router.py` and `docs\backend\apps\pipeline\services\serving_router.md`
- [X] T067 [US3] Enforce no-duplication integration boundary with existing contrastive/detection paths in `backend\apps\pipeline\pipeline_service.py` and `docs\backend\apps\pipeline\pipeline_service.md`
- [X] T068 [US3] Implement retention policy and role-based access checks for behavior artifacts in `backend\apps\anomalies\views.py`, `backend\apps\anomalies\serializers.py`, and `docs\backend\apps\anomalies\views.md`
- [X] T069 [US3] Expose triage transitions (flagged/confirmed/dismissed) with audit notes in `backend\apps\anomalies\services.py` and `docs\backend\apps\anomalies\services.md`
- [X] T070 [US3] Add frontend settings controls for pose-behavior runtime parameters in `frontend\src\pages\SettingsPage.tsx` and `docs\frontend\src\pages\SettingsPage.md`
- [X] T071 [P] [US3] Add frontend typed route/fallback status rendering in `frontend\src\components\health\HealthBadge.tsx` and `docs\frontend\src\components\health\HealthBadge.md`
- [X] T072 [US3] Record config-and-fallback validation evidence in `specs/007-pose-behavior-pipeline/evidence/us3-config-failover.md`

**Checkpoint**: US3 independently proves configurable, reusable, and resilient operation.

---

## Phase 6: User Story 4 - Produce Future-Ready Behavioral Representation (Priority: P2)

**Goal**: Emit normalized spatial-temporal skeleton representations and interaction descriptors compatible with future graph/sequence model stages.

**Independent Test**: Validate exported representation schema includes normalized 17-keypoint sequences, spatial-temporal links, and interaction descriptors usable without contract redesign.

### Tests for User Story 4 (MANDATORY) 🔴

- [X] T073 [P] [US4] Add unit tests for spatial normalization and torso-centered transforms in `backend\tests\unit\pipeline\test_spatial_normalization.py`
- [X] T074 [P] [US4] Add unit tests for temporal alignment and sequence assembly in `backend\tests\unit\pipeline\test_temporal_sequence_builder.py`
- [X] T075 [P] [US4] Add contract tests for skeleton graph representation schema in `backend\tests\contract\test_skeleton_graph_contract.py`
- [X] T076 [P] [US4] Add integration tests for interaction descriptor generation in `backend\tests\integration\test_interaction_descriptor_generation.py`
- [X] T077 [P] [US4] Add offline system test for representation export compatibility in `backend\tests\system\test_representation_export_compatibility.py`

### Implementation for User Story 4

- [X] T078 [US4] Implement spatial normalization utilities in `backend\apps\pipeline\services\spatial_normalization.py` and `docs\backend\apps\pipeline\services\spatial_normalization.md`
- [X] T079 [US4] Implement temporal normalization and sequence alignment utilities in `backend\apps\pipeline\services\temporal_normalization.py` and `docs\backend\apps\pipeline\services\temporal_normalization.md`
- [X] T080 [US4] Implement spatial-temporal skeleton graph builder in `backend\apps\pipeline\services\skeleton_graph_builder.py` and `docs\backend\apps\pipeline\services\skeleton_graph_builder.md`
- [X] T081 [P] [US4] Implement representation export schema for downstream models in `backend\apps\pipeline\schemas\skeleton_representation.py` and `docs\backend\apps\pipeline\schemas\skeleton_representation.md`
- [X] T082 [US4] Integrate interaction descriptors into fused timeline representation in `backend\apps\pipeline\services\interaction_features.py` and `docs\backend\apps\pipeline\services\interaction_features.md`
- [X] T083 [US4] Add representation retrieval endpoint for offline/live analysis in `backend\apps\pipeline\views.py` and `backend\apps\pipeline\urls.py`
- [X] T084 [US4] Add frontend representation inspection panel for ML review in `frontend\src\pages\PredictionsPage.tsx` and `docs\frontend\src\pages\PredictionsPage.md`
- [X] T085 [P] [US4] Add frontend representation typing contracts in `frontend\src\types\behavior.ts` and `docs\frontend\src\types\behavior.md`
- [X] T086 [US4] Record representation compatibility evidence in `specs/007-pose-behavior-pipeline/evidence/us4-representation-compatibility.md`

**Checkpoint**: US4 independently proves future-ready representation outputs.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Final hardening, evidence consolidation, and release-readiness checks.

- [X] T087 [P] Run full backend test suite and coverage report, recording output in `specs/007-pose-behavior-pipeline/evidence/final/backend-test-results.md`
- [X] T088 [P] Run full frontend test/e2e/type-check/build and record output in `specs/007-pose-behavior-pipeline/evidence/final/frontend-test-results.md`
- [X] T089 [P] Validate live p95 latency, student coverage, and identity continuity metrics in `specs/007-pose-behavior-pipeline/evidence/final/live-metrics.md`
- [X] T090 [P] Validate offline frame coverage and event consistency metrics in `specs/007-pose-behavior-pipeline/evidence/final/offline-metrics.md`
- [X] T091 [P] Validate serving fallback metrics and schema consistency in `specs/007-pose-behavior-pipeline/evidence/final/fallback-metrics.md`
- [X] T092 [P] Validate retention/authorization controls and triage auditability in `specs/007-pose-behavior-pipeline/evidence/final/security-retention.md`
- [X] T093 [P] Update architecture and module documentation diagrams for pose-behavior flow in `docs\architecture\pose-behavior-pipeline.md`
- [X] T094 [P] Update backend module READMEs touched by this feature in `backend\apps\pipeline\README.md`, `backend\apps\anomalies\README.md`, and `backend\apps\video_analysis\README.md`
- [X] T095 [P] Update frontend module documentation for timeline/event surfaces in `docs\frontend\README.md` and `frontend\README.md`
- [X] T096 [P] Validate markdown cross-links and Mermaid rendering for changed docs in `specs/007-pose-behavior-pipeline/evidence/final/docs-validation.md`
- [X] T097 Verify quickstart execution and attach run notes in `specs/007-pose-behavior-pipeline/evidence/final/quickstart-validation.md`
- [X] T098 Compile final regression evidence pack and reviewer checklist in `specs/007-pose-behavior-pipeline/evidence/final/evidence-pack.md`

---

## Phase 8: Constitution & Metrics Release Gates

**Purpose**: Close constitution MUST gates and missing measurable coverage found during cross-artifact analysis.

- [X] T099 Create per-task commit evidence ledger proving commit-after-every-modification in `specs/007-pose-behavior-pipeline/evidence/final/commit-evidence.md`
- [X] T100 [P] Add CI gate to validate commit ledger completeness in `scripts/ci/verify_commit_evidence.py` and `docs/scripts/ci/verify_commit_evidence.md`
- [X] T101 [P] Audit touched Markdown files for all constitution-mandated diagram types in `specs/007-pose-behavior-pipeline/evidence/final/diagram-type-audit.md`
- [X] T102 [P] Audit touched Markdown files for required detailed diagram explanations in `specs/007-pose-behavior-pipeline/evidence/final/diagram-explanation-audit.md`
- [X] T103 [P] Enforce and record 100% line and branch coverage gate result in `specs/007-pose-behavior-pipeline/evidence/final/coverage-gate.md`
- [X] T104 [P] If coverage is below 100%, document approved exceptions with owner/expiry/removal plan in `specs/007-pose-behavior-pipeline/evidence/final/coverage-exceptions.md` and `specs/007-pose-behavior-pipeline/plan.md`
- [X] T105 Implement ONNX to OpenVINO IR conversion workflow and verification in `backend\apps\pipeline\model_lifecycle\rtmpose_convert_openvino.py` and `docs\backend\apps\pipeline\model_lifecycle\rtmpose_convert_openvino.md`
- [X] T106 Implement ONNX to TensorRT conversion workflow and verification in `backend\apps\pipeline\model_lifecycle\rtmpose_convert_tensorrt.py` and `docs\backend\apps\pipeline\model_lifecycle\rtmpose_convert_tensorrt.md`
- [X] T107 Add jitter-reduction benchmark task proving >=30% reduction in `backend\tests\system\test_pose_jitter_reduction_benchmark.py` and `specs/007-pose-behavior-pipeline/evidence/final/jitter-benchmark.md`
- [X] T108 Add fusion-window consistency benchmark proving >=95% valid fused outputs in `backend\tests\system\test_fusion_window_consistency.py` and `specs/007-pose-behavior-pipeline/evidence/final/fusion-consistency.md`
- [X] T109 Add labeled pair/group association benchmark proving >=90% correctness in `backend\tests\system\test_group_association_accuracy.py` and `specs/007-pose-behavior-pipeline/evidence/final/group-association.md`
- [X] T110 Add explicit Windows-vs-Linux output equivalence validation in `backend\tests\system\test_windows_linux_equivalence.py` and `specs/007-pose-behavior-pipeline/evidence/final/platform-equivalence.md`
- [X] T111 Add supported-load benchmark at 50 students for live p95 timeline updates in `backend\tests\system\test_live_50_student_load.py` and `specs/007-pose-behavior-pipeline/evidence/final/live-50-student-load.md`

## Phase 9: Artifact Delivery + Integration Closure

**Purpose**: Close remaining implementation gaps with real RTMPose artifacts and end-to-end integration evidence.

- [X] T112 Download RTMPose medium source artifact and register provenance in `specs/007-pose-behavior-pipeline/evidence/final/model-provenance.md`
- [X] T113 Produce RTMPose ONNX artifact in `backend/models/rtmpose/rtmpose-m.onnx` and update `backend/apps/pipeline/model_lifecycle/rtmpose_manifest.py`
- [X] T114 Produce RTMPose OpenVINO IR artifacts in `backend/models/rtmpose/openvino/rtmpose-m.xml` and `backend/models/rtmpose/openvino/rtmpose-m.bin`
- [X] T115 Produce RTMPose TensorRT engine artifact in `backend/models/rtmpose/tensorrt/rtmpose-m.engine`
- [X] T116 Replace conversion stubs with executable conversion workflows in `backend/apps/pipeline/model_lifecycle/rtmpose_convert_openvino.py` and `backend/apps/pipeline/model_lifecycle/rtmpose_convert_tensorrt.py`
- [X] T117 Add artifact existence and format sanity tests in `backend/tests/system/test_rtmpose_artifact_presence.py`
- [X] T118 Execute targeted pose-behavior backend integration tests and record output in `specs/007-pose-behavior-pipeline/evidence/final/integration-regression.md`
- [X] T119 Execute targeted frontend integration/e2e timeline tests and record output in `specs/007-pose-behavior-pipeline/evidence/final/frontend-integration-regression.md`
- [X] T120 Reconcile and complete all unchecked tasks in phases 7/8 using real run outputs (replace placeholders) across `specs/007-pose-behavior-pipeline/evidence/final/*.md`

## Phase 10: Source-Frame ROI Pose Integration Contract

**Purpose**: Ensure RTMPose integration with the general inference pipeline uses a deterministic source-frame ROI flow (for any camera feed dimensions) and preserves coordinate correctness.

- [X] T121 Implement person-box detection stage on original source frames before pose estimation in `backend/apps/pipeline/services/pipeline_service.py` and `backend/apps/pipeline/services/pose_runtime.py`
- [X] T122 Implement per-person ROI crop from original source frame and deterministic resize to RTMPose input (`256x192` unless overridden) in `backend/apps/pipeline/services/pose_runtime.py` and `backend/apps/pipeline/config.py`
- [X] T123 Implement keypoint reverse-mapping from RTMPose crop coordinates back to original source-frame coordinates in `backend/apps/pipeline/services/pose_runtime.py` and `backend/apps/pipeline/services/signal_fusion.py`
- [X] T124 Add system regression test for source-frame ROI -> RTMPose -> mapped-keypoints contract across variable input dimensions in `backend/tests/system/test_rtmpose_variable_resolution_roi_mapping.py`
- [X] T125 Add documentation evidence for the five-step source-frame integration flow and runtime assumptions in `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-source-frame-contract.md`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: Starts immediately.
- **Phase 2 (Foundational)**: Depends on Phase 1 completion and blocks all user story phases.
- **Phase 3-6 (User Stories)**: Depend on Phase 2 completion.
- **Phase 7 (Polish)**: Depends on completion of all selected user stories.
- **Phase 8 (Constitution & Metrics Gates)**: Depends on Phase 7 and is required before `/speckit-implement`.

### User Story Dependencies

- **US1 (P1)**: Can start after Foundational; no dependency on other user stories.
- **US2 (P1)**: Can start after Foundational; independent contract but should keep parity with US1 output schema.
- **US3 (P2)**: Can start after Foundational; integrates with US1/US2 runtime paths for config/fallback validation.
- **US4 (P2)**: Can start after Foundational; should consume stable signal/event contracts from US1/US2.

### Within Each User Story

- Tests first (RED) -> implementation -> integration wiring -> evidence capture.
- Data/schema work before API endpoints.
- Backend contracts before frontend consumption.

---

## Parallel Opportunities

- Setup: T002-T007 can run in parallel after T001.
- Foundational tests: T009-T013 parallel.
- US1 tests: T025-T031 parallel.
- US2 tests: T042-T047 parallel.
- US3 tests: T057-T062 parallel.
- US4 tests: T073-T077 parallel.
- Final validation docs/evidence tasks: T087-T096 parallel where non-overlapping.
- Constitution/metrics gates: T100-T104 parallel, then T107-T111 parallel where resources allow.

## Parallel Example: User Story 1

```text
Task: T025 Add stabilization unit tests
Task: T026 Add behavior feature unit tests
Task: T027 Add live signal contract tests
Task: T030 Add frontend live timeline integration test
Task: T031 Add frontend live e2e scenario
```

## Parallel Example: User Story 2

```text
Task: T042 Add offline temporal buffer unit tests
Task: T043 Add offline artifact contract tests
Task: T046 Add recording timeline frontend integration test
Task: T047 Add offline review e2e scenario
```

## Parallel Example: User Story 3

```text
Task: T057 Add env contract tests
Task: T058 Add route fallback integration tests
Task: T059 Add tracking/ReID reuse integration tests
Task: T061 Add settings config frontend integration test
```

## Parallel Example: User Story 4

```text
Task: T073 Add spatial normalization tests
Task: T074 Add temporal sequence tests
Task: T075 Add skeleton graph contract tests
Task: T076 Add interaction descriptor integration tests
```

---

## Implementation Strategy

### MVP First (US1)

1. Complete Phase 1 and Phase 2.
2. Complete US1 (Phase 3) and validate independently.
3. Demo/deploy MVP live timeline and cheating event capability.

### Incremental Delivery

1. Deliver US1 (live).
2. Deliver US2 (offline parity).
3. Deliver US3 (config + fallback + no-duplication hardening).
4. Deliver US4 (future-ready representation).
5. Execute Phase 7 release validation and evidence packaging.
6. Execute Phase 8 constitution and metrics gates before implementation handoff.

### Parallel Team Strategy

1. Team completes Setup + Foundational together.
2. Then split ownership:
   - Engineer A: US1 live pipeline
   - Engineer B: US2 offline pipeline
   - Engineer C: US3 routing/config hardening
   - Engineer D: US4 representation outputs
3. Merge into Phase 7 with shared validation and evidence review.

---

## Notes

- Keep all tasks traceable to the relevant user story and contracts.
- Preserve existing tracking/ReID and contrastive paths; do not duplicate these capabilities.
- Keep learned anomaly models out of this feature scope.
- Every source/test change must include corresponding docs update task coverage.
- Ensure both Windows development and Linux production assumptions are validated in evidence.
- Phase 8 tasks are blocking release gates added from `/speckit-analyze` findings and must be completed before `/speckit-implement`.

