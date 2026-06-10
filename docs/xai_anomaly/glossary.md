# XAI/Anomaly Glossary

**Last updated:** 2026-06-10

## Source-of-truth references

| Kind | Reference |
|---|---|
| Doc | `specs/015-xai-anomaly-score/spec.md` |
| Doc | `specs/015-xai-anomaly-score/plan.md` |
| Doc | `specs/015-xai-anomaly-score/no-ground-truth-doctrine.md` |
| Doc | `specs/015-xai-anomaly-score/contracts/anomaly-score-contract.md` |
| Doc | `specs/015-xai-anomaly-score/signal-catalog.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |
| File | `scripts/ci/verify_xai_language.py` |

## Required terms

| Term | Meaning | Not equivalent to |
|---|---|---|
| `review_priority_score` | Deterministic triage score for human review | cheating probability |
| `within_observed_pattern` | Valid context exists and the current window remains inside the governed pattern envelope | proof of innocence or normality |
| `pattern_deviation` | Valid context exists and the current window differs materially from the governed pattern envelope | proof of misconduct or intent |
| `insufficient_context` | Valid comparison cannot yet be made | low-risk label |
| `withheld` | A blocking gate invalidated or suppressed the score/output | numeric zero |
| observed-pattern profile | Versioned, contamination-aware baseline for compatible signal windows | known-normal ground truth |
| pattern window | Bounded per-student multivariate signal slice used for comparison | whole-session truth |
| evidence coverage | Fraction of required valid evidence that survived gating | confidence alone |
| reliability | Source-signal trust factor after quality/compatibility checks | truth |
| uncertainty | Explicit statement of remaining score/explanation ambiguity | correctness error rate |
| contradiction | Evidence from different signals that does not reconcile cleanly | failure to decide automatically |
| route snapshot | Immutable point-in-time model/config/runtime fingerprint used for reconstruction | live mutable defaults |
| local boundaries | A student's own time-windowed profile envelope | population norm |
| general boundaries | Population/context baseline envelope across many students/sessions | a single student's baseline |
| quarantine | Exclusion of drifting, disputed, or invalid windows from profile updates | deletion of historical evidence |
| contamination | Risk that unusual/invalid evidence would distort future baseline updates | accepted truth |
| knowledge limits | Explicit statement of what the system cannot conclude | optional caveat |

## Promotion terms

| Term | Meaning |
|---|---|
| `PROBE_ONLY` | Research-only output that cannot drive a production score |
| `SHADOW` | Production inputs copied, outputs discarded from user-visible state |
| `CANARY` | Limited reviewer-visible advisory state under rollback gates |
| `MANDATORY` | Governed production signal/representation role only |

## Language guard

Preferred user-facing wording:

- review priority
- supporting evidence
- contradicting evidence
- unusual relative to baseline
- requires human review

Prohibited wording:

- confirmed cheating
- guilt
- misconduct probability
- automatic penalty
- proven abnormal intent
