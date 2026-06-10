# Cycle 015.6 Investigation — Missingness, Uncertainty, And Conformal Calibration

**Last updated:** 2026-06-10
**Cycle:** `015.6`
**Status:** `staged_local_only`
**Streaming compatibility:** `stream-safe`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `backend/apps/anomalies/scoring/uncertainty.py` |
| File | `backend/apps/anomalies/scoring/conformal.py` |
| File | `backend/apps/anomalies/scoring/conformal_evaluation.py` |
| File | `backend/apps/anomalies/scoring/service.py` |
| File | `backend/apps/anomalies/models.py` |
| File | `backend/tests/unit/anomalies/test_xai_conformal_uncertainty.py` |
| File | `backend/tests/integration/anomalies/test_xai_conformal_runtime.py` |
| Doc | `specs/015-xai-anomaly-score/spec.md` |
| Doc | `specs/015-xai-anomaly-score/plan.md` |
| Doc | `specs/015-xai-anomaly-score/data-model.md` |
| Doc | `specs/015-xai-anomaly-score/research.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |

## Goal

Cycle 015.6 extends the review-priority layer with explicit missingness and
uncertainty propagation plus valid conformal gating, without creating false
behavioral precision or behavioral-ground-truth claims.

## Conformal assumption protocol

Cycle 015.6 keeps conformal output on a strict distributional-only contract:

- conformal state may describe empirical coverage over numeric intervals or
  sets only;
- conformal state may never be described as behavioral correctness, anomaly
  truth, cheating probability, or human-intent evidence;
- outputs are withheld when calibration sample count, route/schema
  compatibility, expiry, empirical coverage, or drift gates fail; and
- adaptive conformal support is allowed only when the stored snapshot records
  the adaptation mode and the drift/coverage evidence used to validate it.

## Distributional-only boundary

The only allowed coverage language in this cycle is distributional:

- target coverage;
- empirical coverage;
- interval or set size;
- drift score;
- calibration sample count; and
- explicit distributional-only notice.

Forbidden authority in this cycle:

- behavioral accuracy;
- anomaly accuracy;
- cheating-detection quality;
- AUROC/AUPRC/F1/precision/recall claims; and
- any statement that conformal coverage proves behavior validity.

## Local implementation boundary

This cycle is complete only when:

- uncertainty propagation is applied before score fusion;
- conformal gates can veto stale, under-covered, or drifted snapshots;
- a PostgreSQL snapshot model persists the governed conformal state; and
- unit/integration validation proves withholding and idempotent persistence.

Production probe, benchmark, rollback, figures, and ledger work remain separate
tasks after the local slice.
