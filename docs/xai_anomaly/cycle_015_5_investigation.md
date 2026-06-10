# Cycle 015.5 Investigation — Transparent Hierarchical Anomaly Score

**Last updated:** 2026-06-10
**Cycle:** `015.5`
**Status:** `staged_local_only`
**Streaming compatibility:** `stream-safe`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `backend/apps/anomalies/scoring/contracts.py` |
| File | `backend/apps/anomalies/scoring/service.py` |
| File | `backend/apps/anomalies/scoring/reconstruction.py` |
| File | `backend/apps/anomalies/scoring/pattern_profiles.py` |
| File | `backend/apps/anomalies/scoring/pattern_comparison.py` |
| File | `backend/apps/anomalies/scoring/components.py` |
| File | `backend/apps/anomalies/scoring/fusion.py` |
| File | `backend/apps/anomalies/models.py` |
| File | `backend/apps/behavior/models.py` |
| File | `backend/apps/anomalies/services.py` |
| File | `backend/apps/anomalies/serializers.py` |
| File | `backend/apps/anomalies/views.py` |
| File | `backend/apps/anomalies/management/commands/evaluate_xai_scores.py` |
| File | `tools/prod/prod_probe_xai_scores.py` |
| File | `tools/prod/prod_generate_xai_cycle015_5_figures.py` |
| File | `backend/tests/unit/anomalies/test_xai_review_priority_score.py` |
| File | `backend/tests/unit/anomalies/test_xai_pattern_scoring_primitives.py` |
| File | `backend/tests/unit/anomalies/test_xai_review_priority_reconstruction.py` |
| File | `backend/tests/unit/anomalies/test_xai_score_persistence_repository.py` |
| File | `backend/tests/unit/anomalies/test_evaluate_xai_scores_command.py` |
| File | `backend/tests/unit/anomalies/test_prod_probe_xai_scores.py` |
| File | `backend/tests/unit/anomalies/test_prod_generate_xai_cycle015_5_figures.py` |
| File | `backend/tests/integration/anomalies/test_xai_score_persistence.py` |
| Commit | `7e8e870e` |
| Doc | `specs/015-xai-anomaly-score/spec.md` |
| Doc | `specs/015-xai-anomaly-score/plan.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |
| Doc | `specs/015-xai-anomaly-score/no-ground-truth-doctrine.md` |
| Doc | `specs/015-xai-anomaly-score/contracts/anomaly-score-contract.md` |
| Doc | `specs/015-xai-anomaly-score/contracts/benchmark-evidence-contract.md` |

## Goal

Cycle 015.5 produces the first reconstructable, non-accusatory
`review_priority_score` from bounded per-student observed signal-pattern
comparison, without anomaly-model training and without behavioral ground-truth
claims.

## Deterministic score profile

The implemented score profile remains the transparent hierarchical form defined
in `specs/015-xai-anomaly-score/plan.md`:

```text
valid_component_i =
    pattern_deviation_magnitude_i
    * reliability_i
    * temporal_support_i

weighted_valid_component_mean =
    sum(valid_component_i * configured_weight_i)
    / sum(configured_weight_i)

review_priority_score =
    100
    * clamp(weighted_valid_component_mean
            * context_support
            * persistence_support
            * (1 - uncertainty))
```

Operational consequences:

- missing or invalid contributions remain explicit and do not silently become
  zero;
- reviewer-approved deviation may lift only the separate session aggregate,
  never a peer student's individual score or pattern state; and
- every valid contribution requires parameter provenance and a counterfactual
  threshold/recovery delta.

## No-ground-truth evaluation protocol

Cycle 015.5 evaluation is limited to deterministic, reconstructable algorithm
behavior. Allowed authority:

- exact reconstruction of score and contribution rows;
- deterministic replay from payloads;
- controlled pattern fixtures;
- metamorphic and invariant checks;
- sensitivity checks over uncertainty, context support, persistence support,
  and contribution magnitude;
- contamination, cold-start, quarantine, route/schema mismatch, and withholding
  behavior; and
- probe-only production evidence helpers and figure generation.

Forbidden authority:

- anomaly accuracy;
- cheating-detection quality;
- any label-based precision/recall/F1/AUROC/AUPRC claim;
- reviewer feedback or assumed-normal history as behavioral ground truth; and
- promotion of a learned signal into behavioral decision authority.

## Prohibited metrics list

The following metric families are explicitly prohibited for Cycle 015.5
decision authority and must be recorded as unavailable with
`no_accepted_behavioral_ground_truth` if requested:

- `accuracy`
- `precision`
- `recall`
- `f1`
- `auroc`
- `auprc`
- `false_positive_rate`
- `false_negative_rate`
- any renamed equivalent claiming behavioral-validity classification quality

The implemented command
`backend/apps/anomalies/management/commands/evaluate_xai_scores.py` enforces
this veto by failing closed when such metrics appear in the reported-metrics
payload.

## Implementation boundary reached

Implemented in the current worktree:

- score contracts and serializer/view payload shape;
- bounded contamination-aware observed-pattern profiles;
- deterministic comparison and contribution decomposition;
- withholding and fusion rules;
- exact reconstruction and counterfactual replay helpers;
- persistence models and idempotent repositories;
- persisted-first score reads with metadata fallback;
- evaluation command, probe helper, and figure generator; and
- DB-free unit coverage across the full score slice.

Still open for full cycle closure:

- `T120` production stride-1 benchmark/collector wrapper;
- `T122` production rollback execution and results doc; and
- `T123` benchmark ledger entry from a real production run.

## Current blocker status

PostgreSQL-backed integration execution now succeeds locally after the dev
PostgreSQL compose recovery settings were corrected and the container was
recreated. The current local evidence is:

```text
.\.venv\Scripts\python.exe -m pytest tests/integration/anomalies/test_xai_score_persistence.py tests/contract/test_xai_review_priority_api_contract.py tests/unit/anomalies/test_views.py
# 25 passed
```

The remaining blocker for Cycle 015.5 is no longer database reachability. It is
the absence of a real production stride-1 benchmark run, rollback execution,
results doc, and benchmark-ledger entry.
