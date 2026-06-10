# Cycle 015.5 Figure Implementation

**Last updated:** 2026-06-10
**Cycle:** `015.5`
**Role:** `Figure Implementer`
**Owner:** `Codex local Cycle 015.5 session`
**Status:** `active`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `tools/prod/prod_generate_xai_cycle015_5_figures.py` |
| File | `tools/prod/prod_probe_xai_scores.py` |
| File | `backend/tests/unit/anomalies/test_prod_generate_xai_cycle015_5_figures.py` |
| File | `backend/tests/unit/anomalies/test_prod_probe_xai_scores.py` |
| File | `backend/tests/unit/anomalies/test_evaluate_xai_scores_command.py` |
| Commit | `7e8e870e` |
| Doc | `docs/xai_anomaly/cycle_015_5_figure_plan.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |

## Implemented artifacts

| Artifact | Role |
|---|---|
| `tools/prod/prod_probe_xai_scores.py` | file-driven probe helper wrapping score evaluation/reconstruction into a `PROBE_ONLY` report |
| `tools/prod/prod_generate_xai_cycle015_5_figures.py` | file-driven figure generator for score and probe artifacts |
| `backend/tests/unit/anomalies/test_prod_probe_xai_scores.py` | unit coverage for probe summary and output writing |
| `backend/tests/unit/anomalies/test_prod_generate_xai_cycle015_5_figures.py` | unit coverage for generated PNGs, manifest, and markdown |

## Generated output contract

The figure generator currently writes:

- `cycle_015_5__pattern_state_distribution.png`
- `cycle_015_5__score_distribution.png`
- `cycle_015_5__contribution_decomposition.png`
- `cycle_015_5__deviation_envelope.png`
- `cycle_015_5__coverage_vs_score.png`
- `cycle_015_5__contamination_quarantine.png`
- `cycle_015_5__invariant_summary.png`
- `figure_manifest.json`
- `cycle_015_5_figures.md` (or caller-supplied markdown output path)

The manifest records input digests, figure digests, cycle id, decision string,
and pattern-state counts. The current schema is
`xai.figure_manifest.v1`.

## Verification performed

DB-free verification passed locally against the current worktree:

```text
.\.venv\Scripts\python.exe -m pytest backend/tests/unit/anomalies/test_prod_run_xai_cycle015_5_shell_syntax.py backend/tests/unit/anomalies/test_prod_rollback_xai_cycle015_5_shell_syntax.py backend/tests/unit/anomalies/test_prod_generate_xai_cycle015_5_figures.py backend/tests/unit/anomalies/test_prod_probe_xai_scores.py backend/tests/unit/anomalies/test_evaluate_xai_scores_command.py backend/tests/unit/anomalies/test_xai_review_priority_score.py backend/tests/unit/anomalies/test_xai_pattern_scoring_primitives.py backend/tests/unit/anomalies/test_xai_review_priority_reconstruction.py backend/tests/unit/anomalies/test_xai_score_persistence_repository.py backend/tests/unit/anomalies/test_xai_review_priority_preview_service.py backend/tests/unit/anomalies/test_xai_review_priority_serializers.py backend/tests/unit/anomalies/test_xai_review_priority_contracts.py
# 73 passed, 2 skipped

.\.venv\Scripts\python.exe -m py_compile tools/prod/prod_probe_xai_scores.py tools/prod/prod_generate_xai_cycle015_5_figures.py backend/tests/unit/anomalies/test_prod_probe_xai_scores.py backend/tests/unit/anomalies/test_prod_generate_xai_cycle015_5_figures.py
# success
```

## Role-separation note

No separate agent was available for figure implementation in this session, so
the planner and implementer owners are the same. Separation is preserved in
documentation only: planning intent lives in `cycle_015_5_figure_plan.md`,
while generator/test evidence is recorded here.
