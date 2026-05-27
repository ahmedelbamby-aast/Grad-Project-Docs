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

## Human-Reviewed Escalation Gate

High-confidence BSIL episodes may only create human review tasks. Acceptance
is blocked if any BSIL output creates an automatic accusation, penalty,
enforcement action, user sanction or irreversible operational decision. Review
task creation MUST be idempotent, linked to the episode lineage record,
auditable, replay-safe and visible in evidence exports.

## Debt, XFail and Drift Registry Gate

BSIL closure requires a registry of accepted technical debt, xfails and
runtime drift. Each entry MUST include owner, scope, affected requirement,
risk, expiry condition, blocking threshold and current evidence. Hidden
xfails, unowned debt, stale drift exceptions, skipped runtime gates or
unbounded deferrals block maturity closure.

The registry MUST explicitly state when no entries exist. A clean registry is
evidence only when CI/static scans verify no hidden xfail markers, placeholder
artifacts, SQLite evidence paths, unsupported fallback routes or drift
exceptions remain.
