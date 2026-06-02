# Behavioral Ontology v1

**Last updated:** 2026-05-25

## Feature Families
- `head`: direction proxy, glance duration, switching frequency
- `wrist`: velocity, disappearance duration, visibility ratio
- `motion`: entropy, activity ratio, instability
- `torso`: orientation variance, leaning variation
- `interaction`: overlap, synchronized motion, proximity cues

## Missing Data Semantics
- Missing values are explicit (`missing_state`) and are never coerced to zero.
- Visibility masks must propagate from pose stream through feature windows.

## Anomaly Primitive Semantics
- `change_point`
- `drift`
- `repeated_pattern`
- `instability`
- `attention_deviation`

