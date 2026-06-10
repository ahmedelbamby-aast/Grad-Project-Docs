# Tasks: Explainable Behavioral Evidence And Anomaly Scoring

**Input**: `spec.md`, `plan.md`, `research.md`, `data-model.md`,
`signal-catalog.md`, `no-ground-truth-doctrine.md`, `atomic-cycles.md`, and
`contracts/`
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
- No task may introduce anomaly-model training/fine-tuning, manufacture
  behavioral ground truth, or use label-based anomaly accuracy as decision
  authority.
- Highest-priority performance rule: Cycle 015 work is blocked from additive
  critical-path acceptance until the canonical `combined.mp4` stride-1
  production benchmark reaches `>=15 FPS` DB-completed end-to-end throughput
  and a 32-frame full authoritative cycle completes in `<=2` seconds. This
  target includes inference, postprocess, PostgreSQL writes, embeddings/derived
  records, telemetry/artifacts, reconciliation, and terminal lifecycle state.

## Urgent Phase 0A: Throughput Bottleneck Remediation Gate

**Goal**: Make bottleneck remediation the highest implementation priority before
downstream XAI/anomaly feature acceptance.
**Independent test**: A native Linux RTX 5090 stride-1 production benchmark on
`combined.mp4` proves `>=15 FPS` DB-completed end-to-end throughput and the
32-frame full-cycle envelope completes in `<=2` seconds, with source-output
correctness, identity, lineage, rollback, and figure evidence intact.

- [X] T000a [US6] Keep
  `docs/xai_anomaly/cycle_015_0_current_run_bottleneck_report.md` current with
  final Cycle 015.0 baseline metrics, unavailable reasons, and root-cause
  classification from production evidence.
- [X] T000b [US6] Add a highest-priority bottleneck-remediation investigation in
  `docs/xai_anomaly/cycle_015_throughput_remediation_investigation.md` with
  causal hypotheses for postprocess wall, inference wall, pose-tail wall,
  telemetry visibility, and lifecycle/status convergence.
- [X] T000c [US6] Add a throughput-target benchmark/collector contract that
  reports DB-completed FPS, 32-frame full-cycle wall, Step 2 through-pose wall,
  postprocess wall, inference wall, GPU/VRAM/power, PostgreSQL/Redis,
  embeddings, derived rows, telemetry, and terminal lifecycle state.
- [X] T000d [US6] Implement or extend production watcher/collector evidence so
  active long-running jobs expose live per-model RTT and stage-wall data before
  task end; end-of-task telemetry alone is insufficient for bottleneck triage.
- [X] T000e [US6] Design the remediation plan for the measured Cycle 015.0
  bottlenecks before additive XAI work: reduce postprocess wall, reduce
  inference/orchestration wall, bound pose-tail wall, and fix status/lifecycle
  convergence where `completed_at` is set while `status` remains non-terminal.
- [ ] T000f [US6] Execute a stride-1 production benchmark proving the
  `>=15 FPS` DB-completed and `32 frames <=2s` end-to-end target, with figures,
  rollback, and `docs/BENCHMARK_RESULTS_LEDGER.md` entry.

## Phase 1: Planning And Shared Governance Setup

- [X] T001 Create XAI/anomaly results directory and index in `docs/xai_anomaly/README.md`
- [X] T002 [P] Add feature-owned environment contract placeholders to `.env.example`
- [X] T003 [P] Add XAI/anomaly source path filters and focused test placeholders to `.github/workflows/xai-anomaly.yml`
- [X] T004 [P] Add static prohibited-accusation, manufactured-ground-truth, anomaly-training, prohibited-metric, and observed-pattern vocabulary verifier in `scripts/ci/verify_xai_language.py`
- [X] T005 [P] Add static signal/explainer registry coverage verifier in `scripts/ci/verify_xai_registry.py`
- [X] T005a [P] Add pretrained-models registry verifier (Class A frozen signal sources present with route snapshots; Class B carry `promotion_status = PROBE_ONLY`; no trainable anomaly target) in `scripts/ci/verify_pretrained_models.py` per `specs/015-xai-anomaly-score/pretrained-models-registry.md`
- [X] T005d [P] Add a reproducible pretrained-model acquisition manifest and cross-platform download helpers in `specs/015-xai-anomaly-score/pretrained-model-acquisition-manifest.json` and `scripts/models/download-xai-registry-models.{py,sh,ps1}`
- [X] T005e [P] Add focused unit coverage for the pretrained-model verifier and acquisition helper in `backend/tests/unit/scripts/test_verify_pretrained_models.py` and `backend/tests/unit/scripts/test_download_xai_registry_models.py`
- [X] T005b [P] Add static no-hardcoded-operational-constant and parameter-provenance verifier (every threshold/weight/envelope/geometric constant resolves to a learned baseline reference or a fingerprinted `.env`/config key with `learned`/`configured` provenance) in `scripts/ci/verify_no_hardcode.py`
- [X] T005c [P] Add static model-promotion-gate verifier (any `MANDATORY` model carries a complete `ModelPromotionRecord` with benchmark/serving-metrics/model-card/approver/rollback; `PROBE_ONLY`/`SHADOW`/`CANARY` outputs never feed a production score/state; `target_role` is signal/representation only; no behavioral-accuracy metric) in `scripts/ci/verify_model_promotion.py` per `specs/015-xai-anomaly-score/pretrained-models-registry.md`
- [X] T006 [P] Add static placeholder/artifact authenticity verifier in `scripts/ci/verify_xai_artifacts.py`
- [X] T007 [P] Define benchmark evidence root layout in `docs/xai_anomaly/benchmark_evidence_layout.md`
- [X] T008 [P] Define XAI privacy, access, and retention policy in `docs/xai_anomaly/security_retention_policy.md`
- [X] T009 [P] Define no-anomaly-ground-truth governance, reviewer-assessment non-training rules, and future-plan boundary in `docs/xai_anomaly/no_ground_truth_governance.md`
- [X] T010 [P] Add required observed-pattern terminology and knowledge-limit glossary in `docs/xai_anomaly/glossary.md`
- [X] T011 Define configuration ownership and fingerprint contract in `docs/xai_anomaly/configuration_contract.md`
- [X] T012 Define exactly-one Figure Planner and Figure Implementer kickoff template in `docs/xai_anomaly/figure_role_template.md`
- [X] T013 Define cycle result/decision template in `docs/xai_anomaly/cycle_result_template.md`
- [X] T014 Define cycle production preflight/rollback template in `docs/xai_anomaly/production_cycle_runbook.md`
- [X] T015 Run documentation and source-reference validation for the setup artifacts and record it in `docs/xai_anomaly/setup_validation.md`

