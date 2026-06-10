# Mandatory No-Ground-Truth Anomaly Doctrine

**Date**: 2026-06-08
**Status**: Binding for every Cycle 015 artifact and implementation decision

## Authoritative Constraint

The project currently has:

- no governed anomaly, abnormal-behavior, cheating, or non-cheating dataset;
- no accepted ground-truth method for assigning those behavioral states; and
- no valid target labels for training or fine-tuning an anomaly model.

Therefore, Cycle 015 MUST NOT train or fine-tune a supervised anomaly,
misconduct, cheating, or normality classifier. Reviewer feedback, heuristic
rules, existing model outputs, and current BSIL records MUST NOT be relabeled as
ground truth or used as hidden training targets.

This constraint does not invalidate separately governed calibration evidence
for an existing source model. Source-model calibration is valid only where its
own task-appropriate held-out evidence exists and is explicitly referenced. It
does not become anomaly-behavior ground truth.

## Production Behavioral Assessment

The production anomaly layer depends on the temporal pattern of valid signals
produced by all available models for each student. It evaluates how a current
bounded multivariate signal window relates to a versioned, contamination-aware
pattern profile. The first production implementation is deterministic,
reconstructable pattern analysis, not a trainable anomaly model.

The pattern feature vector may include:

- model outputs, margins, entropy, reliability, and missingness;
- pose kinematics, posture, gaze direction, hand raising, and transitions;
- position, motion, velocity, acceleration, dwell, and trajectory changes;
- scene-object distance, direction, overlap, and relation changes;
- temporal persistence, recurrence, volatility, synchronization, and
  contradictions across models; and
- identity continuity, occlusion, reconnect gaps, and evidence quality.

Every feature keeps its source, timestamp, unit, validity state, and lineage.
Missing or invalid signals remain missing; they are never converted to normal
evidence.

## Permitted Pattern Methods

The first production path may use governed, versioned, bounded methods such as:

- robust median/MAD and quantile envelopes;
- bounded rolling and exponentially weighted statistics;
- rate-of-change, transition-frequency, persistence, and hysteresis;
- cross-signal correlation and contradiction patterns;
- change-point and drift indicators that do not claim behavioral truth; and
- person/session/camera/scene/context profiles with contamination, cold-start,
  expiry, and quarantine controls.

Unsupervised or self-supervised learned anomaly candidates may only be
`HYPOTHESIS_ONLY` or `PROBE_ONLY` under this plan. They cannot be promoted as a
behavioral-validity improvement, trained on assumed-normal data presented as
truth, or used to fine-tune production behavior decisions. Any future
trainable anomaly model requires a separate governed plan that establishes a
legitimate dataset, target semantics, evaluation authority, and promotion
contract.

## Probe Promotion Boundary

A `PROBE_ONLY`/`HYPOTHESIS_ONLY` model — necessarily unsupervised or
self-supervised, since no ground-truth labels exist — may be promoted to a
production-mandatory role **only** through the evidence-gated promotion lifecycle
defined in
[pretrained-models-registry.md](pretrained-models-registry.md)
(`PROBE_ONLY` → `SHADOW` → `CANARY` → `MANDATORY`), and **only** into a governed
**signal/representation** role whose promotion claims serving quality
(reproducibility/determinism, latency/throughput/resource SLOs, signal
stability, drift behavior), **never** behavioral validity.

Under this plan a probe MUST NOT be promoted into a behavioral, cheating, or
normality **decision authority**, and no promotion may report anomaly accuracy,
precision, recall, F1, AUROC, or AUPRC. Promoting a model into a behavioral
decision target requires the separate governed ground-truth program named above.

Every promotion is **evidence-driven** — a native production benchmark, computed
serving metrics, and proven rollback — recorded in an immutable promotion
record, reversible, and approved by a designated governed approver. Gates are a
bounded minimum bar, not an open-ended obstacle: once the recorded evidence
meets them, promotion is permitted.

