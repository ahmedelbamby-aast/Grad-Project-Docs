# Cycle 015.7 Figure Plan

**Last updated:** 2026-06-10
**Cycle:** `015.7`
**Role:** `Figure Planner`
**Owner:** `Codex local Cycle 015.7 session`
**Status:** `active`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `backend/apps/anomalies/scoring/fusion.py` |
| File | `backend/apps/behavior/explainability/graph.py` |
| File | `backend/apps/behavior/explainability/context.py` |
| File | `backend/apps/behavior/explainability/contradictions.py` |
| File | `backend/tests/unit/behavior/test_xai_explanation_graph.py` |
| File | `backend/tests/integration/behavior/test_xai_context_identity.py` |
| Doc | `docs/xai_anomaly/cycle_015_7_investigation.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |

## Required raw inputs

| Input | Producer | Required fields |
|---|---|---|
| fused score payload | `backend/apps/anomalies/scoring/fusion.py` | source-preserving support/missing/degraded source rows |
| explanation graph payload | `backend/apps/behavior/explainability/graph.py` | nodes, edges, unresolved identity refs, digest |
| explanation record payload | `backend/apps/behavior/explainability/composer.py` | summary text/code, supporting/contradicting/missing factors |

## Required figures

| Figure key | Purpose |
|---|---|
| `explanation_graph` | visualize node/edge flow for one representative explanation |
| `contradiction_matrix` | summarize contradiction kinds by source pair |
| `identity_gate_summary` | show valid versus unresolved peer-context rows |
| `support_flow` | show supporting, contradicting, and missing factor counts |

## Markdown targets

- `docs/xai_anomaly/cycle_015_7_results.md`
- `docs/BENCHMARK_RESULTS_LEDGER.md` once production evidence exists
