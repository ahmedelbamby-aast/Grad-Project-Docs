# Cycle 015.5 Results

**Last updated:** 2026-06-10
**Cycle:** `015.5`
**Status:** `staged_local_only`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `tools/prod/prod_run_xai_cycle015_5.sh` |
| File | `tools/prod/prod_probe_xai_scores.py` |
| File | `tools/prod/prod_generate_xai_cycle015_5_figures.py` |
| File | `tools/prod/prod_rollback_xai_cycle015_5.sh` |
| File | `backend/tests/integration/anomalies/test_xai_score_persistence.py` |
| File | `backend/tests/unit/anomalies/test_prod_run_xai_cycle015_5_shell_syntax.py` |
| File | `backend/tests/unit/anomalies/test_prod_rollback_xai_cycle015_5_shell_syntax.py` |
| Commit | `7e8e870e` |
| Doc | `docs/xai_anomaly/cycle_015_5_investigation.md` |
| Doc | `docs/xai_anomaly/cycle_015_5_figure_plan.md` |
| Doc | `docs/xai_anomaly/cycle_015_5_figure_implementation.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |

## Current decision state

Cycle 015.5 is **not yet a production benchmark decision**. The current state
is `staged_local_only` because:

- the score contracts, scoring logic, persistence layer, read path, probe, and
  figure generator are implemented;
- DB-free unit coverage for the full score slice passes; and
- PostgreSQL-backed integration tests now pass locally.

Still missing before a production decision can be recorded:

- a real stride-1 production run using `tools/prod/prod_run_xai_cycle015_5.sh`
  against authoritative score artifacts;
- rollback execution output from
  `tools/prod/prod_rollback_xai_cycle015_5.sh`;
- a production results table with real artifact paths; and
- a `docs/BENCHMARK_RESULTS_LEDGER.md` entry for that run.

## Verified local evidence

Local evidence currently available:

```text
.\.venv\Scripts\python.exe -m pytest tests/integration/anomalies/test_xai_score_persistence.py tests/contract/test_xai_review_priority_api_contract.py tests/unit/anomalies/test_views.py
# 25 passed

.\.venv\Scripts\python.exe -m pytest backend/tests/unit/anomalies/test_prod_run_xai_cycle015_5_shell_syntax.py backend/tests/unit/anomalies/test_prod_rollback_xai_cycle015_5_shell_syntax.py backend/tests/unit/anomalies/test_prod_generate_xai_cycle015_5_figures.py backend/tests/unit/anomalies/test_prod_probe_xai_scores.py backend/tests/unit/anomalies/test_evaluate_xai_scores_command.py backend/tests/unit/anomalies/test_xai_review_priority_score.py backend/tests/unit/anomalies/test_xai_pattern_scoring_primitives.py backend/tests/unit/anomalies/test_xai_review_priority_reconstruction.py backend/tests/unit/anomalies/test_xai_score_persistence_repository.py backend/tests/unit/anomalies/test_xai_review_priority_preview_service.py backend/tests/unit/anomalies/test_xai_review_priority_serializers.py backend/tests/unit/anomalies/test_xai_review_priority_contracts.py
# 73 passed, 2 skipped

.\.venv\Scripts\python.exe -m pytest backend/tests/contract/test_dev_postgres_compose_contract.py backend/tests/unit/scripts/test_recover_dev_postgres_ps1.py
# 6 passed

.\.venv\Scripts\python.exe manage.py check
# System check identified no issues (0 silenced).
```

These runs establish local contract/integration readiness only. They do not
constitute production acceptance under the constitution.

## Rollback boundary

Cycle 015.5 score tooling is file-driven only. The rollback helper
`tools/prod/prod_rollback_xai_cycle015_5.sh` exists and emits
`rollback_status.json` / `.md`, confirming that:

- runtime state is not mutated;
- database state is not mutated;
- Redis state is not mutated;
- Triton state is not mutated; and
- only local evidence artifacts are optionally cleaned up.

Production rollback proof is still pending because no production run has yet
been executed.
