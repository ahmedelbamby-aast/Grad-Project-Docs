# Cycle 015.4 Figure Implementation Claim

**Figure Implementer:** Claude Cycle 015.4 Figure Implementer (claimed 2026-06-11)

Owns generator `tools/prod/prod_generate_xai_cycle015_4_figures.py` (paired with
the T099 collector), its tests, the MANIFEST.json digests, and the embeds into
`docs/xai_anomaly/cycle_015_4_results.md`. The Figure Planner role (T089) is held
separately and the plan is fixed before any figure code is written; benchmark
evidence is kept separate from the figure artifacts. Bounded-state and
counterfactual figures are data-driven from the temporal/soak probe (T098) and
benchmark collector; any unavailable series renders a labeled placeholder rather
than a fabricated zero.