## Phase 2: Cycle 015.0 - Runtime Truth, Route Snapshot, And BSIL Activation

**Goal**: Establish immutable runtime truth and prove BSIL/lineage activation.
**Independent test**: One production job yields a valid route snapshot and
reconciled nonzero BSIL/lineage records without source-output regression.

- [X] T016 [P] [US6] Write Cycle 015.0 investigation, hypothesis, streaming compatibility, gates, and rollback in `docs/xai_anomaly/cycle_015_0_investigation.md`
- [X] T017 [P] [US6] Name Cycle 015.0 Figure Planner and define raw inputs/plots/targets in `docs/xai_anomaly/cycle_015_0_figure_plan.md`
- [X] T018 [P] [US6] Name separate Cycle 015.0 Figure Implementer and claim generator ownership in `docs/xai_anomaly/cycle_015_0_figure_implementation.md`
- [X] T019 [P] [US1] Add immutable route snapshot value contract in `backend/apps/pipeline/services/model_route_snapshot.py`
- [X] T020 [US1] Remove shared-default route mutation and add snapshot/fingerprint APIs in `backend/apps/pipeline/services/model_route_service.py`
- [X] T021 [P] [US1] Add route snapshot PostgreSQL model, migration, and explicit serializer boundary in `backend/apps/behavior/models.py` and `backend/apps/behavior/serializers.py`
- [X] T022 [P] [US1] Audit and reconcile behavior label mappings in `backend/apps/pipeline/multi_model.py`
- [X] T023 [P] [US1] Audit and reconcile layer-specific behavior mappings in `backend/apps/pipeline/layers/`
- [X] T024 [P] [US2] Replace or deprecate accusatory rule-engine outputs in `backend/apps/pipeline/rule_engine.py`
- [X] T025 [US2] Wire BSIL semantic/temporal/lineage activation flags into existing offline and live orchestration in `backend/apps/video_analysis/tasks.py`
- [X] T026 [P] [US6] Add route immutability, mapping parity, vocabulary, and snapshot digest tests in `backend/tests/unit/pipeline/test_xai_runtime_truth.py`
- [X] T027 [P] [US2] Add BSIL activation/population/reconciliation integration tests in `backend/tests/integration/behavior/test_xai_bsil_activation.py`
- [X] T028 [P] [US6] Add production runtime-truth/BSIL preflight helper in `tools/prod/prod_xai_cycle015_0_preflight.py`
- [X] T029 [US6] Add Cycle 015.0 stride-1 production benchmark and metrics collector in `tools/prod/prod_run_xai_cycle015_0.sh`
- [X] T030 [P] [US6] Implement Cycle 015.0 figure generator and manifest tests in `tools/prod/prod_generate_xai_cycle015_0_figures.py`
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
- [ ] T040 [US1] Register all current raw/derived signals and bounded per-student pattern-feature definitions in `backend/apps/behavior/explainability/signal_definitions.py`
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

**Goal**: Produce compatible, measured calibration snapshots for active source
model outputs only where task-appropriate held-out evidence exists.
**Independent test**: Calibration improves or accurately characterizes
reliability and incompatible/stale snapshots are rejected.

