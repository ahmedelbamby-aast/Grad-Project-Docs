# XAI Anomaly Docs

**Last updated:** 2026-06-10

This directory holds feature-specific investigation, figure-planning, results,
and benchmark-support docs for `specs/015-xai-anomaly-score/`.

## Current files

| File | Purpose | Status |
|---|---|---|
| `cycle_015_5_investigation.md` | Deterministic score profile, no-ground-truth protocol, prohibited metrics, and current implementation boundary for Cycle 015.5 | active |
| `cycle_015_5_figure_plan.md` | Figure Planner output for Cycle 015.5 score evidence | active |
| `cycle_015_5_figure_implementation.md` | Figure Implementer ownership and generator/test evidence for Cycle 015.5 | active |
| `cycle_015_5_results.md` | Local validation state, remaining production-only gaps, and rollback boundary for Cycle 015.5 | active |
| `cycle_015_6_investigation.md` | Conformal assumption protocol, uncertainty propagation rules, and the distributional-only coverage boundary for Cycle 015.6 | active |
| `cycle_015_6_figure_plan.md` | Figure Planner output for Cycle 015.6 uncertainty/conformal evidence | active |
| `cycle_015_6_figure_implementation.md` | Figure Implementer ownership for the Cycle 015.6 local slice | active |
| `cycle_015_6_results.md` | Local validation state, helper-script readiness, and remaining production-only gaps for Cycle 015.6 | active |
| `cycle_015_7_investigation.md` | Identity-gated fusion protocol and explanation-graph local implementation boundary for Cycle 015.7 | active |
| `cycle_015_7_figure_plan.md` | Figure Planner output for Cycle 015.7 graph and contradiction evidence | active |
| `cycle_015_7_figure_implementation.md` | Figure Implementer ownership for the Cycle 015.7 local slice | active |
| `cycle_015_7_results.md` | Local validation state, helper-script readiness, and remaining production-only gaps for Cycle 015.7 | active |
| `cycle_015_8_investigation.md` | Deep-XAI request/artifact/evaluation contract and local persistence boundary for Cycle 015.8 | active |
| `cycle_015_8_figure_plan.md` | Figure Planner output for Cycle 015.8 deep-XAI lifecycle evidence | active |
| `cycle_015_8_figure_implementation.md` | Figure Implementer ownership for the Cycle 015.8 local contract slice | active |
| `cycle_015_0_investigation.md` | Cycle 015.0 runtime-truth local investigation for route immutability, snapshot digests, and BSIL activation prerequisites | active |
| `cycle_015_0_figure_plan.md` | Figure Planner output for Cycle 015.0 runtime-truth and BSIL activation evidence | active |
| `cycle_015_0_figure_implementation.md` | Figure Implementer ownership record for the Cycle 015.0 local route-snapshot slice | active |
| `benchmark_evidence_layout.md` | Feature-owned layout for future Cycle 015 raw benchmark results, manifests, figures, and ledger links | active |
| `security_retention_policy.md` | Feature-owned privacy, access, audit, and retention policy for Cycle 015 XAI artifacts | active |
| `no_ground_truth_governance.md` | Setup-slice governance rules for reviewer feedback, prohibited metrics, and future trainable-model boundaries | active |
| `glossary.md` | Required observed-pattern vocabulary and knowledge-limit terminology for Cycle 015 docs and payloads | active |
| `configuration_contract.md` | Ownership and fingerprint contract for all configured or learned Cycle 015 operational values | active |
| `figure_role_template.md` | Required exactly-one Figure Planner / Figure Implementer kickoff template | active |
| `cycle_result_template.md` | Standard Cycle 015 results/decision template with production-evidence and rollback sections | active |
| `production_cycle_runbook.md` | Template preflight, benchmark, figure, and rollback runbook for future production Cycle 015 runs | active |
| `setup_validation.md` | Record of documentation and source-reference validation for the setup-slice artifacts | active |

## Current implementation state

Cycle 015.5 currently has:

- deterministic score contracts, pattern profiles, comparison, fusion,
  reconstruction, evaluation, persistence models, repositories, persisted-first
  read path, probe helper, and figure generator in the working tree;
- DB-free unit coverage for the full score slice;
- PostgreSQL-backed integration coverage now passing locally in
  `backend/tests/integration/anomalies/test_xai_score_persistence.py`; and
- a staged-local-only results doc that records the remaining production-only
  gaps before Cycle 015.5 can claim a benchmark decision.

Cycle 015.6 currently has:

