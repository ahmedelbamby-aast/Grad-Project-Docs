# Tasks: Human Pose Kinematics Layer

**Input**: Design documents from `specs/013-human-pose-kinematics/`
**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [contracts/pose-kinematics-contract.md](contracts/pose-kinematics-contract.md), [quickstart.md](quickstart.md)

**Tests and Evidence**: Tests are mandatory for this feature because the
specification requires reviewer-labeled agreement, live/offline compatibility,
config-driven override gates, telemetry, artifact lineage, and production
benchmark evidence. Local tests are contract evidence only; production
acceptance requires a native Linux RTX 5090 run.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel after dependencies are satisfied.
- **[Story]**: Required only for user-story phases.
- Every task includes at least one exact target file path.

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Establish flags, module boundaries, and reproducible scaffolding without enabling behavior by default.

- [X] T001 Add bounded `POSE_KINEMATICS_*` settings with defaults in `backend/config/settings/base.py`
- [X] T002 Add production profile env placeholders with disabled default in `tools/prod/prod_enable_parallel_flow.sh`
- [X] T003 [P] Create deterministic kinematics module skeleton in `backend/apps/pipeline/services/pose_kinematics.py`
- [X] T004 [P] Create kinematics configuration helper skeleton in `backend/apps/pipeline/services/pose_kinematics_config.py`
- [X] T005 [P] Create artifact writer skeleton in `backend/apps/video_analysis/services/pose_kinematics_artifacts.py`
- [X] T006 [P] Create persistence helper skeleton in `backend/apps/video_analysis/services/pose_kinematics_persistence.py`
- [X] T007 [P] Create production benchmark wrapper skeleton in `tools/prod/prod_run_pose_kinematics_benchmark.sh`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Shared schema, config, contracts, and test fixtures that all user stories depend on.

**Critical**: No user story work can be claimed complete until this phase is complete.

- [X] T008 Add `PoseKinematicsRecord` and `PoseKinematicsOverrideEvent` Django models in `backend/apps/video_analysis/models.py`
- [X] T009 Add Django migration for pose kinematics models and indexes in `backend/apps/video_analysis/migrations/0015_pose_kinematics.py`
- [X] T010 [P] Register pose kinematics models in Django admin in `backend/apps/video_analysis/admin.py`
- [X] T011 [P] Add contract tests for `pose_kinematics.v1` summary, artifact, override, and config payloads in `backend/tests/contract/test_pose_kinematics_contract.py`
- [X] T012 [P] Add unit tests proving settings are read from env/config and not hardcoded in `backend/tests/unit/config/test_pose_kinematics_settings.py`
- [X] T013 [P] Add reusable COCO keypoint fixtures, reviewer-labeled validation manifest builders, and 500 person-frame sample-count fixture metadata in `backend/tests/fixtures/pose_kinematics.py`
- [X] T014 Add idempotency-key and unique-scope tests for kinematics DB writes in `backend/tests/unit/video_analysis/test_pose_kinematics_models.py`
- [X] T015 Add telemetry field expectations for kinematics counters and latency in `backend/tests/contract/test_pose_kinematics_telemetry_contract.py`
- [X] T016 Extend benchmark collector schema for kinematics metrics in `tools/prod/prod_collect_benchmark_metrics.py`
- [X] T017 Extend live watcher table for kinematics metrics with bounded-cell rendering in `tools/prod/prod_watch_benchmark_metrics.sh`

**Checkpoint**: Foundation ready; user story implementation may begin.

---

## Phase 3: User Story 1 - Explain Pose Mechanics Per Student (Priority: P1)

**Goal**: Produce understandable per-student, per-frame pose mechanics evidence from RTMPose keypoints.

**Independent Test**: Replay or fixture-inject pose records and verify every pose-eligible track/frame receives quality, anchors, geometry, orientation, posture, gesture, attention-support, and physical-validity evidence without changing terminal job state.

### Validation and Evidence for User Story 1

- [X] T018 [P] [US1] Add keypoint quality and missingness unit tests in `backend/tests/unit/pipeline/test_pose_kinematics_quality.py`
- [X] T019 [P] [US1] Add body anchors, geometry, joint-angle, and limb-vector unit tests in `backend/tests/unit/pipeline/test_pose_kinematics_geometry.py`
- [X] T020 [P] [US1] Add torso/head orientation unit tests in `backend/tests/unit/pipeline/test_pose_kinematics_orientation.py`
- [X] T021 [P] [US1] Add posture and hand-raise unit tests in `backend/tests/unit/pipeline/test_pose_kinematics_posture_gesture.py`
- [X] T022 [P] [US1] Add physical-validity and missing-joint estimation unit tests in `backend/tests/unit/pipeline/test_pose_kinematics_constraints.py`
- [X] T023 [P] [US1] Add compact summary serialization contract tests in `backend/tests/contract/test_pose_kinematics_summary_api.py`

