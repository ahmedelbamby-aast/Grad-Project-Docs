# BSIL Behavioral Lineage Contract

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

