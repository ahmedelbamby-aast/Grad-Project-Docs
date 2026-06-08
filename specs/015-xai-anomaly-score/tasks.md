# Tasks: Explainable Behavioral Evidence And Anomaly Scoring

**Input**: `spec.md`, `plan.md`, `research.md`, `data-model.md`,
`signal-catalog.md`, `atomic-cycles.md`, and `contracts/`
**Execution rule**: Each Cycle 015.x phase is an indivisible acceptance unit.
Tasks may run in parallel where marked, but no partial cycle can claim a
production decision.

## Task Format

- `[P]` means parallelizable within the cycle without overlapping ownership.
- `[US1]` model explanations; `[US2]` anomaly/temporal diagnosis; `[US3]`
  cross-model/context; `[US4]` deep XAI/review; `[US5]` WebGL workbench; `[US6]`
  production/scientific acceptance.
- Every production decision requires stride-1 native Linux RTX 5090 evidence,
  figures, manifest/digests, ledger entry, and rollback proof.

## Phase 1: Planning And Shared Governance Setup

- [ ] T001 Create XAI/anomaly results directory and index in `docs/xai_anomaly/README.md`
- [ ] T002 [P] Add feature-owned environment contract placeholders to `.env.example`
- [ ] T003 [P] Add XAI/anomaly source path filters and focused test placeholders to `.github/workflows/xai-anomaly.yml`
- [ ] T004 [P] Add static prohibited-accusation vocabulary verifier in `scripts/ci/verify_xai_language.py`
- [ ] T005 [P] Add static signal/explainer registry coverage verifier in `scripts/ci/verify_xai_registry.py`
- [ ] T006 [P] Add static placeholder/artifact authenticity verifier in `scripts/ci/verify_xai_artifacts.py`
- [ ] T007 [P] Define benchmark evidence root layout in `docs/xai_anomaly/benchmark_evidence_layout.md`
- [ ] T008 [P] Define XAI privacy, access, and retention policy in `docs/xai_anomaly/security_retention_policy.md`
- [ ] T009 [P] Define reviewer-label dataset governance in `docs/xai_anomaly/reviewer_dataset_governance.md`
- [ ] T010 [P] Add XAI/anomaly terminology and knowledge-limit glossary in `docs/xai_anomaly/glossary.md`
- [ ] T011 Define configuration ownership and fingerprint contract in `docs/xai_anomaly/configuration_contract.md`
- [ ] T012 Define exactly-one Figure Planner and Figure Implementer kickoff template in `docs/xai_anomaly/figure_role_template.md`
- [ ] T013 Define cycle result/decision template in `docs/xai_anomaly/cycle_result_template.md`
- [ ] T014 Define cycle production preflight/rollback template in `docs/xai_anomaly/production_cycle_runbook.md`
- [ ] T015 Run documentation and source-reference validation for the setup artifacts and record it in `docs/xai_anomaly/setup_validation.md`

## Phase 2: Cycle 015.0 - Runtime Truth, Route Snapshot, And BSIL Activation

**Goal**: Establish immutable runtime truth and prove BSIL/lineage activation.
**Independent test**: One production job yields a valid route snapshot and
reconciled nonzero BSIL/lineage records without source-output regression.

- [ ] T016 [P] [US6] Write Cycle 015.0 investigation, hypothesis, streaming compatibility, gates, and rollback in `docs/xai_anomaly/cycle_015_0_investigation.md`
- [ ] T017 [P] [US6] Name Cycle 015.0 Figure Planner and define raw inputs/plots/targets in `docs/xai_anomaly/cycle_015_0_figure_plan.md`
- [ ] T018 [P] [US6] Name separate Cycle 015.0 Figure Implementer and claim generator ownership in `docs/xai_anomaly/cycle_015_0_figure_implementation.md`
- [ ] T019 [P] [US1] Add immutable route snapshot value contract in `backend/apps/pipeline/services/model_route_snapshot.py`
- [ ] T020 [US1] Remove shared-default route mutation and add snapshot/fingerprint APIs in `backend/apps/pipeline/services/model_route_service.py`
- [ ] T021 [P] [US1] Add route snapshot PostgreSQL model, migration, and explicit serializer boundary in `backend/apps/behavior/models.py` and `backend/apps/behavior/serializers.py`
- [ ] T022 [P] [US1] Audit and reconcile behavior label mappings in `backend/apps/pipeline/multi_model.py`
- [ ] T023 [P] [US1] Audit and reconcile layer-specific behavior mappings in `backend/apps/pipeline/layers/`
- [ ] T024 [P] [US2] Replace or deprecate accusatory rule-engine outputs in `backend/apps/pipeline/rule_engine.py`
- [ ] T025 [US2] Wire BSIL semantic/temporal/lineage activation flags into existing offline and live orchestration in `backend/apps/video_analysis/tasks.py`
- [ ] T026 [P] [US6] Add route immutability, mapping parity, vocabulary, and snapshot digest tests in `backend/tests/unit/pipeline/test_xai_runtime_truth.py`
- [ ] T027 [P] [US2] Add BSIL activation/population/reconciliation integration tests in `backend/tests/integration/behavior/test_xai_bsil_activation.py`
- [ ] T028 [P] [US6] Add production runtime-truth/BSIL preflight helper in `tools/prod/prod_xai_cycle015_0_preflight.py`
- [ ] T029 [US6] Add Cycle 015.0 stride-1 production benchmark and metrics collector in `tools/prod/prod_run_xai_cycle015_0.sh`
- [ ] T030 [P] [US6] Implement Cycle 015.0 figure generator and manifest tests in `tools/prod/prod_generate_xai_cycle015_0_figures.py`
- [ ] T031 [US6] Execute Cycle 015.0 rollback and reconciliation helper from `tools/prod/prod_rollback_xai_cycle015_0.sh`
- [ ] T032 [US6] Record Cycle 015.0 decision, metrics, figures, unavailable reasons, and rollback in `docs/xai_anomaly/cycle_015_0_results.md`
- [ ] T033 [US6] Record every Cycle 015.0 run and decision in `docs/BENCHMARK_RESULTS_LEDGER.md`

