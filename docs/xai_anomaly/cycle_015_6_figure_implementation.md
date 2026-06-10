# Cycle 015.6 Figure Implementation

**Last updated:** 2026-06-10
**Cycle:** `015.6`
**Role:** `Figure Implementer`
**Owner:** `Codex local Cycle 015.6 session`
**Status:** `active`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `backend/apps/anomalies/scoring/uncertainty.py` |
| File | `backend/apps/anomalies/scoring/conformal.py` |
| File | `backend/apps/anomalies/scoring/conformal_evaluation.py` |
| File | `backend/tests/unit/anomalies/test_xai_conformal_uncertainty.py` |
| File | `backend/tests/integration/anomalies/test_xai_conformal_runtime.py` |
| Doc | `docs/xai_anomaly/cycle_015_6_figure_plan.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |

## Implemented artifacts

| Artifact | Role |
|---|---|
| `tools/prod/prod_probe_xai_conformal.py` | file-driven probe helper for conformal gate usability, drift, coverage, and score-withholding summaries |
| `tools/prod/prod_generate_xai_cycle015_6_figures.py` | file-driven figure generator for coverage, drift, uncertainty, and withholding artifacts |
| `backend/tests/unit/anomalies/test_prod_probe_xai_conformal.py` | unit coverage for probe summary and markdown output |
| `backend/tests/unit/anomalies/test_prod_generate_xai_cycle015_6_figures.py` | unit coverage for generated PNGs, manifest, and markdown |

## Generated output contract

The figure generator currently writes:

- `cycle_015_6__coverage_vs_target.png`
- `cycle_015_6__drift_vs_eligibility.png`
- `cycle_015_6__uncertainty_vs_missingness.png`
- `cycle_015_6__withholding_reasons.png`
- `figure_manifest.json`
- `cycle_015_6_figures.md` (or caller-supplied markdown output path)

## Verification performed

Local focused validation passed against the current worktree:

```text
.\.venv\Scripts\python.exe -m pytest backend/tests/unit/anomalies/test_prod_probe_xai_conformal.py backend/tests/unit/anomalies/test_prod_generate_xai_cycle015_6_figures.py backend/tests/unit/anomalies/test_prod_run_xai_cycle015_6_shell_syntax.py backend/tests/unit/anomalies/test_prod_rollback_xai_cycle015_6_shell_syntax.py
# 10 passed, 2 skipped

.\.venv\Scripts\python.exe -m py_compile tools/prod/prod_probe_xai_conformal.py tools/prod/prod_generate_xai_cycle015_6_figures.py backend/tests/unit/anomalies/test_prod_probe_xai_conformal.py backend/tests/unit/anomalies/test_prod_generate_xai_cycle015_6_figures.py
# success
```

Production figure generation remains pending until authoritative conformal
artifacts exist.
