# Cycle 015.0 Figure Implementation

**Last updated:** 2026-06-10
**Cycle:** `015.0`
**Role:** `Figure Implementer`
**Owner:** `Codex local Cycle 015.0 session`
**Status:** `active`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `backend/apps/pipeline/services/model_route_snapshot.py` |
| File | `backend/tests/unit/pipeline/test_xai_runtime_truth.py` |
| File | `tools/prod/prod_generate_xai_cycle015_0_figures.py` |
| File | `backend/tests/unit/pipeline/test_prod_generate_xai_cycle015_0_figures.py` |
| Doc | `docs/xai_anomaly/cycle_015_0_figure_plan.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |

## Ownership claim

- generator path: `tools/prod/prod_generate_xai_cycle015_0_figures.py`
- manifest validation path:
  `backend/tests/unit/pipeline/test_prod_generate_xai_cycle015_0_figures.py`
- current local responsibility: generate the planned route/BSIL/vocabulary/
  rollback figures from production-helper artifacts with explicit unavailable
  reasons

## Current implementation lane

The local implementation lane now includes:

- immutable route snapshot payloads now exist as a dedicated value contract;
- route-service mutation is isolated per instance rather than shared globally;
- snapshot digests and override metadata are unit-tested; and
- the generator produces the five Figure Planner targets, input/output digests,
  Markdown embeds, and unavailable-metric disclosures.

Production-generated images remain pending the native Linux RTX 5090 benchmark
and rollback artifacts. The local generator and manifest tests do not create a
Cycle 015.0 decision.
