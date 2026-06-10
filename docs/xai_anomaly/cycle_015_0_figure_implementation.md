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
| Doc | `docs/xai_anomaly/cycle_015_0_figure_plan.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |

## Ownership claim

- planned generator path: `tools/prod/prod_generate_xai_cycle015_0_figures.py`
- planned validation path: `backend/tests/unit/pipeline/test_xai_runtime_truth.py`
- current local responsibility: preserve the route-snapshot payload contract so
  later production helpers can serialize stable raw inputs

## Current implementation lane

The current local implementation lane is intentionally narrow:

- immutable route snapshot payloads now exist as a dedicated value contract;
- route-service mutation is isolated per instance rather than shared globally;
- snapshot digests and override metadata are unit-tested; and
- legacy rule-engine wording is checked against non-accusatory expectations.

The actual figure generator, manifest, and production artifact tests remain
future Cycle 015.0 tasks once the benchmark/preflight helpers exist.