- [ ] T051 [P] [US1] Write Cycle 015.2 source-model calibration protocol, unavailable-evidence behavior, and no-anomaly-ground-truth boundary in `docs/xai_anomaly/cycle_015_2_investigation.md`
- [ ] T052 [P] [US6] Name Cycle 015.2 Figure Planner in `docs/xai_anomaly/cycle_015_2_figure_plan.md`
- [ ] T053 [P] [US6] Name separate Cycle 015.2 Figure Implementer in `docs/xai_anomaly/cycle_015_2_figure_implementation.md`
- [ ] T054 [P] [US1] Implement calibration contracts and compatibility rules in `backend/apps/anomalies/scoring/calibration.py`
- [ ] T055 [P] [US1] Add calibration snapshot models and migration in `backend/apps/anomalies/models.py`
- [ ] T056 [P] [US1] Implement source-model calibration evidence-cohort manifest handling in `backend/apps/anomalies/scoring/calibration_artifacts.py`
- [ ] T057 [US1] Implement model/output-specific calibrator registry in `backend/apps/anomalies/scoring/calibrator_registry.py`
- [ ] T058 [US1] Implement calibration lookup and stale/incompatible rejection in `backend/apps/anomalies/scoring/calibration_service.py`
- [ ] T059 [P] [US1] Add calibration fitting/evaluation command in `backend/apps/anomalies/management/commands/build_xai_calibration.py`
- [ ] T060 [P] [US1] Add ECE, reliability, proper-score, CI, and subgroup evaluation in `backend/apps/anomalies/scoring/calibration_evaluation.py`
- [ ] T061 [P] [US1] Add calibration contract and compatibility tests in `backend/tests/unit/anomalies/test_xai_calibration.py`
- [ ] T062 [P] [US1] Add calibration evidence-cohort manifest, unavailable-state, no-anomaly-ground-truth, and PostgreSQL tests in `backend/tests/integration/anomalies/test_xai_calibration_artifacts.py`
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
- [ ] T091 [P] [US2] Implement bounded per-student multivariate signal-pattern windows in `backend/apps/behavior/explainability/temporal_windows.py`
- [ ] T092 [P] [US2] Implement temporal pattern explanation, cold-start, contamination, and quarantine semantics in `backend/apps/behavior/explainability/temporal_explanations.py`
- [ ] T093 [P] [US2] Implement episode contribution/counterfactual composition in `backend/apps/behavior/explainability/episode_explanations.py`
- [ ] T094 [US2] Integrate temporal evidence with existing behavior temporal services in `backend/apps/behavior/temporal/windows.py`
- [ ] T095 [US2] Add bounded cache/eviction/reconnect behavior in `backend/apps/behavior/temporal/cache.py`
- [ ] T096 [P] [US2] Add controlled pattern-fixture, spike, sustained, drift, cold-start, contamination, quarantine, gap, and deterministic replay tests in `backend/tests/unit/behavior/test_xai_temporal_explanations.py`
- [ ] T097 [P] [US2] Add indefinite-stream bounded-state and restart integration tests in `backend/tests/integration/behavior/test_xai_temporal_bounded_state.py`
- [ ] T098 [P] [US6] Add temporal state/soak production probe in `tools/prod/prod_probe_xai_temporal_state.py`
- [ ] T099 [US6] Add Cycle 015.4 stride-1 benchmark/collector in `tools/prod/prod_run_xai_cycle015_4.sh`
- [ ] T100 [P] [US6] Implement Cycle 015.4 temporal/soak figures, manifest/digests, and generator tests in `tools/prod/prod_generate_xai_cycle015_4_figures.py`
- [ ] T101 [P] [US6] Add live bounded-state soak helper in `tools/prod/prod_soak_xai_temporal_live.sh`
- [ ] T102 [US6] Execute rollback and reconciliation from `tools/prod/prod_rollback_xai_cycle015_4.sh`
- [ ] T103 [US6] Record Cycle 015.4 decision and figures in `docs/xai_anomaly/cycle_015_4_results.md`
- [ ] T104 [US6] Record every Cycle 015.4 run and decision in `docs/BENCHMARK_RESULTS_LEDGER.md`

## Phase 7: Cycle 015.5 - Transparent Hierarchical Anomaly Score

**Goal**: Produce a reconstructable non-accusatory review-priority score from
per-student observed signal-pattern comparison without anomaly training or
behavioral ground truth.
**Independent test**: Persisted contributions exactly reconstruct scores and
invalid evidence reliably withholds them.

- [X] T105 [P] [US2] Write Cycle 015.5 investigation, deterministic score profile, no-ground-truth evaluation protocol, and prohibited-metric list in `docs/xai_anomaly/cycle_015_5_investigation.md`
- [X] T106 [P] [US6] Name Cycle 015.5 Figure Planner in `docs/xai_anomaly/cycle_015_5_figure_plan.md`
- [X] T107 [P] [US6] Name separate Cycle 015.5 Figure Implementer in `docs/xai_anomaly/cycle_015_5_figure_implementation.md`
- [X] T108 [P] [US2] Implement anomaly score/contribution contracts in `backend/apps/anomalies/scoring/contracts.py`
- [X] T109 [P] [US2] Implement bounded contamination-aware observed-pattern profiles, compatible pattern comparison, and per-signal deviation components in `backend/apps/anomalies/scoring/pattern_profiles.py`, `backend/apps/anomalies/scoring/pattern_comparison.py`, and `backend/apps/anomalies/scoring/components.py`
- [X] T110 [P] [US2] Implement transparent pattern-deviation fusion, cold-start/quarantine, and withholding gates in `backend/apps/anomalies/scoring/fusion.py`
- [X] T111 [US2] Implement observed-pattern comparison and anomaly scoring orchestration service in `backend/apps/anomalies/scoring/service.py`
- [X] T112 [US2] Add signal-pattern window models in `backend/apps/behavior/models.py` and observed-pattern profile/score/contribution models with migrations in `backend/apps/anomalies/models.py`
- [X] T113 [US2] Add idempotent pattern-profile, score, and contribution repositories in `backend/apps/anomalies/services.py`
- [X] T114 [P] [US2] Add exact reconstruction and counterfactual service in `backend/apps/anomalies/scoring/reconstruction.py`
- [X] T115 [P] [US2] Add explicit score serializers and read endpoints in `backend/apps/anomalies/serializers.py` and `backend/apps/anomalies/views.py`
- [X] T116 [P] [US2] Add contribution math, controlled pattern fixtures, metamorphic/invariant, cold-start, contamination, quarantine, missingness, withholding, and vocabulary tests in `backend/tests/unit/anomalies/test_xai_review_priority_score.py`
- [X] T117 [P] [US2] Add PostgreSQL pattern-profile compatibility/quarantine, reconstruction, and idempotency integration tests in `backend/tests/integration/anomalies/test_xai_score_persistence.py`
- [X] T118 [P] [US2] Add no-ground-truth pattern invariant, sensitivity, counterfactual, and prohibited-metric evaluation command in `backend/apps/anomalies/management/commands/evaluate_xai_scores.py`
- [X] T119 [P] [US6] Add production pattern-profile/score reconstruction/no-ground-truth probe helper in `tools/prod/prod_probe_xai_scores.py`
- [X] T120 [US6] Add Cycle 015.5 stride-1 benchmark/collector in `tools/prod/prod_run_xai_cycle015_5.sh`
- [X] T121 [P] [US6] Implement Cycle 015.5 pattern-envelope, score decomposition, fixture/invariant, contamination/quarantine, manifest/digest figures and generator tests in `tools/prod/prod_generate_xai_cycle015_5_figures.py`
- [ ] T122 [US6] Execute rollback and record Cycle 015.5 decision in `docs/xai_anomaly/cycle_015_5_results.md`
- [ ] T123 [US6] Record every Cycle 015.5 run and decision in `docs/BENCHMARK_RESULTS_LEDGER.md`

