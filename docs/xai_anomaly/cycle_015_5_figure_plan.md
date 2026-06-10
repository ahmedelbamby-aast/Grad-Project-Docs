# Cycle 015.5 Figure Plan

**Last updated:** 2026-06-10
**Cycle:** `015.5`
**Role:** `Figure Planner`
**Owner:** `Codex local Cycle 015.5 session`
**Status:** `active`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `tools/prod/prod_probe_xai_scores.py` |
| File | `tools/prod/prod_generate_xai_cycle015_5_figures.py` |
| File | `backend/apps/anomalies/management/commands/evaluate_xai_scores.py` |
| File | `backend/tests/unit/anomalies/test_prod_probe_xai_scores.py` |
| File | `backend/tests/unit/anomalies/test_prod_generate_xai_cycle015_5_figures.py` |
| Commit | `7e8e870e` |
| Doc | `specs/015-xai-anomaly-score/contracts/benchmark-evidence-contract.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |
| Doc | `docs/xai_anomaly/cycle_015_5_investigation.md` |

## Planner decision

Cycle 015.5 figures must consume the same raw score/probe artifacts used by the
decision table. Until a production benchmark exists, the figures are
probe/evidence artifacts only and must not be presented as acceptance evidence.

## Required raw inputs

| Input | Producer | Required for |
|---|---|---|
| score payload JSON (`{"records":[...]}`) | score repository/read path or exported score artifact | every figure |
| score probe JSON | `tools/prod/prod_probe_xai_scores.py` | invariant summary, decision status |
| optional reported-metrics block | bundled with score payload JSON | prohibited-metric verification context |

## Required figures

| Figure | Purpose | Raw fields consumed |
|---|---|---|
| `pattern_state_distribution` | state mix across `pattern_deviation`, `within_observed_pattern`, `withheld`, `insufficient_context` | `pattern_state` |
| `score_distribution` | numeric score density for non-withheld results | `review_priority_score` |
| `contribution_decomposition` | average effective contribution per component | `contributions[].component_key`, `effective_contribution` |
| `deviation_envelope` | relationship between deviation magnitude and score | `deviation_vs_self`, `review_priority_score`, `pattern_state` |
| `coverage_vs_score` | score behavior under evidence-coverage changes | `evidence_coverage`, `review_priority_score` |
| `contamination_quarantine` | withheld-reason distribution for cold-start/quarantine/coverage failures | `withheld_reasons` |
| `invariant_summary` | reconstruction/invariant/prohibited-metric failure counts from probe | probe `evaluation.*` |

## Unavailable-metric policy

- Missing numeric score values remain omitted from score/deviation/coverage
  plots; they are not coerced to zero.
- If no valid contributions exist, the contribution chart renders a “No valid
  contributions” placeholder.
- If the probe JSON is absent, invariant-summary bars render from zero and the
  figure must be treated as partial, not complete production evidence.
- Any future production wrapper must write unavailable reasons into the figure
  manifest rather than omitting the figure family silently.

## Markdown embed targets

- `docs/xai_anomaly/cycle_015_5_results.md` once a real benchmark exists
- any generated `figures.md` sidecar emitted beside the figure manifest during
  local/probe generation

## Role-separation note

Only one active agent is available in the current session, so the Figure
Planner and Figure Implementer roles cannot be separated operationally. The
planner output here remains separate from the implementation evidence in
`cycle_015_5_figure_implementation.md`.
