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

## Implemented artifacts

| Artifact | Role |
|---|---|
| `tools/prod/prod_probe_xai_fusion.py` | file-driven probe helper for graph digest, contradiction counts, unresolved identity refs, and non-accusatory summary checks |
| `tools/prod/prod_generate_xai_cycle015_7_figures.py` | file-driven figure generator for explanation-graph, contradiction, identity-gate, and support-flow artifacts |
| `backend/tests/unit/behavior/test_prod_probe_xai_fusion.py` | unit coverage for probe summary and markdown output |
| `backend/tests/unit/behavior/test_prod_generate_xai_cycle015_7_figures.py` | unit coverage for generated PNGs, manifest, and markdown |

## Verification performed

```text
.\.venv\Scripts\python.exe -m pytest backend/tests/unit/behavior/test_prod_probe_xai_fusion.py backend/tests/unit/behavior/test_prod_generate_xai_cycle015_7_figures.py backend/tests/unit/behavior/test_prod_run_xai_cycle015_7_shell_syntax.py backend/tests/unit/behavior/test_prod_rollback_xai_cycle015_7_shell_syntax.py
# 10 passed, 2 skipped

.\.venv\Scripts\python.exe -m py_compile tools/prod/prod_probe_xai_fusion.py tools/prod/prod_generate_xai_cycle015_7_figures.py backend/tests/unit/behavior/test_prod_probe_xai_fusion.py backend/tests/unit/behavior/test_prod_generate_xai_cycle015_7_figures.py
# success
```