## Phase 3: Cycle 015.1 - Versioned Evidence Envelope And Signal Registry

**Goal**: Normalize every active signal through one governed evidence contract.
**Independent test**: Every active signal resolves to one definition and
idempotently produces a bounded envelope with explicit missingness.

- [ ] T034 [P] [US1] Write Cycle 015.1 investigation and atomic acceptance gates in `docs/xai_anomaly/cycle_015_1_investigation.md`
- [ ] T035 [P] [US6] Name Cycle 015.1 Figure Planner in `docs/xai_anomaly/cycle_015_1_figure_plan.md`
- [ ] T036 [P] [US6] Name separate Cycle 015.1 Figure Implementer in `docs/xai_anomaly/cycle_015_1_figure_implementation.md`
- [ ] T037 [P] [US1] Implement evidence/signal value contracts in `backend/apps/behavior/explainability/contracts.py`
- [ ] T038 [P] [US1] Implement signal registry and compatibility validation in `backend/apps/behavior/explainability/registry.py`
- [ ] T039 [P] [US1] Implement common source/time/identity/quality normalization in `backend/apps/behavior/explainability/signals.py`
- [ ] T040 [US1] Register all current raw and derived signal definitions in `backend/apps/behavior/explainability/signal_definitions.py`
- [ ] T041 [US1] Add idempotent evidence-envelope repository boundary in `backend/apps/behavior/repositories.py`
- [ ] T042 [US1] Add signal-definition and evidence-envelope PostgreSQL models and migration in `backend/apps/behavior/models.py`
- [ ] T043 [P] [US1] Add explicit evidence serializers and bounded API fields in `backend/apps/behavior/serializers.py`
- [ ] T044 [P] [US1] Add registry, missingness, payload-size, and compatibility unit tests in `backend/tests/unit/behavior/test_xai_evidence_registry.py`
- [ ] T045 [P] [US1] Add envelope idempotency, PostgreSQL, and replay integration tests in `backend/tests/integration/behavior/test_xai_evidence_persistence.py`
- [ ] T046 [P] [US6] Add signal registry coverage production probe in `tools/prod/prod_probe_xai_signal_registry.py`
- [ ] T047 [US6] Add Cycle 015.1 stride-1 benchmark/collector in `tools/prod/prod_run_xai_cycle015_1.sh`
- [ ] T048 [P] [US6] Implement Cycle 015.1 figures, manifest/digests, and generator tests in `tools/prod/prod_generate_xai_cycle015_1_figures.py`
- [ ] T049 [US6] Execute and document Cycle 015.1 rollback in `docs/xai_anomaly/cycle_015_1_results.md`
- [ ] T050 [US6] Record every Cycle 015.1 run and decision in `docs/BENCHMARK_RESULTS_LEDGER.md`

## Phase 4: Cycle 015.2 - Per-Model Calibration And Reliability

**Goal**: Produce compatible, measured calibration snapshots for active model
outputs.
**Independent test**: Calibration improves or accurately characterizes
reliability and incompatible/stale snapshots are rejected.

- [ ] T051 [P] [US1] Write Cycle 015.2 investigation and calibration protocol in `docs/xai_anomaly/cycle_015_2_investigation.md`
- [ ] T052 [P] [US6] Name Cycle 015.2 Figure Planner in `docs/xai_anomaly/cycle_015_2_figure_plan.md`
- [ ] T053 [P] [US6] Name separate Cycle 015.2 Figure Implementer in `docs/xai_anomaly/cycle_015_2_figure_implementation.md`
- [ ] T054 [P] [US1] Implement calibration contracts and compatibility rules in `backend/apps/anomalies/scoring/calibration.py`
- [ ] T055 [P] [US1] Add calibration snapshot models and migration in `backend/apps/anomalies/models.py`
- [ ] T056 [P] [US1] Implement calibration artifact/dataset manifest handling in `backend/apps/anomalies/scoring/calibration_artifacts.py`
- [ ] T057 [US1] Implement model/output-specific calibrator registry in `backend/apps/anomalies/scoring/calibrator_registry.py`
- [ ] T058 [US1] Implement calibration lookup and stale/incompatible rejection in `backend/apps/anomalies/scoring/calibration_service.py`
- [ ] T059 [P] [US1] Add calibration fitting/evaluation command in `backend/apps/anomalies/management/commands/build_xai_calibration.py`
- [ ] T060 [P] [US1] Add ECE, reliability, proper-score, CI, and subgroup evaluation in `backend/apps/anomalies/scoring/calibration_evaluation.py`
- [ ] T061 [P] [US1] Add calibration contract and compatibility tests in `backend/tests/unit/anomalies/test_xai_calibration.py`
- [ ] T062 [P] [US1] Add calibration dataset/manifest and PostgreSQL tests in `backend/tests/integration/anomalies/test_xai_calibration_artifacts.py`
- [ ] T063 [P] [US6] Add production calibration preflight/probe in `tools/prod/prod_probe_xai_calibration.py`
- [ ] T064 [US6] Add Cycle 015.2 stride-1 benchmark/collector in `tools/prod/prod_run_xai_cycle015_2.sh`
- [ ] T065 [P] [US6] Implement Cycle 015.2 reliability figures, manifest/digests, and generator tests in `tools/prod/prod_generate_xai_cycle015_2_figures.py`
- [ ] T066 [US6] Execute and document Cycle 015.2 rollback in `docs/xai_anomaly/cycle_015_2_results.md`
- [ ] T067 [US6] Record every Cycle 015.2 run and decision in `docs/BENCHMARK_RESULTS_LEDGER.md`

