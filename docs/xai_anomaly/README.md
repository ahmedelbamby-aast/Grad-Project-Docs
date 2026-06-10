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
  identity visibility.

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
| Commit | `7e8e870e` |
| Doc | `specs/015-xai-anomaly-score/spec.md` |
| Doc | `specs/015-xai-anomaly-score/plan.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |
| Doc | `specs/015-xai-anomaly-score/no-ground-truth-doctrine.md` |
| Doc | `specs/015-xai-anomaly-score/contracts/anomaly-score-contract.md` |
| Doc | `specs/015-xai-anomaly-score/contracts/benchmark-evidence-contract.md` |
| Doc | `docs/xai_anomaly/cycle_015_6_results.md` |
| Doc | `docs/xai_anomaly/cycle_015_7_investigation.md` |
