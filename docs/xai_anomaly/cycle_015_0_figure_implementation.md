# Cycle 015.0 Figure Implementation

**Last updated:** 2026-06-10
**Cycle:** `015.0`
**Role:** `Figure Implementer`
**Owner:** `Codex Cycle 015.0 Figure Implementer`
**Status:** `local_tests_passed_production_pending`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `backend/apps/pipeline/services/model_route_snapshot.py` |
| File | `tools/prod/prod_generate_xai_cycle015_0_figures.py` |
| File | `backend/tests/unit/pipeline/test_prod_generate_xai_cycle015_0_figures.py` |
| File | `backend/tests/unit/pipeline/test_prod_run_xai_cycle015_0_shell_syntax.py` |
| Doc | `docs/xai_anomaly/cycle_015_0_figure_plan.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |

## Ownership claim

- generator path: `tools/prod/prod_generate_xai_cycle015_0_figures.py`
- manifest validation path:
  `backend/tests/unit/pipeline/test_prod_generate_xai_cycle015_0_figures.py`
- current local responsibility: generate all planned runtime, benchmark,
  model, resource, storage, parity, completeness, unavailable, and rollback
  figures from production artifacts

## Current implementation lane

The generator consumes separate baseline, candidate, and best-comparable metric
artifacts plus baseline/candidate preflights and rollback proof. It produces the
twelve Figure Planner targets, input/output SHA-256 digests, Markdown embeds,
and per-run unavailable-metric disclosures. The runner passes the disabled
baseline metrics as the best-comparable input for this first XAI cycle.

Production-generated images remain pending the native Linux RTX 5090 benchmark
and rollback artifacts. The local generator and manifest tests do not create a
Cycle 015.0 decision.

## Role separation

Only one implementation agent was active, so a second agent could not own this
lane. The planner and implementer names are distinct, and their evidence remains
separated between the plan document and this implementation record.
