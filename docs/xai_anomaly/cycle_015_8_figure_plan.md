# Cycle 015.8 Figure Plan

**Last updated:** 2026-06-10
**Cycle:** `015.8`
**Role:** `Figure Planner`
**Owner:** `Codex local Cycle 015.8 session`
**Status:** `active`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `backend/apps/behavior/explainability/deep_contracts.py` |
| File | `backend/apps/behavior/models.py` |
| File | `backend/tests/unit/behavior/test_xai_deep_methods.py` |
| Doc | `docs/xai_anomaly/cycle_015_8_investigation.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |

## Required raw inputs

| Input | Producer | Required fields |
|---|---|---|
| deep request payload | `deep_contracts.py` | method key/version, route snapshot, deadline, artifact budget, idempotency key |
| attribution artifact payload | `deep_contracts.py` + ORM model | lifecycle status, digest, artifact ref, unavailable reason |
| evaluation payload | `deep_contracts.py` + ORM model | fidelity/stability/sanity metrics, latency/resource metrics, decision |

## Required figures

| Figure key | Purpose |
|---|---|
| `method_matrix` | eligible versus ineligible method counts by source family |
| `lifecycle_flow` | request/artifact/evaluation state distribution |
| `deadline_budget` | deadline and artifact-size budget summary |
| `evaluation_decisions` | valid/degraded/invalidated/probe-only evaluation counts |

## Markdown targets

- `docs/xai_anomaly/cycle_015_8_results.md`
- `docs/BENCHMARK_RESULTS_LEDGER.md` once production evidence exists
