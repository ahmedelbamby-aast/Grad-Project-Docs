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
| Commit | `7e8e870e` |
| Doc | `specs/015-xai-anomaly-score/spec.md` |
| Doc | `specs/015-xai-anomaly-score/plan.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |
| Doc | `specs/015-xai-anomaly-score/no-ground-truth-doctrine.md` |
| Doc | `specs/015-xai-anomaly-score/contracts/anomaly-score-contract.md` |
| Doc | `specs/015-xai-anomaly-score/contracts/benchmark-evidence-contract.md` |
