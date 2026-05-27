# Tasks: Behavioral Semantic Intelligence Layer

**Input**: Design documents from `specs/011-bsil-semantic-runtime/`  
**Prerequisites**: [spec.md](spec.md), [plan.md](plan.md), governance contracts in this directory

**Tests and Evidence**: Tests are mandatory for BSIL because the specification
requires production evidence, runtime reconciliation, scientific validation,
lineage, replay, confidence bands, human-reviewed escalation, full raw
retention and no-fake-maturity gates. Mocks may support unit isolation but
cannot close production or scientific maturity.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Task can run in parallel because it touches different files and
  does not depend on incomplete work.
- **[Story]**: Required only for user story phases.
- Every task includes an explicit file path.

## Phase 1: Setup and Governance Foundation

**Purpose**: Preserve current BSIL plan/spec work while making governance,
contract, and active feature references explicit.

- [X] T001 Update active feature reference in `.specify/feature.json`
- [X] T002 Update active Spec Kit plan reference in `AGENTS.md`
- [X] T003 [P] Add BSIL glossary and prohibited accusation terms to `specs/011-bsil-semantic-runtime/behavioral_lineage_contract.md`
- [X] T004 [P] Add confidence-band definitions to `specs/011-bsil-semantic-runtime/behavioral_lineage_contract.md`
- [X] T005 [P] Add full raw retention policy to `specs/011-bsil-semantic-runtime/evidence_contract.md`
- [X] T006 [P] Add broad authenticated staff access audit policy to `specs/011-bsil-semantic-runtime/runtime_governance.md`
- [X] T007 [P] Add governed review-label policy to `specs/011-bsil-semantic-runtime/dataset_governance.md`
- [X] T008 [P] Add human-reviewed escalation gate to `specs/011-bsil-semantic-runtime/acceptance_gates.md`
- [X] T009 [P] Add confidence-band dashboard requirements to `specs/011-bsil-semantic-runtime/observability_contract.md`
- [X] T010 Add BSIL debt, xfail, and drift registry requirements to `specs/011-bsil-semantic-runtime/acceptance_gates.md`

## Phase 2: Foundational Runtime and Data Contracts

**Purpose**: Blocking prerequisites that every user story depends on.

- [X] T011 Define BSIL enum contracts for confidence bands, truth states, failure taxonomy, and episode lifecycle in `backend/apps/behavior/contracts.py`
- [X] T012 Define typed event envelope schemas for BSIL semantic, temporal, episode, graph, anomaly, access-audit, and review-label events in `backend/apps/behavior/event_schemas.py`
- [X] T013 Define PostgreSQL models for `SemanticPoseState`, `BehavioralConfidenceBreakdown`, and `AttentionTimeline` in `backend/apps/behavior/models.py`
- [X] T014 Define PostgreSQL models for `TemporalReasoningWindow`, `BehavioralState`, and `BehavioralEpisode` in `backend/apps/behavior/models.py`
- [X] T015 Define PostgreSQL models for `InteractionEdge`, `DecisionLineageRecord`, and `BehavioralReviewLabel` in `backend/apps/behavior/models.py`
- [X] T016 Create initial BSIL migrations in `backend/apps/behavior/migrations/`
- [X] T017 Define BSIL repository methods with idempotency keys and no direct inference-path writes in `backend/apps/behavior/repositories.py`
- [X] T018 Define BSIL serializers with explicit fields and no broad serializer exposure in `backend/apps/behavior/serializers.py`
- [X] T019 Define BSIL API route stubs for read-only states, episodes, graphs, evidence, access audit, and review labels in `backend/apps/behavior/views.py`
- [X] T020 Register BSIL API routes in `backend/apps/behavior/urls.py`
- [X] T021 Define live/offline semantic, temporal, graph, anomaly, replay, and evidence queue names in `backend/apps/pipeline/queue_contracts.py`
- [X] T022 Add BSIL queue routing declarations and no-cross-mode validation in `backend/apps/pipeline/routing.py`
- [X] T023 Add runtime anti-fragility constants for retry ceilings, stale windows, degraded duration, queue depth, replay drift, identity fragmentation, and interaction ambiguity in `backend/apps/runtime/bsil_limits.py`
- [X] T024 Add BSIL failure taxonomy severity mapping in `backend/apps/runtime/failure_taxonomy.py`
- [X] T025 Add BSIL telemetry metric names and truth-state labels in `backend/apps/runtime/telemetry_contracts.py`
- [X] T026 Add BSIL artifact names and digest requirements in `backend/apps/forensics/bsil_artifacts.py`
- [X] T027 Add CI static guard for BSIL placeholders, hidden xfails, fallback payloads, and SQLite evidence paths in `scripts/ci/verify_bsil_static_gates.py`
- [X] T028 Add production helper evidence collection entrypoint for BSIL in `tools/prod/prod-bsil-evidence.ps1`