### Implementation for User Story 1

- [X] T024 [US1] Implement keypoint quality labels and reason codes in `backend/apps/pipeline/services/pose_kinematics.py`
- [X] T025 [US1] Implement body anchors, normalized geometry, joint angles, and limb vectors in `backend/apps/pipeline/services/pose_kinematics.py`
- [X] T026 [US1] Implement torso/head orientation, head-torso alignment, and confidence scoring in `backend/apps/pipeline/services/pose_kinematics.py`
- [X] T027 [US1] Implement posture, hand-raise, and attention-support mechanics in `backend/apps/pipeline/services/pose_kinematics.py`
- [X] T028 [US1] Implement physical constraint violations and bounded missing-joint estimation in `backend/apps/pipeline/services/pose_kinematics.py`
- [X] T029 [US1] Implement compact summary persistence and artifact references in `backend/apps/video_analysis/services/pose_kinematics_persistence.py`
- [X] T030 [US1] Implement full keypoint evidence artifact writing with digests in `backend/apps/video_analysis/services/pose_kinematics_artifacts.py`
- [X] T031 [US1] Wire offline pose artifact enrichment after RTMPose output in `backend/apps/video_analysis/tasks.py`
- [X] T032 [US1] Expose `pose_kinematics` compact summary fields in frame API payloads in `backend/apps/video_analysis/views.py`
- [X] T033 [US1] Add frontend/export compatibility fields for `pose_kinematics` states in `backend/apps/video_analysis/serializers.py`

**Checkpoint**: User Story 1 is independently functional and testable.

---

## Phase 4: User Story 2 - Stabilize Pose Signals Across Time (Priority: P2)

**Goal**: Use bounded same-track history to smooth pose evidence, detect spikes, and preserve genuine transitions.

**Independent Test**: Feed a labeled sequence with jitter and real transitions and verify smoothed states reduce flicker without flattening true pose changes, while no accepted history window exceeds the configured bound.

### Validation and Evidence for User Story 2

- [X] T034 [P] [US2] Add bounded-history eviction unit tests in `backend/tests/unit/pipeline/test_pose_kinematics_history.py`
- [X] T035 [P] [US2] Add smoothing, velocity, stability, and sudden-jump unit tests in `backend/tests/unit/pipeline/test_pose_kinematics_temporal.py`
- [X] T036 [P] [US2] Add live-stream gap and reconnect compatibility tests in `backend/tests/integration/test_pose_kinematics_live_streaming.py`
- [X] T037 [P] [US2] Add offline deterministic replay comparison tests in `backend/tests/integration/test_pose_kinematics_offline_replay.py`
- [X] T038 [P] [US2] Add history-bound telemetry contract tests in `backend/tests/contract/test_pose_kinematics_history_contract.py`

### Implementation for User Story 2

- [X] T039 [US2] Implement bounded per-track temporal buffer with time and sample eviction in `backend/apps/pipeline/services/pose_kinematics_history.py`
- [X] T040 [US2] Implement confidence-weighted smoothing and motion velocity in `backend/apps/pipeline/services/pose_kinematics.py`
- [X] T041 [US2] Implement sudden-jump and spike-vs-transition classification in `backend/apps/pipeline/services/pose_kinematics.py`
- [X] T042 [US2] Record history window duration, sample count, and bound violations in `backend/apps/video_analysis/services/pose_kinematics_persistence.py`
- [X] T043 [US2] Wire bounded history into offline pose tail processing in `backend/apps/video_analysis/tasks.py`
- [X] T044 [US2] Wire append-only bounded history into live pose payload processing in `backend/apps/video_analysis/tasks.py`
- [X] T045 [US2] Add degraded/unavailable live-state projection for kinematics gaps in `backend/apps/video_analysis/consumers.py`
- [X] T046 [US2] Add history metrics to benchmark collector output in `tools/prod/prod_collect_benchmark_metrics.py`
- [X] T047 [US2] Add history metrics to live watcher output in `tools/prod/prod_watch_benchmark_metrics.sh`

**Checkpoint**: User Stories 1 and 2 work independently and together.

---

## Phase 5: User Story 3 - Provide Fusion-Ready Pose Evidence (Priority: P3)

**Goal**: Provide traceable support, contradiction, unavailable, and override-applied evidence for higher-level fusion without hiding original model predictions.

