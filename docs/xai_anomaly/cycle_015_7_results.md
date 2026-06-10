# Cycle 015.7 Results

**Last updated:** 2026-06-10
**Cycle:** `015.7`
**Status:** `staged_local_only`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `tools/prod/prod_probe_xai_fusion.py` |
| File | `tools/prod/prod_generate_xai_cycle015_7_figures.py` |
| File | `tools/prod/prod_run_xai_cycle015_7.sh` |
| File | `tools/prod/prod_rollback_xai_cycle015_7.sh` |
| File | `backend/tests/unit/behavior/test_prod_probe_xai_fusion.py` |
| File | `backend/tests/unit/behavior/test_prod_generate_xai_cycle015_7_figures.py` |
| File | `backend/tests/unit/behavior/test_prod_run_xai_cycle015_7_shell_syntax.py` |
| File | `backend/tests/unit/behavior/test_prod_rollback_xai_cycle015_7_shell_syntax.py` |
| File | `backend/tests/unit/behavior/test_xai_explanation_graph.py` |
| File | `backend/tests/integration/behavior/test_xai_context_identity.py` |
| Doc | `docs/xai_anomaly/cycle_015_7_investigation.md` |
| Doc | `docs/xai_anomaly/cycle_015_7_figure_plan.md` |
| Doc | `docs/xai_anomaly/cycle_015_7_figure_implementation.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |

## Current decision state

Cycle 015.7 is **not yet a production benchmark decision**. The current state
is `staged_local_only` because:

- source-preserving fusion, identity-gated context, explanation-graph building,
  and explanation-record persistence are implemented;
- the file-driven probe, figure generator, run wrapper, and rollback helper now
  exist; and
- local unit/integration validation for the fusion slice passes.

Still missing before a production decision can be recorded:

- a real stride-1 production run using `tools/prod/prod_run_xai_cycle015_7.sh`
  against authoritative fusion artifacts;
- rollback execution output from
  `tools/prod/prod_rollback_xai_cycle015_7.sh`;
- a production results table with real artifact paths; and
- a `docs/BENCHMARK_RESULTS_LEDGER.md` entry for that run.

## Verified local evidence

Local evidence currently available:

```text
.\.venv\Scripts\python.exe -m pytest backend/tests/unit/behavior/test_xai_explanation_graph.py backend/tests/integration/behavior/test_xai_context_identity.py backend/tests/unit/behavior/test_interaction_identity_gates.py backend/tests/integration/behavior/test_interaction_graph_replay.py
# 10 passed

.\.venv\Scripts\python.exe -m pytest backend/tests/unit/behavior/test_prod_probe_xai_fusion.py backend/tests/unit/behavior/test_prod_generate_xai_cycle015_7_figures.py backend/tests/unit/behavior/test_prod_run_xai_cycle015_7_shell_syntax.py backend/tests/unit/behavior/test_prod_rollback_xai_cycle015_7_shell_syntax.py
# 10 passed, 2 skipped

.\.venv\Scripts\python.exe -m py_compile backend/apps/behavior/explainability/contradictions.py backend/apps/behavior/explainability/context.py backend/apps/behavior/explainability/graph.py backend/apps/behavior/explainability/composer.py backend/tests/unit/behavior/test_xai_explanation_graph.py backend/tests/integration/behavior/test_xai_context_identity.py
# success

.\.venv\Scripts\python.exe -m py_compile tools/prod/prod_probe_xai_fusion.py tools/prod/prod_generate_xai_cycle015_7_figures.py backend/tests/unit/behavior/test_prod_probe_xai_fusion.py backend/tests/unit/behavior/test_prod_generate_xai_cycle015_7_figures.py
# success

.\.venv\Scripts\python.exe manage.py check
# System check identified no issues (0 silenced).
```

These runs establish local readiness only. They do not constitute production
acceptance under the constitution.

## Rollback boundary

Cycle 015.7 fusion tooling is file-driven only. The rollback helper
`tools/prod/prod_rollback_xai_cycle015_7.sh` exists and emits
`rollback_status.json` / `.md`, confirming that:

- runtime state is not mutated;
- database state is not mutated;
- Redis state is not mutated;
- Triton state is not mutated; and
- only local evidence artifacts are optionally cleaned up.

Production rollback proof is still pending because no production run has yet
been executed.