## Phase 3: User Story 1 - Interpret Pose Semantics Safely (Priority: P1)

**Goal**: Convert validated pose/keypoint streams into safe semantic pose
states with confidence bands, lineage, uncertainty, and degradation behavior.

**Independent Test**: Representative clear, partial, occluded, missing-lineage,
and low-confidence pose sequences produce semantic states only when lineage
and confidence rules pass.

### Tests and Evidence for User Story 1

- [X] T029 [P] [US1] Add semantic pose schema unit tests in `backend/tests/unit/behavior/test_semantic_pose_contracts.py`
- [X] T030 [P] [US1] Add pose lineage and missing-lineage rejection tests in `backend/tests/unit/behavior/test_semantic_pose_lineage.py`
- [X] T031 [P] [US1] Add confidence-band assignment tests in `backend/tests/unit/behavior/test_confidence_bands.py`
- [X] T032 [P] [US1] Add occlusion, partial-pose, and low-confidence degradation tests in `backend/tests/unit/behavior/test_pose_degradation.py`
- [X] T033 [P] [US1] Add semantic trace artifact validation tests in `backend/tests/contract/test_bsil_semantic_trace_contract.py`
- [X] T034 [US1] Add representative semantic pose fixture manifest in `ci_evidence/bsil/semantic_pose_fixture_manifest.json`

### Implementation for User Story 1

- [X] T035 [P] [US1] Implement pose confidence normalization in `backend/apps/behavior/pose_decision/confidence.py`
- [X] T036 [P] [US1] Implement joint reliability and missing-keypoint policy in `backend/apps/behavior/pose_decision/joint_reliability.py`
- [X] T037 [P] [US1] Implement head attention, gaze category, and body orientation derivation in `backend/apps/behavior/pose_decision/attention.py`
- [X] T038 [P] [US1] Implement posture, hand activity, desk interaction, and motion energy derivation in `backend/apps/behavior/pose_decision/semantics.py`
- [X] T039 [US1] Implement semantic transition event creation in `backend/apps/behavior/pose_decision/transitions.py`
- [X] T040 [US1] Implement semantic pose orchestration service with fail-closed lineage checks in `backend/apps/behavior/services/semantic_pose_service.py`
- [X] T041 [US1] Persist semantic pose states through repository boundary in `backend/apps/behavior/repositories.py`
- [X] T042 [US1] Emit semantic latency, uncertainty, degradation, and band distribution telemetry in `backend/apps/runtime/bsil_metrics.py`
- [X] T043 [US1] Expose semantic pose state read endpoint in `backend/apps/behavior/views.py`
- [X] T044 [US1] Render semantic states, uncertainty, and confidence bands in `frontend/src/features/bsil/SemanticStatePanel.tsx`
- [X] T045 [US1] Add semantic trace export command in `backend/apps/behavior/management/commands/export_bsil_semantic_traces.py`

## Phase 4: User Story 2 - Reason Over Temporal Behavioral State (Priority: P1)

**Goal**: Maintain temporal behavioral memory windows that suppress spikes,
track state transitions, decay uncertainty, and replay deterministically.

**Independent Test**: Replayed sequences with spikes, sustained evidence,
continuity gaps, reconnects, and drift reproduce the same accepted state
transitions within the replay drift budget.

### Tests and Evidence for User Story 2

