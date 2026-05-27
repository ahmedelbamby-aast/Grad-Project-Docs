# BSIL Frontend Surfaces

BSIL components render reviewer-facing truth states and governed confidence
bands. They must not convert backend `degraded`, `invalid`, `unknown` or
`unavailable` states into successful UI states.

Semantic and behavioral outputs must stay observational. Do not use
accusatory wording or imply adjudicated misconduct from a confidence band.

## Review Surfaces

| Surface | Reviewer purpose | Required visible context |
| --- | --- | --- |
| `SemanticStatePanel` | Review pose-derived observations | semantic state, source-time window, track, uncertainty, truth state and confidence band |
| `TemporalStateTimeline` | Understand accumulated behavior state | transition reason, lifecycle, source-time window, truth state and replay mismatch |
| `BehavioralEpisodePanel` | Review explainable episode candidates | causal evidence count, lifecycle, truth state, confidence and human review task reference |
| `InteractionGraphPanel` | Review multi-person context | participants, identity continuity confidence, edge confidence and suppression reason |
| `AnomalyReviewPanel` | Apply governed validation labels | baseline version, drift/provenance state and validation-only label status |

## Confidence Bands

Confidence is evidence context, not a verdict. Surfaces must display the
governed band received from the backend and keep the following meaning:

| Band | UX meaning | Allowed reviewer action |
| --- | --- | --- |
| `advisory` | Observable signal with limited certainty | inspect supporting context only |
| `review-needed` | Evidence warrants human attention | open or continue human review |
| `high-confidence` | Strong lineage-backed candidate | create or display an auditable human review task only |
| `degraded` | Runtime, identity, pose or lineage quality is impaired | show limitation; do not promote the output |
| `invalid` | Record cannot support interpretation | suppress decisions and display invalidity reason |

The UI must not reinterpret a numeric confidence score into a stronger band,
nor present `high-confidence` as misconduct, confirmation or enforcement.

## Truth States And Uncertainty

- Truth state and confidence band must remain visible together; a degraded or
  invalid truth state overrides positive presentation.
- Missing lineage, identity discontinuity, stale runtime, replay mismatch,
  suppressed graph edges and baseline drift must remain reviewer-visible.
- Source-time windows and causal evidence references provide traceability and
  must not be replaced by presentation-only timestamps or summaries.
- Unknown or unavailable values should remain explicit rather than rendered
  as empty success, normal operation or zero risk.

## Review-Only Escalation

High-confidence episodes are eligible for human review task creation only
when backend lineage and runtime gates accept them. The frontend may expose
the linked review task and its status; it must not create language or controls
that imply automatic accusation, discipline, penalty or enforcement.

Review labels, including false-positive and false-negative labels, are
validation dataset feedback only. `AnomalyReviewPanel` must continue to show
that status and must not imply reviewer feedback directly changes production
thresholds or current episode truth.
