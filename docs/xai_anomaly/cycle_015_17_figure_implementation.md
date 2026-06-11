# Cycle 015.17 Figure Implementation

**Last updated:** 2026-06-11
**Status:** `completed`
**Figure Implementer:** `Codex Cycle 015.17 Figure Implementer`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `tools/prod/prod_generate_xai_cycle015_17_figures.py` |
| File | `backend/tests/unit/scripts/test_prod_generate_xai_cycle015_17_figures.py` |
| Doc | `docs/xai_anomaly/cycle_015_17_figure_plan.md` |
| Doc | `docs/xai_anomaly/cycle_015_17_results.md` |
| Artifact | `docs/figures/benchmark_artifacts/cycle015-17-prod-20260611/figures/MANIFEST.json` |
| Artifact | `docs/figures/benchmark_artifacts/cycle015-17-prod-r2-20260611/figures/MANIFEST.json` |
| Workflow | `.github/workflows/inference-parallelization.yml` |

## Implementation Contract

The generator emits five PNG files, `FIGURES.md`, and a digest-addressed
`MANIFEST.json`. It accepts baseline/candidate metrics and a lane-stat payload
covering both `db_rows` and `embedding`. Missing metrics produce explicit
unavailable figures and reasons rather than absent files or zero values.

The generator is locally covered by
`backend/tests/unit/scripts/test_prod_generate_xai_cycle015_17_figures.py`.
Both production attempts generated five PNG files, `FIGURES.md`, and
`MANIFEST.json`. The r2 manifest records the three decision input digests and
the five output image digests. Its unavailable ledger explicitly records the
missing generator key rather than plotting it as zero.
