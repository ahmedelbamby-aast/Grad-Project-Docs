# Cycle 015.7 Figure Implementation

**Last updated:** 2026-06-10
**Cycle:** `015.7`
**Role:** `Figure Implementer`
**Owner:** `Codex local Cycle 015.7 session`
**Status:** `active`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `backend/apps/anomalies/scoring/fusion.py` |
| File | `backend/apps/behavior/explainability/graph.py` |
| File | `backend/apps/behavior/explainability/composer.py` |
| File | `backend/tests/unit/behavior/test_xai_explanation_graph.py` |
| File | `backend/tests/integration/behavior/test_xai_context_identity.py` |
| Doc | `docs/xai_anomaly/cycle_015_7_figure_plan.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |

## Implemented local lane

This turn owns the local implementation slice only:

- graph-ready contradiction/context payloads exist;
- deterministic explanation graph generation exists;
- explanation records persist and read through the behavior API; and
- local unit/integration validation proves unresolved identity remains visible.

Production figure generation remains pending until the Cycle 015.7 probe and
benchmark helpers exist.