## Phase 5: Cycle 015.3 - Deterministic Fast Explainers For Every Active Model

**Goal**: Cover every active production route and deterministic derived layer
with a compatible fast explainer.
**Independent test**: Every ready route returns a deterministic explanation or
explicit compatibility rejection.

- [ ] T068 [P] [US1] Write Cycle 015.3 investigation and adapter coverage matrix in `docs/xai_anomaly/cycle_015_3_investigation.md`
- [ ] T069 [P] [US6] Name Cycle 015.3 Figure Planner in `docs/xai_anomaly/cycle_015_3_figure_plan.md`
- [ ] T070 [P] [US6] Name separate Cycle 015.3 Figure Implementer in `docs/xai_anomaly/cycle_015_3_figure_implementation.md`
- [ ] T071 [P] [US1] Implement explainer interface and registry in `backend/apps/behavior/explainability/registry.py`
- [ ] T072 [P] [US1] Implement detector fast explainer in `backend/apps/behavior/explainability/adapters/person_detector.py`
- [ ] T073 [P] [US1] Implement posture/gaze/ensemble fast explainers in `backend/apps/behavior/explainability/adapters/behavior_models.py`
- [ ] T074 [P] [US1] Implement RTMPose and pose-kinematics fast explainers in `backend/apps/behavior/explainability/adapters/pose.py`
- [ ] T075 [P] [US1] Implement OSNet/tracking fast explainer in `backend/apps/behavior/explainability/adapters/reid_tracking.py`
- [ ] T076 [P] [US1] Implement YOLOE scene fast explainer in `backend/apps/behavior/explainability/adapters/yoloe_scene.py`
- [ ] T077 [P] [US1] Implement SRVL deterministic explainer in `backend/apps/behavior/explainability/adapters/srvl.py`
- [ ] T078 [P] [US1] Implement BSIL/rule deterministic explainer in `backend/apps/behavior/explainability/adapters/bsil_rules.py`
- [ ] T079 [US1] Implement common explanation fragment composer in `backend/apps/behavior/explainability/composer.py`
- [ ] T080 [US1] Replace or retire placeholder explanation and visualization writers in `backend/apps/pipeline/model_lifecycle/explainability.py` and `backend/apps/pipeline/model_lifecycle/visualizations.py`
- [ ] T081 [P] [US1] Add per-adapter deterministic, quality, contradiction, and counterfactual tests in `backend/tests/unit/behavior/test_xai_fast_explainers.py`
- [ ] T082 [P] [US1] Add active-route coverage and output-parity integration tests in `backend/tests/integration/behavior/test_xai_fast_explainer_coverage.py`
- [ ] T083 [P] [US6] Add production adapter coverage/latency probe in `tools/prod/prod_probe_xai_fast_explainers.py`
- [ ] T084 [US6] Add Cycle 015.3 stride-1 benchmark/collector in `tools/prod/prod_run_xai_cycle015_3.sh`
- [ ] T085 [P] [US6] Implement Cycle 015.3 figures, manifest/digests, and generator tests in `tools/prod/prod_generate_xai_cycle015_3_figures.py`
- [ ] T086 [US6] Execute rollback and record Cycle 015.3 decision in `docs/xai_anomaly/cycle_015_3_results.md`
- [ ] T087 [US6] Record every Cycle 015.3 run and decision in `docs/BENCHMARK_RESULTS_LEDGER.md`

## Phase 6: Cycle 015.4 - Bounded Temporal Evidence And Episode Explanations

**Goal**: Explain temporal patterns and episodes with bounded state and invalid
gap handling.
**Independent test**: Replay and indefinite-stream tests reproduce transitions,
suppress spikes, and keep memory bounded.

