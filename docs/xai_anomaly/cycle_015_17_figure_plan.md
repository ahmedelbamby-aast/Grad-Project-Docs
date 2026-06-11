# Cycle 015.17 Figure Plan

**Last updated:** 2026-06-11
**Status:** `planned_pending_production_inputs`
**Figure Planner:** `Codex Cycle 015.17 Figure Planner`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `tools/prod/prod_generate_xai_cycle015_17_figures.py` |
| File | `tools/prod/prod_run_xai_cycle015_17.sh` |
| File | `tools/prod/prod_watch_benchmark_metrics.sh` |
| Doc | `specs/015-xai-anomaly-score/contracts/benchmark-evidence-contract.md` |
| Doc | `docs/xai_anomaly/cycle_015_throughput_remediation_investigation.md` |

## Required Figures

| Figure | Raw inputs | Unavailable policy | Embed target |
|---|---|---|---|
| Throughput comparison | Baseline/candidate metrics JSON | Render unavailable with reason | Cycle 015.17 results |
| Correctness parity | PostgreSQL row counters from both runs | Never substitute zero | Cycle 015.17 results |
| Persistence lane evidence | Redis stats for `db_rows` and `embedding` | Render missing lane/reason | Cycle 015.17 results |
| Model RTT and call rate | Per-model benchmark metrics | Render unavailable with reason | Cycle 015.17 results |
| Unavailable summary | Generator unavailable ledger | Always emit | Cycle 015.17 results |

Every figure must be generated from the same raw artifacts used by the
decision table. The manifest records each input path and SHA-256 digest.

## Role Separation

No second agent was active for this cycle. One Codex agent holds both named
roles, but planning evidence in this file remains separate from generator,
test, manifest, and image evidence.