- [X] T046 [P] [US2] Add temporal window contract tests in `backend/tests/unit/behavior/test_temporal_window_contracts.py`
- [X] T047 [P] [US2] Add spike suppression and hysteresis tests in `backend/tests/unit/behavior/test_temporal_spike_suppression.py`
- [X] T048 [P] [US2] Add semantic decay, cooldown, and uncertainty propagation tests in `backend/tests/unit/behavior/test_temporal_decay.py`
- [X] T049 [P] [US2] Add replay determinism tests in `backend/tests/integration/behavior/test_temporal_replay_determinism.py`
- [X] T050 [P] [US2] Add Redis cache vs PostgreSQL authority reconciliation tests in `backend/tests/integration/behavior/test_online_state_reconciliation.py`
- [X] T051 [US2] Add temporal reasoning audit fixture manifest in `ci_evidence/bsil/temporal_reasoning_fixture_manifest.json`

### Implementation for User Story 2

- [X] T052 [P] [US2] Implement temporal window builder in `backend/apps/behavior/temporal/windows.py`
- [X] T053 [P] [US2] Implement behavior state machine transitions in `backend/apps/behavior/temporal/state_machine.py`
- [X] T054 [P] [US2] Implement hysteresis, cooldown, smoothing, and decay rules in `backend/apps/behavior/temporal/decay.py`
- [X] T055 [P] [US2] Implement timestamp authority and frame ordering checks in `backend/apps/behavior/temporal/time_authority.py`
- [X] T056 [US2] Implement online state cache TTL and stale-window eviction in `backend/apps/behavior/temporal/cache.py`
- [X] T057 [US2] Implement crash-safe state rebuild from durable lineage in `backend/apps/behavior/temporal/rebuild.py`
- [X] T058 [US2] Persist temporal windows and behavioral states through `backend/apps/behavior/repositories.py`
- [X] T059 [US2] Emit temporal aggregation, stale-window, replay-drift, and invalid-window telemetry in `backend/apps/runtime/bsil_metrics.py`
- [X] T060 [US2] Add temporal state read endpoint in `backend/apps/behavior/views.py`
- [X] T061 [US2] Render temporal state timeline and replay mismatch indicators in `frontend/src/features/bsil/TemporalStateTimeline.tsx`
- [X] T062 [US2] Add temporal reasoning audit export command in `backend/apps/behavior/management/commands/export_bsil_temporal_audit.py`

## Phase 5: User Story 3 - Generate Explainable Behavioral Episodes (Priority: P2)

**Goal**: Create rollback-safe, non-accusatory behavioral episodes with causal
evidence chains and human-reviewed escalation for high-confidence episodes.

**Independent Test**: Episode creation, extension, suppression, supersession,
missing evidence, high-confidence review task creation, and duplicate task
replay are all deterministic and auditable.

### Tests and Evidence for User Story 3

- [X] T063 [P] [US3] Add behavioral episode lifecycle tests in `backend/tests/unit/behavior/test_episode_lifecycle.py`
- [X] T064 [P] [US3] Add causal evidence chain validation tests in `backend/tests/unit/behavior/test_episode_lineage.py`
- [X] T065 [P] [US3] Add no-accusation vocabulary tests in `backend/tests/unit/behavior/test_no_accusation_language.py`
- [X] T066 [P] [US3] Add high-confidence human review task tests in `backend/tests/integration/behavior/test_episode_review_tasks.py`
- [X] T067 [P] [US3] Add duplicate event suppression tests in `backend/tests/integration/behavior/test_episode_idempotency.py`
- [X] T068 [US3] Add episode lineage fixture manifest in `ci_evidence/bsil/episode_fixture_manifest.json`

### Implementation for User Story 3

- [X] T069 [P] [US3] Implement episode lifecycle state machine in `backend/apps/behavior/episodes/lifecycle.py`
- [X] T070 [P] [US3] Implement causal evidence chain assembly in `backend/apps/behavior/episodes/evidence_chain.py`
- [X] T071 [P] [US3] Implement episode confidence-band and escalation eligibility rules in `backend/apps/behavior/episodes/confidence.py`
- [X] T072 [US3] Implement human review task creation with idempotency in `backend/apps/behavior/episodes/review_tasks.py`
- [X] T073 [US3] Persist episode records, supersession history, and review task references in `backend/apps/behavior/repositories.py`
- [X] T074 [US3] Emit episode lifecycle and human-review telemetry in `backend/apps/runtime/bsil_metrics.py`
- [X] T075 [US3] Add behavioral episode and review task read endpoints in `backend/apps/behavior/views.py`
- [X] T076 [US3] Render episode cards, review-needed state, and no-accusation wording in `frontend/src/features/bsil/BehavioralEpisodePanel.tsx`
- [X] T077 [US3] Add episode lineage export command in `backend/apps/behavior/management/commands/export_bsil_episode_lineage.py`