## Phase 8: Cycle 015.6 - Missingness, Uncertainty, And Conformal Calibration

**Goal**: Add explicit uncertainty and valid conformal outputs without false
precision.
**Independent test**: Coverage is measured and outputs are withheld when
assumptions or drift gates fail.

- [X] T124 [P] [US2] Write Cycle 015.6 investigation, conformal assumption protocol, and distributional-not-behavioral coverage boundary in `docs/xai_anomaly/cycle_015_6_investigation.md`
- [X] T125 [P] [US6] Name Cycle 015.6 Figure Planner in `docs/xai_anomaly/cycle_015_6_figure_plan.md`
- [X] T126 [P] [US6] Name separate Cycle 015.6 Figure Implementer in `docs/xai_anomaly/cycle_015_6_figure_implementation.md`
- [X] T127 [P] [US2] Implement pattern-profile/window uncertainty and missingness propagation in `backend/apps/anomalies/scoring/uncertainty.py`
- [X] T128 [P] [US2] Implement conformal snapshot/contracts and assumption gates in `backend/apps/anomalies/scoring/conformal.py`
- [X] T129 [US2] Add conformal snapshot model and migration in `backend/apps/anomalies/models.py`
- [X] T130 [P] [US2] Implement distributional coverage/drift evaluation without behavioral-correctness claims in `backend/apps/anomalies/scoring/conformal_evaluation.py`
- [X] T131 [US2] Integrate uncertainty/conformal outputs into score service in `backend/apps/anomalies/scoring/service.py`
- [X] T132 [P] [US2] Add missingness, coverage, drift, and withholding unit tests in `backend/tests/unit/anomalies/test_xai_conformal_uncertainty.py`
- [X] T133 [P] [US2] Add adaptive bounded-state and PostgreSQL integration tests in `backend/tests/integration/anomalies/test_xai_conformal_runtime.py`
- [X] T134 [P] [US6] Add production conformal coverage/drift probe in `tools/prod/prod_probe_xai_conformal.py`
- [X] T135 [US6] Add Cycle 015.6 stride-1 benchmark/collector in `tools/prod/prod_run_xai_cycle015_6.sh`
- [X] T136 [P] [US6] Implement Cycle 015.6 coverage figures, manifest/digests, and generator tests in `tools/prod/prod_generate_xai_cycle015_6_figures.py`
- [ ] T137 [US6] Execute rollback from `tools/prod/prod_rollback_xai_cycle015_6.sh`
- [ ] T138 [US6] Record Cycle 015.6 decision in `docs/xai_anomaly/cycle_015_6_results.md`
- [ ] T139 [US6] Record every Cycle 015.6 run and decision in `docs/BENCHMARK_RESULTS_LEDGER.md`

## Phase 9: Cycle 015.7 - Cross-Model Fusion And Explanation Graph

**Goal**: Fuse evidence while preserving sources, contradictions, identity
limits, and graph lineage.
**Independent test**: Crowded/crossing/contradiction sequences keep unresolved
identity unresolved and reconstruct the explanation graph.

- [X] T140 [P] [US3] Write Cycle 015.7 investigation and identity-gated fusion protocol in `docs/xai_anomaly/cycle_015_7_investigation.md`
- [X] T141 [P] [US6] Name Cycle 015.7 Figure Planner in `docs/xai_anomaly/cycle_015_7_figure_plan.md`
- [X] T142 [P] [US6] Name separate Cycle 015.7 Figure Implementer in `docs/xai_anomaly/cycle_015_7_figure_implementation.md`
- [X] T143 [P] [US3] Implement source-preserving fusion policy in `backend/apps/anomalies/scoring/fusion.py`
- [X] T144 [P] [US3] Implement contradiction policy and visibility contract in `backend/apps/behavior/explainability/contradictions.py`
- [X] T145 [P] [US3] Implement identity-gated scene/SRVL/peer context in `backend/apps/behavior/explainability/context.py`
- [X] T146 [P] [US3] Implement explanation graph contracts and builder in `backend/apps/behavior/explainability/graph.py`
- [X] T147 [US3] Implement composed explanation records and persistence in `backend/apps/behavior/explainability/composer.py`
- [X] T148 [US3] Add explanation record models and migration in `backend/apps/behavior/models.py`
- [X] T149 [P] [US3] Add explanation graph serializers/endpoints in `backend/apps/behavior/serializers.py`
- [X] T150 [P] [US3] Add fusion, contradiction, identity, and graph reconstruction tests in `backend/tests/unit/behavior/test_xai_explanation_graph.py`
- [X] T151 [P] [US3] Add crossing/fragmentation/unresolved integration tests in `backend/tests/integration/behavior/test_xai_context_identity.py`
- [X] T152 [P] [US6] Add production fusion/identity/graph probe in `tools/prod/prod_probe_xai_fusion.py`
- [X] T153 [US6] Add Cycle 015.7 stride-1 benchmark/collector in `tools/prod/prod_run_xai_cycle015_7.sh`
- [X] T154 [P] [US6] Implement Cycle 015.7 graph/identity figures, manifest/digests, and generator tests in `tools/prod/prod_generate_xai_cycle015_7_figures.py`
- [ ] T155 [US6] Execute rollback from `tools/prod/prod_rollback_xai_cycle015_7.sh`
- [ ] T156 [US6] Record Cycle 015.7 decision in `docs/xai_anomaly/cycle_015_7_results.md`
- [ ] T157 [US6] Record every Cycle 015.7 run and decision in `docs/BENCHMARK_RESULTS_LEDGER.md`