- [ ] T088 [P] [US2] Write Cycle 015.4 investigation and bounded-state contract in `docs/xai_anomaly/cycle_015_4_investigation.md`
- [ ] T089 [P] [US6] Name Cycle 015.4 Figure Planner in `docs/xai_anomaly/cycle_015_4_figure_plan.md`
- [ ] T090 [P] [US6] Name separate Cycle 015.4 Figure Implementer in `docs/xai_anomaly/cycle_015_4_figure_implementation.md`
- [ ] T091 [P] [US2] Implement bounded temporal evidence windows in `backend/apps/behavior/explainability/temporal_windows.py`
- [ ] T092 [P] [US2] Implement temporal pattern explanation in `backend/apps/behavior/explainability/temporal_explanations.py`
- [ ] T093 [P] [US2] Implement episode contribution/counterfactual composition in `backend/apps/behavior/explainability/episode_explanations.py`
- [ ] T094 [US2] Integrate temporal evidence with existing behavior temporal services in `backend/apps/behavior/temporal/windows.py`
- [ ] T095 [US2] Add bounded cache/eviction/reconnect behavior in `backend/apps/behavior/temporal/cache.py`
- [ ] T096 [P] [US2] Add spike, sustained, drift, gap, and deterministic replay tests in `backend/tests/unit/behavior/test_xai_temporal_explanations.py`
- [ ] T097 [P] [US2] Add indefinite-stream bounded-state and restart integration tests in `backend/tests/integration/behavior/test_xai_temporal_bounded_state.py`
- [ ] T098 [P] [US6] Add temporal state/soak production probe in `tools/prod/prod_probe_xai_temporal_state.py`
- [ ] T099 [US6] Add Cycle 015.4 stride-1 benchmark/collector in `tools/prod/prod_run_xai_cycle015_4.sh`
- [ ] T100 [P] [US6] Implement Cycle 015.4 temporal/soak figures, manifest/digests, and generator tests in `tools/prod/prod_generate_xai_cycle015_4_figures.py`
- [ ] T101 [P] [US6] Add live bounded-state soak helper in `tools/prod/prod_soak_xai_temporal_live.sh`
- [ ] T102 [US6] Execute rollback and reconciliation from `tools/prod/prod_rollback_xai_cycle015_4.sh`
- [ ] T103 [US6] Record Cycle 015.4 decision and figures in `docs/xai_anomaly/cycle_015_4_results.md`
- [ ] T104 [US6] Record every Cycle 015.4 run and decision in `docs/BENCHMARK_RESULTS_LEDGER.md`

## Phase 7: Cycle 015.5 - Transparent Hierarchical Anomaly Score

**Goal**: Produce a reconstructable non-accusatory review-priority score.
**Independent test**: Persisted contributions exactly reconstruct scores and
invalid evidence reliably withholds them.

- [ ] T105 [P] [US2] Write Cycle 015.5 investigation, score profile, and labeled evaluation protocol in `docs/xai_anomaly/cycle_015_5_investigation.md`
- [ ] T106 [P] [US6] Name Cycle 015.5 Figure Planner in `docs/xai_anomaly/cycle_015_5_figure_plan.md`
- [ ] T107 [P] [US6] Name separate Cycle 015.5 Figure Implementer in `docs/xai_anomaly/cycle_015_5_figure_implementation.md`
- [ ] T108 [P] [US2] Implement anomaly score/contribution contracts in `backend/apps/anomalies/scoring/contracts.py`
- [ ] T109 [P] [US2] Implement calibrated surprise and contribution components in `backend/apps/anomalies/scoring/components.py`
- [ ] T110 [P] [US2] Implement transparent score fusion and withholding gates in `backend/apps/anomalies/scoring/fusion.py`
- [ ] T111 [US2] Implement anomaly scoring orchestration service in `backend/apps/anomalies/scoring/service.py`
- [ ] T112 [US2] Add score/contribution models and migration in `backend/apps/anomalies/models.py`
- [ ] T113 [US2] Add idempotent score/contribution repositories in `backend/apps/anomalies/services.py`
- [ ] T114 [P] [US2] Add exact reconstruction and counterfactual service in `backend/apps/anomalies/scoring/reconstruction.py`
- [ ] T115 [P] [US2] Add explicit score serializers and read endpoints in `backend/apps/anomalies/serializers.py` and `backend/apps/anomalies/views.py`
- [ ] T116 [P] [US2] Add contribution math, missingness, withholding, and vocabulary tests in `backend/tests/unit/anomalies/test_xai_review_priority_score.py`
- [ ] T117 [P] [US2] Add PostgreSQL reconstruction/idempotency/integration tests in `backend/tests/integration/anomalies/test_xai_score_persistence.py`
- [ ] T118 [P] [US2] Add labeled ranking/error evaluation command in `backend/apps/anomalies/management/commands/evaluate_xai_scores.py`
- [ ] T119 [P] [US6] Add production score reconstruction/probe helper in `tools/prod/prod_probe_xai_scores.py`
- [ ] T120 [US6] Add Cycle 015.5 stride-1 benchmark/collector in `tools/prod/prod_run_xai_cycle015_5.sh`
- [ ] T121 [P] [US6] Implement Cycle 015.5 score/error figures, manifest/digests, and generator tests in `tools/prod/prod_generate_xai_cycle015_5_figures.py`
- [ ] T122 [US6] Execute rollback and record Cycle 015.5 decision in `docs/xai_anomaly/cycle_015_5_results.md`
- [ ] T123 [US6] Record every Cycle 015.5 run and decision in `docs/BENCHMARK_RESULTS_LEDGER.md`

## Phase 8: Cycle 015.6 - Missingness, Uncertainty, And Conformal Calibration

**Goal**: Add explicit uncertainty and valid conformal outputs without false
precision.
**Independent test**: Coverage is measured and outputs are withheld when
assumptions or drift gates fail.

