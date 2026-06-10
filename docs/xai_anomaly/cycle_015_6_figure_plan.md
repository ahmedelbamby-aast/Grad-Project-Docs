# Cycle 015.6 Figure Plan

**Last updated:** 2026-06-10
**Cycle:** `015.6`
**Role:** `Figure Planner`
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
| Doc | `docs/xai_anomaly/cycle_015_6_investigation.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |

## Required raw inputs

| Input | Producer | Required fields |
|---|---|---|
| score payload JSON | `backend/apps/anomalies/scoring/service.py` | `evidence_coverage`, `uncertainty`, `withheld_reasons`, `contradictions.conformal` |
| conformal snapshot JSON | `backend/apps/anomalies/scoring/conformal.py` | target coverage, empirical coverage, drift score, sample count, truth state |
| conformal evaluation JSON | `backend/apps/anomalies/scoring/conformal_evaluation.py` | empirical coverage, average interval width, reason codes |

## Required figures

| Figure key | Purpose |
|---|---|
| `coverage_vs_target` | compare empirical versus target coverage |
| `drift_vs_eligibility` | show when drift suppresses conformal eligibility |
| `uncertainty_vs_missingness` | show uncertainty growth under missing features and invalid windows |
| `withholding_reasons` | summarize conformal and uncertainty-related veto reasons |

## Markdown targets

- `docs/xai_anomaly/cycle_015_6_results.md`
- `docs/BENCHMARK_RESULTS_LEDGER.md` once production evidence exists
