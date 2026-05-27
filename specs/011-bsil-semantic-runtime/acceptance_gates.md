# BSIL Acceptance Gates

## Bypass Prevention

No BSIL capability is accepted when:

- evidence is placeholder-only, stale or temporary-path-only;
- behavioral outputs lack lineage or confidence decomposition;
- CI passes but runtime evidence is absent;
- semantic outputs are generated from fallback payloads;
- dev-only evidence is marked production;
- PostgreSQL authority is bypassed;
- runtime reconciliation remains unresolved;
- stale artifacts are included in acceptance;
- fake or constant causality metrics are used as truth;
- heuristic outputs are presented as scientific truth;
- interaction claims ignore identity continuity confidence.

## Required Gate Results

| Gate | Required result |
| --- | --- |
| Runtime authority | active mode validated, inactive mode isolated, no silent fallback |
| Semantic safety | confidence, uncertainty and lineage on every accepted semantic output |
| Temporal reasoning | spike suppression, hysteresis, decay and invalid-window behavior pass |
| Episode integrity | immutable causal evidence chains and rollback-safe updates pass |
| Interaction graph | identity-gated edge confidence and suppression pass |
| Adaptive anomaly | baseline source, drift visibility and threshold context pass |
| Observability | dashboards and telemetry emit valid/degraded/unavailable states |
| Scientific evidence | representative datasets, repeated runs and false-positive/negative review pass |
| Reconciliation | task, DB, queue, artifact, telemetry and frontend state converge |