## Phase 6: User Story 4 - Reason About Multi-Person Interaction Context (Priority: P2)

**Goal**: Build identity-gated interaction edges and graph context that cannot
propagate across invalid windows or unstable identity continuity.

**Independent Test**: Crowded, crossing, occlusion, re-entry, identity
fragmentation, and graph replay scenarios suppress false associations and
record edge lifecycle lineage.

### Tests and Evidence for User Story 4

- [X] T078 [P] [US4] Add interaction edge lifecycle tests in `backend/tests/unit/behavior/test_interaction_edge_lifecycle.py`
- [X] T079 [P] [US4] Add identity continuity gate tests in `backend/tests/unit/behavior/test_interaction_identity_gates.py`
- [X] T080 [P] [US4] Add invalid-window graph propagation tests in `backend/tests/unit/behavior/test_graph_contamination.py`
- [X] T081 [P] [US4] Add graph replay determinism tests in `backend/tests/integration/behavior/test_interaction_graph_replay.py`
- [X] T082 [P] [US4] Add crowded/crossing/occlusion fixture manifest in `ci_evidence/bsil/interaction_graph_fixture_manifest.json`

### Implementation for User Story 4

- [X] T083 [P] [US4] Implement interaction edge lifecycle and decay semantics in `backend/apps/behavior/graph/edges.py`
- [X] T084 [P] [US4] Implement synchronized gaze and peer attention detection in `backend/apps/behavior/graph/attention_edges.py`
- [X] T085 [P] [US4] Implement motion synchronization and crowd context scoring in `backend/apps/behavior/graph/context.py`
- [X] T086 [P] [US4] Implement identity merge/split and fragmentation gates in `backend/apps/behavior/graph/identity_gates.py`
- [X] T087 [US4] Persist interaction edges and graph audit records in `backend/apps/behavior/repositories.py`
- [X] T088 [US4] Emit graph edge, ambiguity, suppression, and contamination telemetry in `backend/apps/runtime/bsil_metrics.py`
- [X] T089 [US4] Add interaction graph read endpoint in `backend/apps/behavior/views.py`
- [X] T090 [US4] Render interaction graph, ambiguity state, and suppression reasons in `frontend/src/features/bsil/InteractionGraphPanel.tsx`
- [X] T091 [US4] Add interaction graph audit export command in `backend/apps/behavior/management/commands/export_bsil_interaction_graph_audit.py`

## Phase 7: User Story 5 - Apply Adaptive Anomaly Accumulation (Priority: P3)

**Goal**: Add bounded adaptive baselines, anomaly accumulation, drift
quarantine, and governed review labels without hidden self-modifying behavior.

**Independent Test**: Baseline updates, drift quarantine, validation labels,
and threshold shifts are explainable, reconstructable, and never mutate
production behavior directly from reviewer feedback.

### Tests and Evidence for User Story 5

- [ ] T092 [P] [US5] Add baseline snapshot and drift tests in `backend/tests/unit/anomalies/test_bsil_baselines.py`
- [ ] T093 [P] [US5] Add adaptive threshold reconstruction tests in `backend/tests/unit/anomalies/test_bsil_thresholds.py`
- [ ] T094 [P] [US5] Add governed review label tests in `backend/tests/unit/anomalies/test_bsil_review_labels.py`
- [ ] T095 [P] [US5] Add baseline contamination and quarantine tests in `backend/tests/integration/anomalies/test_bsil_baseline_quarantine.py`
- [ ] T096 [P] [US5] Add false-positive and false-negative review fixture manifest in `ci_evidence/bsil/review_label_fixture_manifest.json`

### Implementation for User Story 5

