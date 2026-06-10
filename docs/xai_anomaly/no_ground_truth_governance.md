# No-Ground-Truth Governance

**Last updated:** 2026-06-10

## Purpose

This document operationalizes the binding no-ground-truth doctrine for Cycle
015 setup and reviewer-facing governance artifacts. It exists so future XAI and
anomaly implementation work does not drift into hidden training, manufactured
truth, or accusatory language.

## Source-of-truth references

| Kind | Reference |
|---|---|
| Doc | `specs/015-xai-anomaly-score/no-ground-truth-doctrine.md` |
| Doc | `specs/015-xai-anomaly-score/spec.md` |
| Doc | `specs/015-xai-anomaly-score/plan.md` |
| Doc | `specs/015-xai-anomaly-score/research.md` |
| Doc | `specs/015-xai-anomaly-score/contracts/anomaly-score-contract.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |
| File | `scripts/ci/verify_xai_language.py` |
| Workflow | `.github/workflows/xai-anomaly.yml` |

## Binding governance rules

1. Cycle 015 has no accepted anomaly, cheating, misconduct, or normality
   ground truth.
2. Reviewer assessments are operational feedback only. They can identify
   confusing evidence, disagreement, or triage usefulness, but they cannot:
   train or fine-tune an anomaly model, relabel production history as truth,
   mutate a student's score/profile, or establish behavioral-validity metrics.
3. Heuristic output, BSIL output, source-model agreement, and assumed-normal
   history are not anomaly ground truth and may not be backfilled as labels.
4. The allowed persisted/user-facing states remain
   `within_observed_pattern`, `pattern_deviation`,
   `insufficient_context`, and `withheld`.
5. `review_priority_score` remains a human-review triage score, not a verdict,
   probability of cheating, or proof of intent.

## Reviewer-assessment handling

| Reviewer activity | Allowed use | Forbidden use |
|---|---|---|
| Flagging confusing evidence | UX/documentation improvement | Converting the flag into anomaly truth |
| Confirming a review-worthy deviation | Session/audit evidence only | Training target or peer-student mutation |
| Disagreeing with a score | Calibration of workflow/documentation limits | Rewriting historical labels as truth |
| Providing free-text notes | Immutable audit/supporting evidence | Hidden supervised dataset creation |

## Metric boundary

The following claims remain prohibited under the current plan:

- anomaly accuracy;
- cheating-detection accuracy;
- precision, recall, F1, AUROC, AUPRC, false-positive rate, or false-negative
  rate for anomaly behavior; and
- language that equates `within_observed_pattern` with innocence or
  `pattern_deviation` with misconduct.

If a future artifact would ordinarily require one of those metrics, it must
record `no_accepted_behavioral_ground_truth` instead.

## Future-plan boundary

Any future trainable anomaly/behavior model requires a separate governed
program that establishes all of the following before implementation authority
exists:

- legitimate behavioral target semantics;
- dataset provenance and labeling method;
- reviewer-role boundaries;
- acceptance metrics and benchmark authority; and
- promotion/rollback governance for a behavioral decision surface.

Until that program exists, learned anomaly methods remain `HYPOTHESIS_ONLY` or
`PROBE_ONLY` and may be promoted only into governed signal/representation roles
 under the model-promotion lifecycle.

## Enforcement surfaces

- `scripts/ci/verify_xai_language.py` guards prohibited vocabulary and
  manufactured-ground-truth phrasing.
- `.github/workflows/xai-anomaly.yml` runs the language verifier on focused
  Cycle 015 changes.
- Cycle investigation/results docs must keep knowledge limits explicit and must
  not use reviewer feedback as training authority.
