# Cycle 015.17 Figure Implementation

**Last updated:** 2026-06-11
**Status:** `implemented_pending_production_inputs`
**Figure Implementer:** `Codex Cycle 015.17 Figure Implementer`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `tools/prod/prod_generate_xai_cycle015_17_figures.py` |
| File | `backend/tests/unit/scripts/test_prod_generate_xai_cycle015_17_figures.py` |
| Doc | `docs/xai_anomaly/cycle_015_17_figure_plan.md` |
| Workflow | `.github/workflows/inference-parallelization.yml` |

## Implementation Contract

The generator emits five PNG files, `FIGURES.md`, and a digest-addressed
`MANIFEST.json`. It accepts baseline/candidate metrics and a lane-stat payload
covering both `db_rows` and `embedding`. Missing metrics produce explicit
unavailable figures and reasons rather than absent files or zero values.

The generator is locally covered by
`backend/tests/unit/scripts/test_prod_generate_xai_cycle015_17_figures.py`.
Production images cannot be generated until the native RTX 5090
baseline/candidate artifacts exist.