## Phase 10: Cycle 015.8 - Offline And On-Demand Deep Vision XAI

**Goal**: Provide isolated, faithful, evaluated deep-XAI artifacts.
**Independent test**: Eligible methods complete idempotently with fidelity,
sanity, stability, security, and critical-path isolation evidence.

- [X] T158 [P] [US4] Write Cycle 015.8 investigation and per-model method eligibility matrix in `docs/xai_anomaly/cycle_015_8_investigation.md`
- [X] T159 [P] [US6] Name Cycle 015.8 Figure Planner in `docs/xai_anomaly/cycle_015_8_figure_plan.md`
- [X] T160 [P] [US6] Name separate Cycle 015.8 Figure Implementer in `docs/xai_anomaly/cycle_015_8_figure_implementation.md`
- [X] T161 [P] [US4] Implement deep-XAI request/artifact/evaluation contracts in `backend/apps/behavior/explainability/deep_contracts.py`
- [X] T162 [US4] Add deep-XAI request/artifact/evaluation models and migration in `backend/apps/behavior/models.py`
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

- [ ] T178 [P] [US4] Write Cycle 015.9 non-ground-truth reviewer-usability/disagreement study protocol in `docs/xai_anomaly/cycle_015_9_investigation.md`
- [ ] T179 [P] [US6] Name Cycle 015.9 Figure Planner in `docs/xai_anomaly/cycle_015_9_figure_plan.md`
- [ ] T180 [P] [US6] Name separate Cycle 015.9 Figure Implementer in `docs/xai_anomaly/cycle_015_9_figure_implementation.md`
- [ ] T181 [P] [US4] Implement governed prototype/exemplar contracts in `backend/apps/behavior/explainability/prototypes.py`
- [ ] T182 [P] [US4] Implement similar-case comparison service in `backend/apps/behavior/explainability/case_comparison.py`
- [ ] T183 [US4] Add reviewer-assessment model/migration and no-ground-truth/no-training/no-direct-mutation guards in `backend/apps/anomalies/models.py`
- [ ] T184 [US4] Implement non-ground-truth reviewer assessment/audit service in `backend/apps/anomalies/scoring/review_feedback.py`
- [ ] T185 [P] [US4] Add prototype/case/review endpoints in `backend/apps/anomalies/views.py`
- [ ] T186 [P] [US4] Add feedback immutability, access, no-ground-truth, no-training-target, no-profile-update, and no-direct-mutation tests in `backend/tests/unit/anomalies/test_xai_review_feedback.py`
- [ ] T187 [P] [US4] Add reviewer workflow/audit integration tests in `backend/tests/integration/anomalies/test_xai_review_workflow.py`
- [ ] T188 [P] [US6] Add explicitly non-ground-truth reviewer-usability/disagreement evidence collector in `tools/prod/prod_collect_xai_reviewer_study.py`
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
- [ ] T223 [P] [US6] Add source-model calibration/subgroup/explanation stability and no-ground-truth pattern-invariant evaluator in `backend/apps/anomalies/scoring/integrated_evaluation.py`
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
- [ ] T239 [P] [US6] Add canary stop/promote policy evaluator that vetoes anomaly training, behavioral-ground-truth, prohibited-metric, and vocabulary violations in `backend/apps/behavior/explainability/canary_policy.py`
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

## Phase 15: Cycle 015.13 - Student Interaction Graph Signals And Plots

**Goal**: Add a deterministic, identity-gated student-interaction graph that
emits per-student graph signals for scoring and renders node-link/adjacency
plots through the shared WebGL2 core, without any collusion/cheating claim and
with learned graph models kept `PROBE_ONLY`.
**Independent test**: The graph is reconstructable from referenced relational
signals, ambiguous-identity edges stay unresolved, per-student features feed the
observed-pattern profile, and the graph renders in WebGL2 within budget with a
numeric fallback.

- [ ] T251 [P] [US3] Write Cycle 015.13 investigation, deterministic-graph contract, non-accusatory/identity-gating gates, and rollback in `docs/xai_anomaly/cycle_015_13_investigation.md`
- [ ] T252 [P] [US6] Name Cycle 015.13 Figure Planner in `docs/xai_anomaly/cycle_015_13_figure_plan.md`
- [ ] T253 [P] [US6] Name separate Cycle 015.13 Figure Implementer in `docs/xai_anomaly/cycle_015_13_figure_implementation.md`
- [ ] T254 [P] [US3] Implement interaction-graph node/edge/feature contracts in `backend/apps/behavior/explainability/interaction_graph_contracts.py`
- [ ] T255 [P] [US3] Implement deterministic identity-gated graph builder (proximity, directed/mutual gaze cone, orientation, co-movement, shared-scene edges) reusing SRVL/gaze/pose/scene signals in `backend/apps/behavior/explainability/interaction_graph.py`
- [ ] T256 [P] [US2] Register per-student graph-derived feature signals (degree, mutual-gaze dwell, directed-attention, persistence, clustering/centrality proxy, dyad strength) feeding observed-pattern profiles in `backend/apps/behavior/explainability/signal_definitions.py`
- [ ] T257 [US3] Add `StudentInteractionGraphFrame` model, bounds, and migration in `backend/apps/behavior/models.py`
- [ ] T258 [P] [US3] Add interaction-graph serializers and bounded chunked endpoints in `backend/apps/behavior/serializers.py` and `backend/apps/behavior/views.py`
- [ ] T259 [P] [US5] Implement shared-WebGL2 node-link and adjacency graph layers as a **live, real-time, continuously-updating** plot (incremental append on `xai.interaction_graph.appended`, latest-frame-wins, rendered alongside the other plots) in `frontend/src/services/webgl/GraphLayer.ts` and the graph view in `frontend/src/features/xai/`
- [ ] T260 [P] [US3] Add identity-gating, unresolved-edge, node/edge-bound/overflow, and deterministic-reconstruction unit tests in `backend/tests/unit/behavior/test_xai_interaction_graph.py`
- [ ] T261 [P] [US2] Add graph-feature determinism, profile-feed, and no-peer-mutation tests in `backend/tests/unit/behavior/test_xai_interaction_graph_signals.py`
- [ ] T262 [P] [US5] Add WebGL graph render, live real-time incremental-update/soak, context-budget, large-graph LOD, context-loss, backpressure-drop, and numeric-fallback tests in `frontend/src/services/webgl/__tests__/graphLayer.test.ts`
- [ ] T263 [P] [US3] Add `PROBE_ONLY` learned-graph isolation guard proving no learned graph model drives a production score in `backend/tests/unit/behavior/test_xai_interaction_graph_probe_only.py`
- [ ] T264 [P] [US6] Add production graph signal/render coverage probe in `tools/prod/prod_probe_xai_interaction_graph.py`
- [ ] T265 [US6] Add Cycle 015.13 stride-1 benchmark/collector in `tools/prod/prod_run_xai_cycle015_13.sh`
- [ ] T266 [P] [US6] Implement Cycle 015.13 graph/adjacency/feature figures, manifest/digests, and generator tests in `tools/prod/prod_generate_xai_cycle015_13_figures.py`
- [ ] T267 [US6] Execute rollback and record Cycle 015.13 decision in `docs/xai_anomaly/cycle_015_13_results.md`
- [ ] T268 [US6] Record every Cycle 015.13 run and decision in `docs/BENCHMARK_RESULTS_LEDGER.md`

