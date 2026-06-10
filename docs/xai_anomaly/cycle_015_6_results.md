# Cycle 015.6 Results

**Last updated:** 2026-06-10
**Cycle:** `015.6`
**Status:** `staged_local_only`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `tools/prod/prod_probe_xai_conformal.py` |
| File | `tools/prod/prod_generate_xai_cycle015_6_figures.py` |
| File | `tools/prod/prod_run_xai_cycle015_6.sh` |
| File | `tools/prod/prod_rollback_xai_cycle015_6.sh` |
| File | `backend/tests/unit/anomalies/test_prod_probe_xai_conformal.py` |
| File | `backend/tests/unit/anomalies/test_prod_generate_xai_cycle015_6_figures.py` |
| File | `backend/tests/unit/anomalies/test_prod_run_xai_cycle015_6_shell_syntax.py` |
| File | `backend/tests/unit/anomalies/test_prod_rollback_xai_cycle015_6_shell_syntax.py` |
| File | `backend/tests/unit/anomalies/test_xai_conformal_uncertainty.py` |
| File | `backend/tests/integration/anomalies/test_xai_conformal_runtime.py` |
| Doc | `docs/xai_anomaly/cycle_015_6_investigation.md` |
| Doc | `docs/xai_anomaly/cycle_015_6_figure_plan.md` |
| Doc | `docs/xai_anomaly/cycle_015_6_figure_implementation.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |

## Current decision state

Cycle 015.6 is **not yet a production benchmark decision**. The current state
is `staged_local_only` because:

- uncertainty propagation, conformal snapshot gating, and distributional-only
  coverage evaluation are implemented;
- the file-driven probe, figure generator, run wrapper, and rollback helper now
  exist; and
- local unit/integration validation for the conformal slice passes.

Still missing before a production decision can be recorded:

- a real stride-1 production run using `tools/prod/prod_run_xai_cycle015_6.sh`
  against authoritative conformal artifacts;
- rollback execution output from
  `tools/prod/prod_rollback_xai_cycle015_6.sh`;
- a production results table with real artifact paths; and
- a `docs/BENCHMARK_RESULTS_LEDGER.md` entry for that run.

## Verified local evidence

Local evidence currently available:

```text
.\.venv\Scripts\python.exe -m pytest backend/tests/unit/anomalies/test_xai_conformal_uncertainty.py backend/tests/integration/anomalies/test_xai_conformal_runtime.py backend/tests/unit/anomalies/test_xai_pattern_scoring_primitives.py backend/tests/unit/anomalies/test_xai_review_priority_score.py
# 31 passed

.\.venv\Scripts\python.exe -m pytest backend/tests/unit/anomalies/test_prod_probe_xai_conformal.py backend/tests/unit/anomalies/test_prod_generate_xai_cycle015_6_figures.py backend/tests/unit/anomalies/test_prod_run_xai_cycle015_6_shell_syntax.py backend/tests/unit/anomalies/test_prod_rollback_xai_cycle015_6_shell_syntax.py
# 10 passed, 2 skipped

.\.venv\Scripts\python.exe -m py_compile tools/prod/prod_probe_xai_conformal.py tools/prod/prod_generate_xai_cycle015_6_figures.py backend/tests/unit/anomalies/test_prod_probe_xai_conformal.py backend/tests/unit/anomalies/test_prod_generate_xai_cycle015_6_figures.py
# success

.\.venv\Scripts\python.exe manage.py check
# System check identified no issues (0 silenced).
```

These runs establish local readiness only. They do not constitute production
acceptance under the constitution.

## Rollback boundary

Cycle 015.6 conformal tooling is file-driven only. The rollback helper
`tools/prod/prod_rollback_xai_cycle015_6.sh` exists and emits
`rollback_status.json` / `.md`, confirming that:

- runtime state is not mutated;
- database state is not mutated;
- Redis state is not mutated;
- Triton state is not mutated; and
- only local evidence artifacts are optionally cleaned up.

Production rollback proof is still pending because no production run has yet
been executed.