- [ ] T097 [P] [US5] Implement baseline snapshot versioning in `backend/apps/anomalies/bsil_baselines.py`
- [ ] T098 [P] [US5] Implement adaptive threshold and hysteresis rules in `backend/apps/anomalies/bsil_thresholds.py`
- [ ] T099 [P] [US5] Implement baseline drift detection and quarantine in `backend/apps/anomalies/bsil_drift.py`
- [ ] T100 [P] [US5] Implement governed review label ingestion in `backend/apps/anomalies/bsil_review_labels.py`
- [ ] T101 [US5] Persist baseline snapshots, threshold shifts, drift states, and review labels in `backend/apps/anomalies/models.py`
- [ ] T102 [US5] Add anomaly candidate and review-label endpoints in `backend/apps/anomalies/views.py`
- [ ] T103 [US5] Emit baseline drift, threshold shift, and false-positive/false-negative telemetry in `backend/apps/runtime/bsil_metrics.py`
- [ ] T104 [US5] Render anomaly candidates, baseline provenance, drift state, and review labels in `frontend/src/features/bsil/AnomalyReviewPanel.tsx`
- [ ] T105 [US5] Add baseline replay export command in `backend/apps/anomalies/management/commands/export_bsil_baseline_replay.py`

## Phase 8: User Story 6 - Validate Scientific and Operational Evidence (Priority: P3)

**Goal**: Produce durable production and scientific evidence with repeated
benchmarks, raw traces, full raw retention references, access audit, and no
fake maturity vectors.

**Independent Test**: Acceptance rejects placeholder evidence, stale
artifacts, missing raw traces, dev-only production claims, hidden xfails,
SQLite-backed evidence, unresolved reconciliation, and confidence-band
contradictions.

### Tests and Evidence for User Story 6

- [ ] T106 [P] [US6] Add evidence authenticity validation tests in `backend/tests/contract/test_bsil_evidence_authenticity.py`
- [ ] T107 [P] [US6] Add access audit tests for broad authenticated staff views in `backend/tests/integration/behavior/test_bsil_access_audit.py`
- [ ] T108 [P] [US6] Add full raw retention reference tests in `backend/tests/integration/forensics/test_bsil_retention.py`
- [ ] T109 [P] [US6] Add runtime reconciliation acceptance tests in `backend/tests/system/test_bsil_runtime_reconciliation.py`
- [ ] T110 [P] [US6] Add GPU/queue causality and raw trace tests in `backend/tests/system/test_bsil_gpu_queue_causality.py`
- [ ] T111 [P] [US6] Add long-running soak evidence validation tests in `backend/tests/system/test_bsil_soak_evidence.py`
- [ ] T112 [P] [US6] Add CI guard tests for no fake maturity vectors in `backend/tests/unit/scripts/test_verify_bsil_acceptance.py`

### Implementation for User Story 6

- [ ] T113 [P] [US6] Implement BSIL evidence manifest validator in `backend/apps/forensics/bsil_evidence_validator.py`
- [ ] T114 [P] [US6] Implement BSIL access audit writer in `backend/apps/forensics/bsil_access_audit.py`
- [ ] T115 [P] [US6] Implement full raw retention reference writer in `backend/apps/forensics/bsil_retention.py`
- [ ] T116 [P] [US6] Implement runtime reconciliation checker in `backend/apps/runtime/bsil_reconciliation.py`
- [ ] T117 [P] [US6] Implement GPU/queue causality exporter in `backend/apps/runtime/bsil_causality.py`
- [ ] T118 [P] [US6] Implement BSIL benchmark report generator in `tools/benchmarks/bsil_benchmark_report.py`
- [ ] T119 [P] [US6] Implement CI acceptance verifier in `scripts/ci/verify_bsil_acceptance.py`
- [ ] T120 [US6] Add production BSIL final evidence helper in `tools/prod/prod-bsil-final-evidence.ps1`
- [ ] T121 [US6] Add final BSIL acceptance management command in `backend/apps/behavior/management/commands/run_bsil_acceptance.py`
- [ ] T122 [US6] Store final BSIL evidence manifest schema in `ci_evidence/bsil/final_acceptance_manifest.schema.json`

## Phase 9: Polish and Cross-Cutting Closure

**Purpose**: Final alignment, documentation, dashboards, CI wiring, and
production readiness preparation across all stories.

