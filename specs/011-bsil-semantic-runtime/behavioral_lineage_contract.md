# BSIL Behavioral Lineage Contract

## Glossary

| Term | Binding meaning |
| --- | --- |
| Semantic pose state | A lineage-backed interpretation of pose geometry and motion, not a conclusion about intent. |
| Behavioral state | A temporal state derived from repeated semantic evidence over a valid window. |
| Behavioral episode | A reviewable time-bounded summary assembled from behavioral states and causal evidence. |
| Interaction edge | A confidence-scored relationship between identities inside a valid temporal graph window. |
| Anomaly candidate | A review candidate produced by bounded baseline logic; it is not an adjudicated violation. |
| Human review task | A workflow item requesting reviewer assessment of evidence, not an automated penalty. |
| Review label | Governed validation feedback such as false-positive or false-negative, excluded from direct production threshold mutation. |

## Prohibited Accusation Terms

BSIL outputs, APIs, UI labels, evidence exports and task names MUST NOT use
deterministic accusation language for semantic, behavioral, episode,
interaction or anomaly candidates. Prohibited terms include `cheating`,
`misconduct`, `guilty`, `violation`, `offender`, `caught`, `proof`,
`confirmed cheating`, `confirmed misconduct`, `fraud`, `intentional` and
equivalent language that implies adjudicated wrongdoing.

Allowed language MUST remain observational and review-oriented, such as
`attention deviation candidate`, `sustained disengagement candidate`,
`interaction candidate`, `review-needed episode`, `uncertain`, `degraded`,
`suppressed` or `invalid`.

## Confidence Bands

Every accepted BSIL output MUST expose exactly one governed confidence band:

| Band | Meaning | Acceptance behavior |
| --- | --- | --- |
| `advisory` | Low-to-moderate support, suitable only for reviewer context. | Visible with uncertainty and no escalation. |
| `review-needed` | Sufficient support for human review workflow, but not high-confidence. | May create review queue context when policy allows. |
| `high-confidence` | Strong lineage-backed support across required temporal/runtime gates. | May create a human review task; never creates automatic accusation or penalty. |
| `degraded` | Output exists but one or more authorities are impaired. | Must render degraded cause and cannot satisfy maturity evidence without scoped acceptance. |
| `invalid` | Required lineage, timestamp, identity, replay, runtime or evidence authority failed. | Must be suppressed from production-valid behavior and counted as invalid. |

Numeric confidence, when present, is subordinate to the governed band. UI,
API, WebSocket and evidence exports MUST render the same band for the same
decision lineage record.

## Decision Lineage

Every accepted BSIL decision MUST be reconstructable from:

- source media or live capture manifest;
- source timestamps and frame references;
- session, camera, canonical identity and local track lineage;
- pose stream, confidence, missingness and occlusion state;
- semantic pose state and uncertainty reason;
- temporal reasoning window and state transition record;
- interaction edge lineage where multi-person context contributes;
- baseline and threshold context where anomaly scoring contributes;
- runtime mode, queue route, model or heuristic source and artifact digests.

## Required Artifacts

- `behavioral_decision_trace.json`
- `episode_lineage_report.json`
- `temporal_reasoning_audit.json`
- `interaction_graph_audit.json`

## Mutation Rules

- Accepted lineage records MUST NOT be mutated silently.
- Corrections MUST create superseding records with prior digest, reason and
  reviewer/operator context.
- Missing lineage invalidates the affected semantic state, behavioral state,
  episode, interaction edge or anomaly candidate.