- explicit uncertainty propagation for missing profile features, degraded
  features, and invalid windows;
- governed conformal snapshots with route/schema/expiry/drift gates;
- distributional-only coverage evaluation helpers; and
- PostgreSQL-backed conformal snapshot persistence, local unit/integration
  validation, and file-driven probe/figure/rollback helper tooling.

Cycle 015.7 currently has:

- source-preserving fusion summaries for score contributions and context rows;
- identity-gated peer/scene context with unresolved peer context kept
  suppressed rather than coerced;
- deterministic explanation graph nodes/edges plus persisted explanation
  records and a read endpoint; and
- local unit/integration validation for graph reconstruction and unresolved
  identity visibility plus file-driven probe/figure/rollback helper tooling.

Cycle 015.8 currently has:

- deep-XAI request/artifact/evaluation dataclass contracts;
- persisted ORM models and migrations for the deep lifecycle boundary; and
- focused unit validation for method allowlisting, idempotency, deadlines, and
  evaluation digest semantics.

## Related sources

| Kind | Reference |
|---|---|
| File | `backend/apps/anomalies/scoring/contracts.py` |
| File | `backend/apps/anomalies/scoring/service.py` |
| File | `backend/apps/anomalies/scoring/reconstruction.py` |
| File | `backend/apps/anomalies/scoring/pattern_profiles.py` |
| File | `backend/apps/anomalies/scoring/pattern_comparison.py` |
| File | `backend/apps/anomalies/scoring/components.py` |
| File | `backend/apps/anomalies/scoring/fusion.py` |
| File | `backend/apps/anomalies/services.py` |
| File | `backend/apps/anomalies/management/commands/evaluate_xai_scores.py` |
| File | `tools/prod/prod_probe_xai_scores.py` |
| File | `tools/prod/prod_generate_xai_cycle015_5_figures.py` |
| File | `backend/tests/integration/anomalies/test_xai_score_persistence.py` |
| File | `backend/apps/anomalies/scoring/uncertainty.py` |
| File | `backend/apps/anomalies/scoring/conformal.py` |
| File | `backend/apps/anomalies/scoring/conformal_evaluation.py` |
| File | `backend/tests/unit/anomalies/test_xai_conformal_uncertainty.py` |
| File | `backend/tests/integration/anomalies/test_xai_conformal_runtime.py` |
| File | `tools/prod/prod_probe_xai_conformal.py` |
| File | `tools/prod/prod_generate_xai_cycle015_6_figures.py` |
| File | `tools/prod/prod_run_xai_cycle015_6.sh` |
| File | `tools/prod/prod_rollback_xai_cycle015_6.sh` |
| File | `backend/apps/behavior/explainability/contradictions.py` |
| File | `backend/apps/behavior/explainability/context.py` |
| File | `backend/apps/behavior/explainability/graph.py` |
| File | `backend/apps/behavior/explainability/composer.py` |
| File | `backend/tests/unit/behavior/test_xai_explanation_graph.py` |
| File | `backend/tests/integration/behavior/test_xai_context_identity.py` |
| File | `tools/prod/prod_probe_xai_fusion.py` |
| File | `tools/prod/prod_generate_xai_cycle015_7_figures.py` |
| File | `tools/prod/prod_run_xai_cycle015_7.sh` |
| File | `tools/prod/prod_rollback_xai_cycle015_7.sh` |
| File | `backend/apps/behavior/explainability/deep_contracts.py` |
| File | `backend/tests/unit/behavior/test_xai_deep_methods.py` |
| Commit | `7e8e870e` |
| Doc | `specs/015-xai-anomaly-score/spec.md` |
| Doc | `specs/015-xai-anomaly-score/plan.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |
| Doc | `specs/015-xai-anomaly-score/no-ground-truth-doctrine.md` |
| Doc | `specs/015-xai-anomaly-score/contracts/anomaly-score-contract.md` |
| Doc | `specs/015-xai-anomaly-score/contracts/benchmark-evidence-contract.md` |
| Doc | `docs/xai_anomaly/benchmark_evidence_layout.md` |
| Doc | `docs/xai_anomaly/security_retention_policy.md` |
| Doc | `docs/xai_anomaly/cycle_015_6_results.md` |
| Doc | `docs/xai_anomaly/cycle_015_7_investigation.md` |
| Doc | `docs/xai_anomaly/cycle_015_7_results.md` |
| Doc | `docs/xai_anomaly/cycle_015_8_investigation.md` |