- [ ] T123 [P] Update BSIL API documentation in `docs/backend/architecture/bsil-runtime.md`
- [ ] T124 [P] Update BSIL observability dashboard documentation in `docs/backend/architecture/observability-runbook.md`
- [ ] T125 [P] Update production topology and queue routing documentation in `docs/backend/architecture/deployment-topology.md`
- [ ] T126 [P] Update frontend BSIL review UX notes in `frontend/src/features/bsil/README.md`
- [ ] T127 [P] Add BSIL workflow to CI configuration in `.github/workflows/bsil-maturity.yml`
- [ ] T128 [P] Add BSIL production evidence paths to `.gitignore`
- [ ] T129 [P] Add BSIL evidence package README in `ci_evidence/bsil/README.md`
- [ ] T130 Run focused backend BSIL unit tests and record output in `ci_evidence/bsil/backend_unit_tests.txt`
- [ ] T131 Run BSIL contract and integration tests and record output in `ci_evidence/bsil/backend_contract_integration_tests.txt`
- [ ] T132 Run BSIL system acceptance tests and record output in `ci_evidence/bsil/backend_system_tests.txt`
- [ ] T133 Run frontend BSIL unit tests and record output in `ci_evidence/bsil/frontend_unit_tests.txt`
- [ ] T134 Run BSIL CI verifiers and record output in `ci_evidence/bsil/ci_verifier_results.txt`
- [ ] T135 Verify no hidden xfails, skipped runtime gates, placeholder artifacts, SQLite evidence paths, or unsupported fallback paths remain using `scripts/ci/verify_bsil_acceptance.py`
- [ ] T136 Produce final BSIL maturity report in `ci_evidence/bsil/bsil_maturity_report.md`

## Dependencies and Execution Order

### Phase Dependencies

- Phase 1 Setup blocks all implementation.
- Phase 2 Foundational blocks every user story.
- US1 and US2 are P1 and may proceed in parallel after Phase 2 when their shared contracts are stable.
- US3 depends on US1 and US2 because episodes require semantic states and temporal state windows.
- US4 depends on US1 and US2 because interaction edges require semantic and temporal authority.
- US5 depends on US2 and may use US3/US4 outputs as optional inputs.
- US6 depends on all behavioral output paths and closes final evidence.
- Phase 9 runs after desired user-story slices are complete.

### User Story Completion Order

1. US1 - Interpret Pose Semantics Safely
2. US2 - Reason Over Temporal Behavioral State
3. US3 - Generate Explainable Behavioral Episodes
4. US4 - Reason About Multi-Person Interaction Context
5. US5 - Apply Adaptive Anomaly Accumulation
6. US6 - Validate Scientific and Operational Evidence

## Parallel Execution Examples

### US1 Parallel Work

```text
T029 semantic pose contract tests
T030 pose lineage rejection tests
T031 confidence-band tests
T035 confidence normalization
T036 joint reliability policy
T037 attention derivation
T038 posture and motion semantics
```

### US2 Parallel Work

```text
T046 temporal window tests
T047 spike suppression tests
T052 temporal window builder
T053 state machine
T054 decay rules
T055 timestamp authority
```

### US4 Parallel Work

```text
T078 edge lifecycle tests
T079 identity gate tests
T083 edge lifecycle implementation
T084 synchronized attention edges
T085 crowd context scoring
T086 identity gates
```

### US6 Parallel Work

```text
T106 evidence authenticity tests
T107 access audit tests
T108 retention reference tests
T113 evidence manifest validator
T114 access audit writer
T115 retention reference writer
T117 GPU/queue causality exporter
```

## Implementation Strategy

### MVP First

1. Complete Phase 1 and Phase 2.
2. Complete US1 semantic pose safety.
3. Complete enough US2 temporal state to replay and suppress spikes.
4. Stop and validate semantic output, confidence bands, lineage, and temporal replay before creating episodes.

### Incremental Delivery

1. US1 delivers safe semantic pose state.
2. US2 adds temporal memory and state confidence.
3. US3 adds explainable episodes and human-reviewed escalation.
4. US4 adds interaction context.
5. US5 adds adaptive baseline governance.
6. US6 closes production/scientific evidence.

### Acceptance Rule

No story is complete until its tests, runtime reconciliation, evidence export,
telemetry, confidence-band behavior, and anti-regression gates pass for that
story's scope.