## Phase 16: Cycle 015.14 - General Baseline Ingestion, Dual Comparison, And Parameter Provenance

**Goal**: Learn a corpus-ingested, contamination-aware general baseline
(population/context tiers), score by dual comparison against it and the student's
own profile, and bind every operational value to a `learned`/`configured`
provenance so nothing is hardcoded — with XAI reconstructability as the priority.
**Independent test**: Scores expose `deviation_vs_self` and
`deviation_vs_population`, reconstructable from both snapshots; the general
baseline stays assumed-normal (never ground truth); and the no-hardcoding
verifier passes with every value resolved to a provenance record.

- [ ] T269 [P] [US2] Write Cycle 015.14 investigation, general-baseline ingestion protocol, dual-comparison/withholding gates, no-hardcoding policy, and rollback in `docs/xai_anomaly/cycle_015_14_investigation.md`
- [ ] T270 [P] [US6] Name Cycle 015.14 Figure Planner in `docs/xai_anomaly/cycle_015_14_figure_plan.md`
- [ ] T271 [P] [US6] Name separate Cycle 015.14 Figure Implementer in `docs/xai_anomaly/cycle_015_14_figure_implementation.md`
- [ ] T272 [P] [US2] Implement the General Population Baseline (population/context tier, aggregated across >= configured minimum distinct students/sessions, never from a single student; General Boundaries vs Local Boundaries) contracts and robust-statistics reuse in `backend/apps/anomalies/scoring/general_baseline.py`
- [ ] T273 [P] [US2] Implement offline corpus-ingestion pass and command in `backend/apps/anomalies/scoring/baseline_ingestion.py` and `backend/apps/anomalies/management/commands/build_general_baseline.py`
- [ ] T274 [US2] Implement dual comparison (`deviation_vs_self` vs Local Boundaries + `deviation_vs_population` vs General Boundaries), a general classroom-level deviation, tier precedence, and tier-disagreement visibility in `backend/apps/anomalies/scoring/pattern_comparison.py` and `backend/apps/anomalies/scoring/service.py`
- [ ] T275 [P] [US1] Implement the single parameter resolver and `learned`/`configured` provenance binding in `backend/apps/anomalies/scoring/parameter_resolver.py` and `backend/apps/anomalies/scoring/parameter_provenance.py`
- [ ] T276 [US2] Add `baseline_tier` to observed-pattern profiles and a `ParameterProvenanceRecord` model with migration in `backend/apps/anomalies/models.py`
- [ ] T277 [P] [US2] Add general-baseline and provenance serializers and bounded endpoints in `backend/apps/anomalies/serializers.py` and `backend/apps/anomalies/views.py`
- [ ] T278 [P] [US2] Add dual-comparison, general-baseline cold-start/contamination/quarantine/drift, and tier-disagreement unit tests in `backend/tests/unit/anomalies/test_xai_general_baseline.py`
- [ ] T279 [P] [US1] Add no-hardcoding and provenance-reconstruction unit tests (every value `learned` or `configured`, none inline) in `backend/tests/unit/anomalies/test_xai_parameter_provenance.py`
- [ ] T280 [P] [US2] Add PostgreSQL baseline-tier and provenance reconstruction/idempotency integration tests in `backend/tests/integration/anomalies/test_xai_general_baseline_persistence.py`
- [ ] T281 [P] [US6] Add production baseline-ingestion, dual-comparison, and provenance coverage probe in `tools/prod/prod_probe_xai_general_baseline.py`
- [ ] T282 [US6] Add Cycle 015.14 stride-1 benchmark/collector in `tools/prod/prod_run_xai_cycle015_14.sh`
- [ ] T283 [P] [US6] Implement Cycle 015.14 population-vs-self envelope, dual-delta, and provenance-coverage figures, manifest/digests, and generator tests in `tools/prod/prod_generate_xai_cycle015_14_figures.py`
- [ ] T284 [US6] Execute rollback and record Cycle 015.14 decision in `docs/xai_anomaly/cycle_015_14_results.md`
- [ ] T285 [US6] Record every Cycle 015.14 run and decision in `docs/BENCHMARK_RESULTS_LEDGER.md`