- [ ] T124 [P] [US2] Write Cycle 015.6 investigation and conformal assumption protocol in `docs/xai_anomaly/cycle_015_6_investigation.md`
- [ ] T125 [P] [US6] Name Cycle 015.6 Figure Planner in `docs/xai_anomaly/cycle_015_6_figure_plan.md`
- [ ] T126 [P] [US6] Name separate Cycle 015.6 Figure Implementer in `docs/xai_anomaly/cycle_015_6_figure_implementation.md`
- [ ] T127 [P] [US2] Implement uncertainty/missingness propagation in `backend/apps/anomalies/scoring/uncertainty.py`
- [ ] T128 [P] [US2] Implement conformal snapshot/contracts and assumption gates in `backend/apps/anomalies/scoring/conformal.py`
- [ ] T129 [US2] Add conformal snapshot model and migration in `backend/apps/anomalies/models.py`
- [ ] T130 [P] [US2] Implement coverage/drift evaluation in `backend/apps/anomalies/scoring/conformal_evaluation.py`
- [ ] T131 [US2] Integrate uncertainty/conformal outputs into score service in `backend/apps/anomalies/scoring/service.py`
- [ ] T132 [P] [US2] Add missingness, coverage, drift, and withholding unit tests in `backend/tests/unit/anomalies/test_xai_conformal_uncertainty.py`
- [ ] T133 [P] [US2] Add adaptive bounded-state and PostgreSQL integration tests in `backend/tests/integration/anomalies/test_xai_conformal_runtime.py`
- [ ] T134 [P] [US6] Add production conformal coverage/drift probe in `tools/prod/prod_probe_xai_conformal.py`
- [ ] T135 [US6] Add Cycle 015.6 stride-1 benchmark/collector in `tools/prod/prod_run_xai_cycle015_6.sh`
- [ ] T136 [P] [US6] Implement Cycle 015.6 coverage figures, manifest/digests, and generator tests in `tools/prod/prod_generate_xai_cycle015_6_figures.py`
- [ ] T137 [US6] Execute rollback from `tools/prod/prod_rollback_xai_cycle015_6.sh`
- [ ] T138 [US6] Record Cycle 015.6 decision in `docs/xai_anomaly/cycle_015_6_results.md`
- [ ] T139 [US6] Record every Cycle 015.6 run and decision in `docs/BENCHMARK_RESULTS_LEDGER.md`

## Phase 9: Cycle 015.7 - Cross-Model Fusion And Explanation Graph

**Goal**: Fuse evidence while preserving sources, contradictions, identity
limits, and graph lineage.
**Independent test**: Crowded/crossing/contradiction sequences keep unresolved
identity unresolved and reconstruct the explanation graph.

- [ ] T140 [P] [US3] Write Cycle 015.7 investigation and identity-gated fusion protocol in `docs/xai_anomaly/cycle_015_7_investigation.md`
- [ ] T141 [P] [US6] Name Cycle 015.7 Figure Planner in `docs/xai_anomaly/cycle_015_7_figure_plan.md`
- [ ] T142 [P] [US6] Name separate Cycle 015.7 Figure Implementer in `docs/xai_anomaly/cycle_015_7_figure_implementation.md`
- [ ] T143 [P] [US3] Implement source-preserving fusion policy in `backend/apps/anomalies/scoring/fusion.py`
- [ ] T144 [P] [US3] Implement contradiction policy and visibility contract in `backend/apps/behavior/explainability/contradictions.py`
- [ ] T145 [P] [US3] Implement identity-gated scene/SRVL/peer context in `backend/apps/behavior/explainability/context.py`
- [ ] T146 [P] [US3] Implement explanation graph contracts and builder in `backend/apps/behavior/explainability/graph.py`
- [ ] T147 [US3] Implement composed explanation records and persistence in `backend/apps/behavior/explainability/composer.py`
- [ ] T148 [US3] Add explanation record models and migration in `backend/apps/behavior/models.py`
- [ ] T149 [P] [US3] Add explanation graph serializers/endpoints in `backend/apps/behavior/serializers.py`
- [ ] T150 [P] [US3] Add fusion, contradiction, identity, and graph reconstruction tests in `backend/tests/unit/behavior/test_xai_explanation_graph.py`
- [ ] T151 [P] [US3] Add crossing/fragmentation/unresolved integration tests in `backend/tests/integration/behavior/test_xai_context_identity.py`
- [ ] T152 [P] [US6] Add production fusion/identity/graph probe in `tools/prod/prod_probe_xai_fusion.py`
- [ ] T153 [US6] Add Cycle 015.7 stride-1 benchmark/collector in `tools/prod/prod_run_xai_cycle015_7.sh`
- [ ] T154 [P] [US6] Implement Cycle 015.7 graph/identity figures, manifest/digests, and generator tests in `tools/prod/prod_generate_xai_cycle015_7_figures.py`
- [ ] T155 [US6] Execute rollback from `tools/prod/prod_rollback_xai_cycle015_7.sh`
- [ ] T156 [US6] Record Cycle 015.7 decision in `docs/xai_anomaly/cycle_015_7_results.md`
- [ ] T157 [US6] Record every Cycle 015.7 run and decision in `docs/BENCHMARK_RESULTS_LEDGER.md`

## Phase 10: Cycle 015.8 - Offline And On-Demand Deep Vision XAI

**Goal**: Provide isolated, faithful, evaluated deep-XAI artifacts.
**Independent test**: Eligible methods complete idempotently with fidelity,
sanity, stability, security, and critical-path isolation evidence.

