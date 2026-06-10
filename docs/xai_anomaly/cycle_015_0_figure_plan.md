# Cycle 015.0 Figure Plan

**Last updated:** 2026-06-10
**Cycle:** `015.0`
**Role:** `Figure Planner`
**Owner:** `Codex local Cycle 015.0 session`
**Status:** `active`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `backend/apps/pipeline/services/model_route_snapshot.py` |
| File | `backend/tests/unit/pipeline/test_xai_runtime_truth.py` |
| Doc | `specs/015-xai-anomaly-score/contracts/benchmark-evidence-contract.md` |
| Doc | `docs/xai_anomaly/cycle_015_0_investigation.md` |

## Planner decision

Cycle 015.0 figures must prove two things once production evidence exists:

- resolved route truth is explicit and stable across the candidate run; and
- BSIL/lineage rows become nonzero without regressing source outputs.

This turn does not implement the generator; it defines the future raw inputs and
figure targets so later production helpers have a governed contract.

## Required raw inputs

| Input | Producer | Required for |
|---|---|---|
| route snapshot JSON | future `prod_xai_cycle015_0_preflight.py` | route truth tables and digest evidence |
| benchmark metrics JSON | future `prod_run_xai_cycle015_0.sh` | FPS/latency/resource plots |
| BSIL row-count summary JSON | future preflight or collector | activation delta plots |
| rollback status JSON | future `prod_rollback_xai_cycle015_0.sh` | rollback verification plot |

## Required figures

| Figure | Purpose | Raw fields consumed |
|---|---|---|
| `route_snapshot_matrix` | active task-to-model truth at benchmark time | task key, model name, version, source, digest |
| `route_digest_stability` | prove candidate run uses one coherent route truth | route digest, environment digest |
| `bsil_activation_delta` | show nonzero lineage/semantic activation | baseline vs candidate row counts |
| `runtime_truth_vocabulary` | show no prohibited wording leaked through runtime summaries | summary text, reason codes |
| `rollback_verification` | prove route/profile state converged after rollback | rollback flags and reconciled digests |

## Unavailable-metric policy

- Missing production row counts must render as `unavailable` with a reason; do
  not plot them as zero.
- If no benchmark helper has yet written route snapshot JSON, only local
  contract figures may exist and must be labeled probe-only.
- The final manifest must link figures back to the exact raw route/benchmark
  artifacts used by the Cycle 015.0 decision table.