## Phase 17: Cycle 015.15 - Model Promotion Governance (Probe -> Mandatory)

**Goal**: Make a research/probe model promotable to a production-mandatory role
only through an evidence-gated lifecycle (`PROBE_ONLY` -> `SHADOW` -> `CANARY` ->
`MANDATORY`) with a native benchmark, computed serving metrics, governed-approver
sign-off, and proven rollback — and only as a governed signal/representation,
never a behavioral judge.
**Independent test**: A probe advances only with a complete `ModelPromotionRecord`;
`PROBE_ONLY`/`SHADOW`/`CANARY` outputs never feed a production score; no behavioral
accuracy/AUROC is recorded; and a rollback restores the prior stage.

- [ ] T286 [P] [US6] Write Cycle 015.15 investigation, promotion lifecycle/gates (G1-G6), doctrine cap, and rollback in `docs/xai_anomaly/cycle_015_15_investigation.md`
- [ ] T287 [P] [US6] Name Cycle 015.15 Figure Planner in `docs/xai_anomaly/cycle_015_15_figure_plan.md`
- [ ] T288 [P] [US6] Name separate Cycle 015.15 Figure Implementer in `docs/xai_anomaly/cycle_015_15_figure_implementation.md`
- [ ] T289 [P] [US6] Implement promotion lifecycle/stage/gate contracts in `backend/apps/pipeline/services/model_promotion.py`
- [ ] T290 [US6] Add `ModelPromotionRecord` model, `promotion_status` enum, and migration in `backend/apps/pipeline/models.py`
- [ ] T291 [US6] Implement shadow/canary runner and champion/challenger serving-metrics collector (latency/throughput/resource, serving distribution, shadow duration) in `backend/apps/pipeline/services/model_promotion.py`
- [ ] T292 [P] [US6] Implement governed-approver/RBAC sign-off, decision, and automated rollback service in `backend/apps/pipeline/services/model_promotion.py`
- [ ] T293 [P] [US6] Add promotion serializers and authenticated endpoints in `backend/apps/pipeline/serializers.py` and `backend/apps/pipeline/views.py`
- [ ] T294 [P] [US6] Add model-card template and generator in `docs/xai_anomaly/model_card_template.md` and `backend/apps/pipeline/services/model_card.py`
- [ ] T295 [P] [US6] Add promotion-gate, doctrine-cap (signal-role-only, no behavioral accuracy/AUROC), and record-completeness unit tests in `backend/tests/unit/pipeline/test_xai_model_promotion.py`
- [ ] T296 [P] [US6] Add shadow/canary serving-metrics, champion/challenger, and auto-rollback integration tests in `backend/tests/integration/pipeline/test_xai_promotion_runtime.py`
- [ ] T297 [P] [US6] Add production shadow/canary promotion-evidence probe in `tools/prod/prod_probe_xai_model_promotion.py`
- [ ] T298 [US6] Add Cycle 015.15 stride-1 benchmark/collector in `tools/prod/prod_run_xai_cycle015_15.sh`
- [ ] T299 [P] [US6] Implement Cycle 015.15 serving-distribution, latency-SLO, and gate-coverage figures, manifest/digests, and generator tests in `tools/prod/prod_generate_xai_cycle015_15_figures.py`
- [ ] T300 [US6] Execute rollback and record Cycle 015.15 decision in `docs/xai_anomaly/cycle_015_15_results.md`
- [ ] T301 [US6] Record every Cycle 015.15 run and decision in `docs/BENCHMARK_RESULTS_LEDGER.md`

## Phase 18: Cycle 015.16 - Probe Fine-Tuning Lane (Corpus Adaptation Of Copies)

**Goal**: Adapt COPIES of probe models on the ingested corpus through governed
options — filtered pseudo-label self-training, frozen-vs-fine-tuned
champion/challenger, self-supervised continued pretraining, ephemeral test-time
adaptation, and distillation from governed deterministic signals — with full
reconstruction, no parent mutation, and no ground-truth claims.
**Independent test**: A fine-tune run reproduces from its corpus manifest,
filter-policy provenance, and accept/refuse/edit ledger; the frozen parent digest
verifies unchanged; benchmark deltas vs the parent and the accepted deterministic
baseline resolve; and the copy stays `PROBE_ONLY` absent a promotion record.

- [ ] T302 [P] [US6] Write Cycle 015.16 investigation, option matrix (a-e), filter policy, error-accumulation counters, and rollback in `docs/xai_anomaly/cycle_015_16_investigation.md`
- [ ] T303 [P] [US6] Name Cycle 015.16 Figure Planner in `docs/xai_anomaly/cycle_015_16_figure_plan.md`
- [ ] T304 [P] [US6] Name separate Cycle 015.16 Figure Implementer in `docs/xai_anomaly/cycle_015_16_figure_implementation.md`
- [ ] T305 [P] [US6] Implement fine-tune run/option/ledger contracts in `backend/apps/pipeline/services/probe_finetune.py`
- [ ] T306 [US6] Add `ProbeFineTuneRun` model and migration in `backend/apps/pipeline/models.py`
- [ ] T307 [US6] Implement accepted-standard-method inference filter (accept/refuse/edit, deterministic-baseline agreement, identity gating, temporal consistency, confidence/entropy gates — all parameter-provenance-bound) in `backend/apps/pipeline/services/probe_finetune_filter.py`
- [ ] T308 [P] [US6] Implement copy-based fine-tune executors for options (a)/(c)/(e) and the ephemeral-TTA guard (d) in `backend/apps/pipeline/services/probe_finetune_runners.py`
- [ ] T309 [P] [US6] Implement frozen-vs-fine-tuned champion/challenger comparison (b) on held-out corpus with serving/stability/drift/agreement deltas in `backend/apps/pipeline/services/probe_finetune_compare.py`
- [ ] T310 [P] [US6] Add parent-digest-immutability, ledger-completeness, no-ground-truth-vocabulary, and TTA-non-persistence unit tests in `backend/tests/unit/pipeline/test_xai_probe_finetune.py`
- [ ] T311 [P] [US6] Add reconstruction/idempotency and PostgreSQL integration tests in `backend/tests/integration/pipeline/test_xai_probe_finetune_persistence.py`
- [ ] T312 [P] [US6] Add reviewer-exclusion-only guard tests (feedback can quarantine corpus windows, never label) in `backend/tests/unit/pipeline/test_xai_probe_finetune_review_guard.py`
- [ ] T313 [P] [US6] Add production fine-tune-lane probe in `tools/prod/prod_probe_xai_probe_finetune.py`
- [ ] T314 [US6] Add Cycle 015.16 stride-1 benchmark/collector in `tools/prod/prod_run_xai_cycle015_16.sh`
- [ ] T315 [P] [US6] Implement Cycle 015.16 figures (accept/refuse/edit distribution, parent-vs-child deltas, baseline-agreement trend, drift), manifest/digests, and generator tests in `tools/prod/prod_generate_xai_cycle015_16_figures.py`
- [ ] T316 [US6] Execute rollback and record Cycle 015.16 decision in `docs/xai_anomaly/cycle_015_16_results.md`
- [ ] T317 [US6] Record every Cycle 015.16 run and decision in `docs/BENCHMARK_RESULTS_LEDGER.md`