- [ ] T158 [P] [US4] Write Cycle 015.8 investigation and per-model method eligibility matrix in `docs/xai_anomaly/cycle_015_8_investigation.md`
- [ ] T159 [P] [US6] Name Cycle 015.8 Figure Planner in `docs/xai_anomaly/cycle_015_8_figure_plan.md`
- [ ] T160 [P] [US6] Name separate Cycle 015.8 Figure Implementer in `docs/xai_anomaly/cycle_015_8_figure_implementation.md`
- [ ] T161 [P] [US4] Implement deep-XAI request/artifact/evaluation contracts in `backend/apps/behavior/explainability/deep_contracts.py`
- [ ] T162 [US4] Add deep-XAI request/artifact/evaluation models and migration in `backend/apps/behavior/models.py`
- [ ] T163 [P] [US4] Implement D-RISE adapter in `backend/apps/behavior/explainability/adapters/deep_drise.py`
- [ ] T164 [P] [US4] Implement CAM/IG eligibility adapters in `backend/apps/behavior/explainability/adapters/deep_gradient.py`
- [ ] T165 [P] [US4] Implement pose joint-occlusion adapter in `backend/apps/behavior/explainability/adapters/deep_pose.py`
- [ ] T166 [P] [US4] Implement ReID prototype/exemplar adapter in `backend/apps/behavior/explainability/adapters/deep_reid.py`
- [ ] T167 [P] [US4] Implement fidelity/stability/sanity evaluation in `backend/apps/behavior/explainability/evaluation.py`
- [ ] T168 [US4] Implement bounded idempotent deep-XAI Celery task, per-stage outcome accounting, fail-closed threshold, deadline, and reconciler in `backend/apps/behavior/tasks.py`
- [ ] T169 [US4] Add deep-XAI queue routing/isolation and Beat-scheduled reconciler in `backend/apps/pipeline/routing.py` and `backend/config/celery.py`
- [ ] T170 [US4] Add authenticated artifact/request endpoints in `backend/apps/behavior/views.py`
- [ ] T171 [P] [US4] Add method eligibility, fidelity, sanity, deadline, idempotency, and payload tests in `backend/tests/unit/behavior/test_xai_deep_methods.py`
- [ ] T172 [P] [US4] Add queue isolation, artifact access, and reconciler integration tests in `backend/tests/integration/behavior/test_xai_deep_runtime.py`
- [ ] T173 [P] [US6] Add production deep-XAI isolation/fidelity probe in `tools/prod/prod_probe_xai_deep.py`
- [ ] T174 [US6] Add Cycle 015.8 stride-1 benchmark/collector in `tools/prod/prod_run_xai_cycle015_8.sh`
- [ ] T175 [P] [US6] Implement Cycle 015.8 attribution/fidelity figures, manifest/digests, and generator tests in `tools/prod/prod_generate_xai_cycle015_8_figures.py`
- [ ] T176 [US6] Execute rollback and record Cycle 015.8 decision in `docs/xai_anomaly/cycle_015_8_results.md`
- [ ] T177 [US6] Record every Cycle 015.8 run and decision in `docs/BENCHMARK_RESULTS_LEDGER.md`

## Phase 11: Cycle 015.9 - Prototype, Case Comparison, And Governed Review Feedback

**Goal**: Add a safe human-review diagnosis loop without direct production
self-modification.
**Independent test**: Feedback is immutable evaluation evidence and prototype
review improves measured reviewer outcomes without mutating runtime policy.

- [ ] T178 [P] [US4] Write Cycle 015.9 investigation and reviewer-study protocol in `docs/xai_anomaly/cycle_015_9_investigation.md`
- [ ] T179 [P] [US6] Name Cycle 015.9 Figure Planner in `docs/xai_anomaly/cycle_015_9_figure_plan.md`
- [ ] T180 [P] [US6] Name separate Cycle 015.9 Figure Implementer in `docs/xai_anomaly/cycle_015_9_figure_implementation.md`
- [ ] T181 [P] [US4] Implement governed prototype/exemplar contracts in `backend/apps/behavior/explainability/prototypes.py`
- [ ] T182 [P] [US4] Implement similar-case comparison service in `backend/apps/behavior/explainability/case_comparison.py`
- [ ] T183 [US4] Add review-feedback model/migration and no-direct-mutation guard in `backend/apps/anomalies/models.py`
- [ ] T184 [US4] Implement review feedback/audit service in `backend/apps/anomalies/scoring/review_feedback.py`
- [ ] T185 [P] [US4] Add prototype/case/review endpoints in `backend/apps/anomalies/views.py`
- [ ] T186 [P] [US4] Add feedback immutability, access, and no-direct-mutation tests in `backend/tests/unit/anomalies/test_xai_review_feedback.py`
- [ ] T187 [P] [US4] Add reviewer workflow/audit integration tests in `backend/tests/integration/anomalies/test_xai_review_workflow.py`
- [ ] T188 [P] [US6] Add reviewer-study evidence collector in `tools/prod/prod_collect_xai_reviewer_study.py`
- [ ] T189 [US6] Add Cycle 015.9 production benchmark/collector in `tools/prod/prod_run_xai_cycle015_9.sh`
- [ ] T190 [P] [US6] Implement Cycle 015.9 reviewer/prototype figures, manifest/digests, and generator tests in `tools/prod/prod_generate_xai_cycle015_9_figures.py`
- [ ] T191 [US6] Execute rollback and record Cycle 015.9 decision in `docs/xai_anomaly/cycle_015_9_results.md`
- [ ] T192 [US6] Record every Cycle 015.9 run and decision in `docs/BENCHMARK_RESULTS_LEDGER.md`

## Phase 12: Cycle 015.10 - Shared WebGL2 Explanation Workbench

