# Cycle 015.6 Figure Implementation

**Last updated:** 2026-06-10
**Cycle:** `015.6`
**Role:** `Figure Implementer`
**Owner:** `Codex local Cycle 015.6 session`
**Status:** `planned`

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

## Current lane ownership

This turn owns only the local implementation slice for Cycle 015.6. Production
figure generation remains out of scope until the probe/benchmark tasks land.

## Planned figure-generator responsibilities

- render empirical versus target coverage from the conformal evaluation payload;
- render drift and eligibility gates from the snapshot metadata;
- render uncertainty versus missingness from scored windows; and
- emit a manifest with input digests and explicit unavailable reasons.