**Independent Test**: Feed pose and gaze/behavior evidence into isolated fusion review and verify every support, contradiction, blocked override, and applied override records original prediction, pose basis, confidence values, config gate values, bounded-history support, and root-cause category.

### Validation and Evidence for User Story 3

- [X] T048 [P] [US3] Add override-gate config unit tests in `backend/tests/unit/pipeline/test_pose_kinematics_override_gate.py`
- [X] T049 [P] [US3] Add contradiction and root-cause classification tests in `backend/tests/unit/pipeline/test_pose_kinematics_fusion.py`
- [X] T050 [P] [US3] Add override event persistence and idempotency tests in `backend/tests/unit/video_analysis/test_pose_kinematics_override_events.py`
- [X] T051 [P] [US3] Add LPM/fusion integration tests preserving original predictions in `backend/tests/integration/test_pose_kinematics_fusion_flow.py`
- [X] T052 [P] [US3] Add reviewer-label agreement harness tests enforcing at least 500 person-frame samples plus posture, hand raise, orientation, and attention-support thresholds in `backend/tests/integration/test_pose_kinematics_reviewer_labels.py`
- [X] T053 [P] [US3] Add contract tests for monitoring counters and blocked-override categories in `backend/tests/contract/test_pose_kinematics_override_contract.py`

### Implementation for User Story 3

- [X] T054 [US3] Implement config-driven override gate evaluator in `backend/apps/pipeline/services/pose_kinematics_config.py`
- [X] T055 [US3] Implement fusion pose signal builder and contradiction categories in `backend/apps/pipeline/services/pose_kinematics.py`
- [X] T056 [US3] Persist override-applied and override-blocked events in `backend/apps/video_analysis/services/pose_kinematics_persistence.py`
- [X] T057 [US3] Integrate pose fusion signals with logical path matrix inputs without mutating source predictions in `backend/apps/pipeline/services/logical_path_matrix.py`
- [X] T058 [US3] Wire contradiction and override summaries into offline artifacts in `backend/apps/video_analysis/tasks.py`
- [X] T059 [US3] Expose contradiction, blocked override, and override-applied states in API/export payloads in `backend/apps/video_analysis/views.py`
- [X] T060 [US3] Add aggregate override and contradiction metrics to telemetry writer payloads in `backend/apps/telemetry/writer.py`

**Checkpoint**: All user stories are independently functional.

---

## Phase 6: Polish and Cross-Cutting Concerns

**Purpose**: Documentation, production validation, rollback, and final evidence.

- [X] T061 [P] Update feature quickstart commands and expected metrics in `specs/013-human-pose-kinematics/quickstart.md`
- [X] T062 [P] Update module documentation for kinematics services in `backend/apps/pipeline/README.md`
- [X] T063 [P] Update video-analysis module documentation for persistence, API, and artifacts in `backend/apps/video_analysis/README.md`
- [X] T064 [P] Add or update entity documentation for the pose kinematics feature in `docs/entity/cycles/cycle_013_human_pose_kinematics.md`
- [X] T065 Add production offline benchmark matrix wrapper for baseline-disabled and candidate-enabled `combined.mp4` runs with `new-attempt` replay policy in `tools/prod/prod_run_pose_kinematics_benchmark.sh`
- [X] T066 Run local PostgreSQL-backed unit, contract, and integration validation commands from `specs/013-human-pose-kinematics/quickstart.md`
- [ ] T067 Run production Linux RTX 5090 offline baseline-disabled and candidate-enabled benchmark matrix and collect evidence using `tools/prod/prod_run_pose_kinematics_benchmark.sh`
- [ ] T068 Append offline baseline/candidate benchmark decision table, reviewer-label agreement table, runtime reconciliation table, and artifact paths to `docs/production_inference_benchmark.md`
- [X] T069 Add and run pose-kinematics hardening gate for hidden xfails, SQLite fallback, hardcoded override thresholds, unbounded history, unsupported live assumptions, and env/config drift in `scripts/ci/verify_pose_kinematics_requirements_gates.py`
- [X] T070 Add production live-profile validation wrapper for real RTSP/RTSPS/WHEP media or governed live-capture manifest in `tools/prod/prod_run_pose_kinematics_live_validation.sh`
- [ ] T071 Run production live-profile validation and collect session latency, frame/drop/gap, unavailable/degraded, telemetry, artifact, and evidence-path metrics using `tools/prod/prod_run_pose_kinematics_live_validation.sh`
- [X] T072 Add runtime reconciliation checker joining task, queue, PostgreSQL, artifact, telemetry, and API/export states in `tools/prod/prod_check_pose_kinematics_reconciliation.py`
- [ ] T073 Run runtime reconciliation checker for both offline benchmark matrix and live validation evidence using `tools/prod/prod_check_pose_kinematics_reconciliation.py`
- [X] T074 Add reviewer-label agreement report generator with manifest digest, sample count, per-model threshold results, and artifact path in `tools/prod/prod_collect_pose_kinematics_label_agreement.py`
- [ ] T075 Run reviewer-label agreement report on the accepted validation manifest and attach results to production evidence using `tools/prod/prod_collect_pose_kinematics_label_agreement.py`
- [X] T076 Add disabled-layer rollback verification proving existing pose/behavior flows continue with explicit `unavailable` kinematics state in `tools/prod/prod_verify_pose_kinematics_rollback.sh`
- [ ] T077 Run disabled-layer rollback verification and document rollback command, env delta, job/session identity, and evidence paths using `tools/prod/prod_verify_pose_kinematics_rollback.sh`