## Phase 19: Cycle 015.17 - Staged Persistence And Postprocess Offload Pipeline

**Goal**: Remediate the measured Cycle 015.0 throughput collapse (1.773
DB-completed FPS, 0 mid-run authoritative rows, idle GPU, postprocess 68.753%
of summed stage time) with a default-off, fail-closed, CROSS-PROCESS
producer-consumer pipeline: bounded Redis Stream lanes + separate Celery
consumer workers (research Decision 20; NOT the rejected in-process 20.E
shape). Targets the constitution ladder: >=15 FPS DB-completed first, then the
25-30 FPS semi-realtime last-stage target.
**Independent test**: With the lane enabled on the canonical stride-1 video,
mid-run authoritative row counts are nonzero, row/parity/identity/lifecycle
correctness is exact versus serial, every lane drains with failures serially
reconciled before terminal state, and DB-completed FPS does not regress.

- [X] T318 [US6] Verify the flush-only-at-end hypothesis against production evidence and record the verdict + accepted architecture in `docs/xai_anomaly/cycle_015_throughput_remediation_investigation.md`
- [X] T319 [US6] Implement the producer-consumer pipeline module (bounded lane streams, compact idempotent packet contract, consumer-group consumer with XAUTOCLAIM retry, drain/reconcile API, watcher-readable counters) in `backend/apps/video_analysis/persistence_pipeline.py`
- [X] T320 [US6] Add default-off settings + environment contract in `backend/config/settings/base.py` and `.env.example`
- [X] T321 [US6] Add pipeline contract unit tests (disabled-default, live-exclusion, packet roundtrip/digest, bounded enqueue, fail-closed enqueue, consumer apply/ack/drain, no-writer abort, max-attempt failure accounting, drain-and-collect) in `backend/tests/unit/video_analysis/test_persistence_pipeline.py`
- [X] T322 [US6] Wire the new tests into CI in `.github/workflows/inference-parallelization.yml`
- [ ] T323 [US6] Implement the `db_rows` lane writer adapter over the existing idempotent packet writer and register it in `backend/apps/video_analysis/tasks.py`
- [ ] T324 [US6] Integrate the producer seam into the offline frame loop / Step 3 (enqueue instead of inline persist when enabled; serial fallback + drain/reconcile before terminal state) in `backend/apps/video_analysis/tasks.py`
- [ ] T325 [US6] Extend the production watcher with per-lane depth/pending/applied/failed evidence in `tools/prod/prod_watch_benchmark_metrics.sh`
- [ ] T326 [US6] Fix lifecycle/status terminal convergence (RC-8: completed evidence while `status=embedding`) with reconciler proof in `backend/apps/video_analysis/tasks.py`
- [ ] T327 [US6] Offload postprocess fanout construction (row building, derived records) into the consumer lanes, keeping ordered tracking on the producer path, in `backend/apps/video_analysis/tasks.py` + `backend/apps/video_analysis/persistence_pipeline.py`
- [ ] T328 [US6] Add embedding-lane consumer slice (separate process embedding application) behind the same flag in `backend/apps/video_analysis/persistence_pipeline.py`
- [ ] T329 [US6] Add Cycle 015.17 stride-1 baseline/candidate benchmark runner in `tools/prod/prod_run_xai_cycle015_17.sh`
- [ ] T330 [US6] Run the native RTX 5090 stride-1 baseline/candidate pair (after the GPU is free of training runs) and record parity/identity/lifecycle/FPS evidence
- [ ] T331 [US6] Generate Cycle 015.17 figures + manifest/digests in `tools/prod/prod_generate_xai_cycle015_17_figures.py`
- [ ] T332 [US6] Execute rollback (flag off → serial parity proof) and record the Cycle 015.17 decision in `docs/xai_anomaly/cycle_015_17_results.md`
- [ ] T333 [US6] Record every Cycle 015.17 run and decision in `docs/BENCHMARK_RESULTS_LEDGER.md`

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
015.3 + 015.5 + 015.7 + 015.10 -> 015.13
015.4 + 015.5 -> 015.14
015.0 + 015.8 -> 015.15
015.14 + 015.15 -> 015.16
015.0 (throughput baseline evidence) -> 015.17 (BLOCKS additive critical-path acceptance)
015.0 through 015.10, plus 015.13 + 015.14 + 015.15 + 015.16 + 015.17 -> 015.11 -> 015.12
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