**Goal**: Migrate all analytical rendering to one performant shared WebGL2
core.
**Independent test**: Representative and stress views remain interactive,
recover context loss, and never fall back to a mislabeled Canvas2D figure.

- [ ] T193 [P] [US5] Write Cycle 015.10 investigation and renderer benchmark matrix in `docs/xai_anomaly/cycle_015_10_investigation.md`
- [ ] T194 [P] [US6] Name Cycle 015.10 Figure Planner in `docs/xai_anomaly/cycle_015_10_figure_plan.md`
- [ ] T195 [P] [US6] Name separate Cycle 015.10 Figure Implementer in `docs/xai_anomaly/cycle_015_10_figure_implementation.md`
- [ ] T196 [P] [US5] Implement shared renderer host/context budget in `frontend/src/services/webgl/RendererHost.ts`
- [ ] T197 [P] [US5] Implement context budget/loss recovery in `frontend/src/services/webgl/ContextBudgetManager.ts`
- [ ] T198 [P] [US5] Implement buffer pool and typed series store in `frontend/src/services/webgl/BufferPool.ts` and `frontend/src/services/webgl/TypedSeriesStore.ts`
- [ ] T199 [P] [US5] Implement tiled matrix/LOD store in `frontend/src/services/webgl/MatrixTileStore.ts`
- [ ] T200 [P] [US5] Implement Worker viewport aggregation/downsampling in `frontend/src/services/webgl/ViewportAggregator.worker.ts`
- [ ] T201 [P] [US5] Implement stable identity/behavior color registry in `frontend/src/services/webgl/ColorRegistry.ts`
- [ ] T202 [P] [US5] Implement shared interaction and figure export controllers in `frontend/src/services/webgl/InteractionController.ts` and `frontend/src/services/webgl/FigureExporter.ts`
- [ ] T203 [US5] Migrate time-series plot to shared core in `frontend/src/components/telemetry/WebGLTimeseriesPlot.tsx`
- [ ] T204 [US5] Migrate matrix heatmap and metric charts to shared core in `frontend/src/components/telemetry/WebGLMatrixHeatmap.tsx` and `frontend/src/components/telemetry/WebGLMetricChart.tsx`
- [ ] T205 [US5] Migrate telemetry lanes from Canvas2D in `frontend/src/components/telemetry/TelemetryCanvas.tsx`
- [ ] T206 [US5] Migrate scene map from Canvas2D fallback path in `frontend/src/components/scene/SceneMapRenderer.tsx`
- [ ] T207 [P] [US5] Implement explanation contribution/graph/attribution WebGL views in `frontend/src/features/xai/`
- [ ] T208 [P] [US5] Implement binary/chunked XAI API/WS backend and frontend data clients in `backend/apps/behavior/views.py`, `backend/apps/behavior/consumers.py`, and `frontend/src/services/xaiApi.ts`
- [ ] T209 [P] [US5] Add renderer telemetry client in `frontend/src/services/webgl/rendererTelemetry.ts`
- [ ] T210 [P] [US5] Add WebGL context, interaction, large-series/matrix, and accessibility unit tests in `frontend/src/services/webgl/__tests__/`
- [ ] T211 [P] [US5] Add Playwright stress/context-loss/download tests in `frontend/tests/e2e/xai-webgl.spec.ts`
- [ ] T212 [P] [US6] Add production WebGL benchmark collector in `tools/prod/prod_benchmark_xai_webgl.mjs`
- [ ] T213 [US6] Add Cycle 015.10 production benchmark runner in `tools/prod/prod_run_xai_cycle015_10.sh`
- [ ] T214 [P] [US6] Implement Cycle 015.10 renderer figures, manifest/digests, and generator tests in `tools/prod/prod_generate_xai_cycle015_10_figures.py`
- [ ] T215 [US6] Execute rollback, record Cycle 015.10 decision, and add ledger row in `docs/xai_anomaly/cycle_015_10_results.md` and `docs/BENCHMARK_RESULTS_LEDGER.md`

## Phase 13: Cycle 015.11 - Observability, Performance, Stability, Security, And Fairness

**Goal**: Prove the integrated capability is measurable, stable, secure, and
scientifically auditable.
**Independent test**: Cross-cutting acceptance suite exposes every required
metric or a reason and fails on any critical veto.