---

## Dependencies and Execution Order

### Phase Dependencies

- **Phase 1 Setup**: No dependencies.
- **Phase 2 Foundational**: Depends on Phase 1.
- **US1**: Depends on Phase 2 and is the MVP.
- **US2**: Depends on Phase 2 and can begin after the shared schema/config contracts exist; full integration benefits from US1 services.
- **US3**: Depends on Phase 2 and is safest after US1/US2 provide evidence and bounded history primitives.
- **Polish**: Depends on all desired user stories.

### User Story Dependencies

- **US1 (P1)**: First deliverable; creates core mechanics evidence.
- **US2 (P2)**: Uses US1 outputs plus bounded history for smoothing and spike handling.
- **US3 (P3)**: Uses US1 outputs and US2 history to make fusion/override decisions.

### Parallel Opportunities

- Setup tasks T003-T007 can run in parallel after T001/T002 are understood.
- Foundational tests T011-T015 can run in parallel after model field names are agreed.
- US1 test tasks T018-T023 can run in parallel.
- US2 test tasks T034-T038 can run in parallel.
- US3 test tasks T048-T053 can run in parallel.
- Documentation tasks T061-T064 can run in parallel after implementation stabilizes.

## Parallel Examples

### User Story 1

```bash
# Parallelizable after Phase 2:
T018 backend/tests/unit/pipeline/test_pose_kinematics_quality.py
T019 backend/tests/unit/pipeline/test_pose_kinematics_geometry.py
T020 backend/tests/unit/pipeline/test_pose_kinematics_orientation.py
T021 backend/tests/unit/pipeline/test_pose_kinematics_posture_gesture.py
T022 backend/tests/unit/pipeline/test_pose_kinematics_constraints.py
T023 backend/tests/contract/test_pose_kinematics_summary_api.py
```

### User Story 2

```bash
# Parallelizable after Phase 2:
T034 backend/tests/unit/pipeline/test_pose_kinematics_history.py
T035 backend/tests/unit/pipeline/test_pose_kinematics_temporal.py
T036 backend/tests/integration/test_pose_kinematics_live_streaming.py
T037 backend/tests/integration/test_pose_kinematics_offline_replay.py
T038 backend/tests/contract/test_pose_kinematics_history_contract.py
```

### User Story 3

```bash
# Parallelizable after Phase 2:
T048 backend/tests/unit/pipeline/test_pose_kinematics_override_gate.py
T049 backend/tests/unit/pipeline/test_pose_kinematics_fusion.py
T050 backend/tests/unit/video_analysis/test_pose_kinematics_override_events.py
T051 backend/tests/integration/test_pose_kinematics_fusion_flow.py
T052 backend/tests/integration/test_pose_kinematics_reviewer_labels.py
T053 backend/tests/contract/test_pose_kinematics_override_contract.py
```

## Implementation Strategy

### MVP First

1. Complete Phase 1 and Phase 2.
2. Complete US1 only.
3. Validate one compact pose mechanics record per pose-eligible track/frame.
4. Keep `POSE_KINEMATICS_ENABLED=0` by default until production evidence exists.

### Incremental Delivery

1. US1: explainable mechanics evidence.
2. US2: bounded temporal stability and spike detection.
3. US3: fusion-ready support/contradiction/override evidence.
4. Polish: production benchmark, docs, rollback, and final evidence.

### Acceptance Boundary

Code completion is not acceptance. Acceptance requires the production offline
benchmark matrix in T067, documented evidence in T068, hardening checks in
T069, live production validation in T071, runtime reconciliation in T073,
reviewer-label agreement evidence in T075, and rollback verification in T077.
