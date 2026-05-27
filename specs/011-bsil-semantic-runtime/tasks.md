# Tasks: Behavioral Semantic Intelligence Layer

**Input**: Design documents from `specs/011-bsil-semantic-runtime/`  
**Prerequisites**: [spec.md](spec.md), [plan.md](plan.md), governance contracts in this directory

**Tests and Evidence**: All tasks that produce behavioral outputs require
lineage, confidence, uncertainty, reconciliation, observability and evidence
validation. Mocks may support unit isolation but cannot close production or
scientific maturity gates.

## Phase 1: Specification and Governance Foundation

- [ ] T001 Review BSIL spec against constitution v2.1.0 anti-regression matrix.
- [ ] T002 Define semantic output vocabulary and prohibited accusation terms.
- [ ] T003 Define behavioral confidence decomposition dimensions and thresholds.
- [ ] T004 Define immutable decision lineage identifiers and artifact names.
- [ ] T005 Define PostgreSQL-only persistence contracts for all BSIL entities.
- [ ] T006 Define live/offline queue ownership for semantic, behavioral and interaction work.
- [ ] T007 Define telemetry names, dashboard panels, SLO budgets and degradation states.
- [ ] T008 Define xfail/debt/drift registry rules for BSIL closure.

## Phase 2: Pose Semantic Decision Engine

- [ ] T009 Define semantic pose state schema and validation rules.
- [ ] T010 Define head attention, gaze category, engagement, posture fatigue, hand activity and desk interaction labels.
- [ ] T011 Define low-confidence, partial-pose and occlusion degradation behavior.
- [ ] T012 Add semantic pose lineage validation tests.
- [ ] T013 Add semantic confidence and uncertainty audit tests.
- [ ] T014 Add acceptance evidence for semantic traces on representative offline and live samples.

## Phase 3: Temporal Behavioral State Engine

- [ ] T015 Define temporal reasoning window schema and state machine semantics.
- [ ] T016 Define hysteresis, cooldown, smoothing, decay and long-duration continuity rules.
- [ ] T017 Define spike suppression and invalid-window behavior.
- [ ] T018 Add replay tests for sustained distraction, repeated scanning, engagement decay and posture instability.
- [ ] T019 Add runtime reconciliation tests for state transitions and persistence.

## Phase 4: Behavioral Episode Accumulation

- [ ] T020 Define behavioral episode lifecycle, immutable update and supersession rules.
- [ ] T021 Define causal evidence chain and supporting event requirements.
- [ ] T022 Define rollback-safe update and reconciliation behavior.
- [ ] T023 Add episode export and lineage report validation.
- [ ] T024 Add no-missing-evidence and no-accusation-language acceptance tests.

## Phase 5: Multi-Person Interaction Graph

- [ ] T025 Define interaction edge schema and confidence scoring.
- [ ] T026 Define synchronized gaze, peer attention, local motion synchronization and crowd context rules.
- [ ] T027 Define identity-confidence gates and low-confidence suppression.
- [ ] T028 Add crowded, crossing, occlusion and re-entry validation datasets.
- [ ] T029 Add interaction graph audit export validation.

## Phase 6: Adaptive Contextual Anomaly Layer

- [ ] T030 Define session-aware, person-aware and environment-aware baseline semantics.
- [ ] T031 Define anomaly score decomposition, drift visibility and threshold governance.
- [ ] T032 Define baseline freeze, invalidation and review states.
- [ ] T033 Add uncertainty, false-positive and false-negative review workflow.
- [ ] T034 Add adaptive baseline reproducibility tests.

## Phase 7: Explainability and Decision Lineage

- [ ] T035 Define `behavioral_decision_trace.json` artifact schema.
- [ ] T036 Define `episode_lineage_report.json` artifact schema.
- [ ] T037 Define `temporal_reasoning_audit.json` artifact schema.
- [ ] T038 Define `interaction_graph_audit.json` artifact schema.
- [ ] T039 Add replay/debug reconstruction validation for accepted behavioral states.
- [ ] T040 Add artifact authenticity, digest and non-placeholder validation.

## Phase 8: Scientific Validation and Acceptance

- [ ] T041 Assemble at least three offline representative classroom datasets.
- [ ] T042 Assemble at least two live RTSP representative datasets.
- [ ] T043 Add crowded interaction, occlusion/re-entry and long-duration session evidence.
- [ ] T044 Add repeated-run benchmark validation with baseline/candidate causality and variance.
- [ ] T045 Add behavioral confidence and uncertainty audit reports.
- [ ] T046 Add queue pressure, GPU attribution and runtime causality exports.
- [ ] T047 Add CI gates for lineage completeness, no fallback, no placeholders, PostgreSQL-only evidence, temporal replay and interaction consistency.
- [ ] T048 Run final acceptance gate and store immutable evidence snapshot.

## Phase 9: Closure

- [ ] T049 Verify no hidden xfails, unbounded debt, runtime drift or unsupported fallback paths remain.
- [ ] T050 Verify dashboards expose semantic runtime, temporal state, interaction graph and confidence distributions.
- [ ] T051 Verify production rollback and freeze criteria are documented and tested.
- [ ] T052 Prepare final BSIL maturity report separating runtime performance, scientific validity and operational stability.