Probe **fine-tuning** is permitted only on **copies** (the parent frozen model is
never mutated) under the registry's Probe Fine-Tuning Lane: filtered
pseudo-label self-training, self-supervised continued pretraining, ephemeral
test-time adaptation, or distillation from governed deterministic signals.
Accepted/refused/edited inference signals used for fine-tuning are model-derived
training signals — they MUST NOT be called ground truth, and reviewer feedback
may only exclude/quarantine corpus windows, never supply a label. A fine-tuned
copy is a new `PROBE_ONLY` lineage entry and earns any higher stage only through
the same evidence-gated lifecycle.

## Required Behavioral Vocabulary

The user-facing and persisted states are:

- `within_observed_pattern`: enough valid context exists and the current
  window remains inside the governed pattern envelope;
- `pattern_deviation`: enough valid context exists and the current window
  differs materially from the governed pattern envelope;
- `insufficient_context`: a valid pattern comparison cannot be made; and
- `withheld`: a blocking validity, identity, drift, or reconciliation gate
  failed.

`within_observed_pattern` does not mean a student is normal, innocent, or not
cheating. `pattern_deviation` does not mean abnormal intent, misconduct, or
cheating. A `review_priority_score` ranks evidence for human inspection; it is
not a class probability or behavioral verdict.

Operationally, "normal until proven otherwise" is represented only as
`within_observed_pattern` or `insufficient_context`. If one student's evidence
is later review-approved as a meaningful deviation, that approval may raise a
separate session/classroom aggregate but must not mutate any peer student's
score or pattern state. If operators use `abnormal` or `cheating` as
equivalent shorthand, that shorthand collapses to the same governed
review-approved deviation path; it does not create a separate persisted
behavioral taxonomy.

## Baseline And Cold-Start Rules

- A profile is never called known-normal ground truth.
- A new student/session/camera remains `insufficient_context` until the
  configured minimum valid evidence exists.
- Profile updates are bounded, versioned, append-only, and contamination-aware.
- High-deviation, disputed, drifting, invalid, or low-quality windows are
  quarantined from profile updates.
- A whole session that is consistently unusual cannot silently redefine itself
  as normal.
- Cross-person context is advisory, identity-gated, and cannot substitute for
  the student's own valid history.
- The general/population baseline is computed across many students/sessions and is
  never a single student's profile. It supplies General Boundaries, while each
  student's own time-windowed, cold-start-aware profile supplies Local Boundaries;
  a classroom-level deviation derived from the General Boundaries is never a
  per-student verdict.
- Route, ontology, feature-schema, or model-artifact changes invalidate
  incompatible profiles.

## Evaluation Without Behavioral Ground Truth

No cycle may claim anomaly accuracy, precision, recall, F1, AUROC, AUPRC,
cheating detection, or abnormal-behavior detection quality under the current
constraint.

Production acceptance instead requires:

- exact score and contribution reconstruction;
- deterministic replay and idempotency;
- controlled synthetic signal-pattern fixtures that test algorithm semantics,
  not real-world behavioral truth;
- metamorphic and invariant tests for missingness, scale, ordering, route
  compatibility, identity gaps, and contradiction handling;
- sensitivity, threshold, hysteresis, and counterfactual checks;
- cold-start, contamination, quarantine, drift, and recovery tests;
- real-media stability, bounded-state, latency, throughput, storage, and
  rollback evidence; and
- reviewer usability and disagreement evidence explicitly labeled
  non-ground-truth.

Reviewer feedback may identify confusing evidence, false escalation concerns,
or operational usefulness. It remains immutable evaluation evidence and MUST
NOT directly train/fine-tune an anomaly model, establish behavioral truth,
update a pattern profile, or mutate production policy.

## Mandatory Vetoes

An implementation or cycle decision is invalid if it:

- introduces a supervised anomaly/cheating/normality target;
- calls reviewer feedback, heuristic output, assumed-normal history, or source
  model agreement ground truth;
- reports label-based anomaly accuracy without a separately governed future
  ground-truth authority;
- describes `within_observed_pattern` as proof of non-cheating;
- describes `pattern_deviation` or a high score as proof of cheating or
  abnormal intent;
- silently updates profiles from quarantined or invalid windows;
- promotes a probe into a behavioral/cheating/normality decision authority, or
  promotes any model into production without the recorded evidence, benchmark,
  serving metrics, rollback proof, and governed-approver sign-off; or
- hides insufficient context behind a numeric zero or low-risk label.