- [ ] T216 [P] [US6] Write Cycle 015.11 investigation and integrated acceptance matrix in `docs/xai_anomaly/cycle_015_11_investigation.md`
- [ ] T217 [P] [US6] Name Cycle 015.11 Figure Planner in `docs/xai_anomaly/cycle_015_11_figure_plan.md`
- [ ] T218 [P] [US6] Name separate Cycle 015.11 Figure Implementer in `docs/xai_anomaly/cycle_015_11_figure_implementation.md`
- [ ] T219 [P] [US6] Add XAI/anomaly/renderer telemetry contracts in `backend/apps/telemetry/xai_contracts.py`
- [ ] T220 [P] [US6] Add runtime reconciliation checker in `backend/apps/behavior/explainability/reconciliation.py`
- [ ] T221 [P] [US6] Add evidence/manifest authenticity validator in `backend/apps/behavior/explainability/evidence_validator.py`
- [ ] T222 [P] [US6] Add access/privacy/retention audit verifier in `backend/apps/behavior/explainability/security_audit.py`
- [ ] T223 [P] [US6] Add calibration/subgroup/explanation stability evaluator in `backend/apps/anomalies/scoring/integrated_evaluation.py`
- [ ] T224 [P] [US6] Add integrated load/soak/failure/restart tests in `backend/tests/system/test_xai_integrated_stability.py`
- [ ] T225 [P] [US6] Add security/access/retention system tests in `backend/tests/system/test_xai_security_retention.py`
- [ ] T226 [P] [US6] Add frontend integrated workbench/reconciliation tests in `frontend/tests/e2e/xai-integrated.spec.ts`
- [ ] T227 [P] [US6] Add production instrumentation/preflight checker in `tools/prod/prod_xai_integrated_preflight.py`
- [ ] T228 [P] [US6] Add production soak/failure injection runner in `tools/prod/prod_soak_xai_integrated.sh`
- [ ] T229 [US6] Add Cycle 015.11 stride-1 production benchmark/collector in `tools/prod/prod_run_xai_cycle015_11.sh`
- [ ] T230 [P] [US6] Implement Cycle 015.11 integrated figures, manifest/digests, and generator tests in `tools/prod/prod_generate_xai_cycle015_11_figures.py`
- [ ] T231 [US6] Execute integrated rollback from `tools/prod/prod_rollback_xai_integrated.sh`
- [ ] T232 [US6] Record Cycle 015.11 decision and remaining debt in `docs/xai_anomaly/cycle_015_11_results.md`
- [ ] T233 [US6] Record every Cycle 015.11 run and decision in `docs/BENCHMARK_RESULTS_LEDGER.md`

## Phase 14: Cycle 015.12 - Production Canary, Rollback, And Final Acceptance

**Goal**: Execute one governed canary/promotion/rollback decision.
**Independent test**: Canary stop/promote criteria, runtime truth, human-review
semantics, and rollback all reconcile from immutable evidence.

- [ ] T234 [P] [US6] Write Cycle 015.12 canary plan, exposure, stop, promotion, and rollback gates in `docs/xai_anomaly/cycle_015_12_investigation.md`
- [ ] T235 [P] [US6] Name Cycle 015.12 Figure Planner in `docs/xai_anomaly/cycle_015_12_figure_plan.md`
- [ ] T236 [P] [US6] Name separate Cycle 015.12 Figure Implementer in `docs/xai_anomaly/cycle_015_12_figure_implementation.md`
- [ ] T237 [P] [US6] Add disabled/shadow/reviewer-visible/promoted profile helper in `tools/prod/prod_set_xai_profile.sh`
- [ ] T238 [P] [US6] Add canary preflight and branch/SHA/env/route/calibration fingerprint helper in `tools/prod/prod_xai_canary_preflight.py`
- [ ] T239 [P] [US6] Add canary stop/promote policy evaluator in `backend/apps/behavior/explainability/canary_policy.py`
- [ ] T240 [P] [US6] Add canary runtime reconciliation and final manifest validator in `backend/apps/behavior/explainability/final_acceptance.py`
- [ ] T241 [P] [US6] Add canary policy, stop, promotion, and rollback tests in `backend/tests/system/test_xai_canary.py`
- [ ] T242 [P] [US6] Add canary reviewer-surface Playwright tests in `frontend/tests/e2e/xai-canary.spec.ts`
- [ ] T243 [US6] Execute disabled-profile production baseline with `tools/prod/prod_run_xai_cycle015_12.sh`
- [ ] T244 [US6] Execute shadow-profile production canary with `tools/prod/prod_run_xai_cycle015_12.sh`
- [ ] T245 [US6] Execute reviewer-visible production canary with `tools/prod/prod_run_xai_cycle015_12.sh`
- [ ] T246 [P] [US6] Generate Cycle 015.12 canary/final figures, manifest/digests, and generator tests in `tools/prod/prod_generate_xai_cycle015_12_figures.py`
- [ ] T247 [US6] Execute and prove full rollback with `tools/prod/prod_rollback_xai_integrated.sh`
- [ ] T248 [US6] Package immutable final acceptance evidence in `ci_evidence/production/xai_anomaly/final/`
- [ ] T249 [US6] Record final decision, limitations, research backlog, and rollback in `docs/xai_anomaly/cycle_015_12_results.md`
- [ ] T250 [US6] Record every Cycle 015.12 run and final decision in `docs/BENCHMARK_RESULTS_LEDGER.md`

## Dependency Graph

```text
Setup
  -> 015.0
  -> 015.1
  -> 015.2
  -> 015.3
  -> 015.4
  -> 015.5
  -> 015.6
  -> 015.7

015.3 + 015.7 -> 015.8 -> 015.9
015.3 + 015.5 + 015.7 -> 015.10
015.0 through 015.10 -> 015.11 -> 015.12
```

## Parallel Execution Guidance

- Within each cycle, investigation, Figure Planner work, test design, and
  non-overlapping adapter files may proceed in parallel.
- Figure Implementer work starts only after the Figure Planner defines inputs
  and targets.
- Backend and frontend work may proceed in parallel when their versioned
  contract is frozen.
- Production benchmarks, decisions, and rollback are sequential.
- No later cycle may use an earlier cycle as accepted authority until the
  earlier cycle has a completed production decision.

## Suggested First Implementation Slice

Cycle 015.0 is the only valid first slice. Production currently has zero rows in
the BSIL/anomaly/lineage tables captured by the inventory, and the active
runtime route differs from documented accepted variants. Building anomaly
scores or deep XAI before resolving that truth would create unreconstructable
evidence.
