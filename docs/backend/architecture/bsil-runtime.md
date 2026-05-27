# BSIL Runtime Read Surfaces and Evidence

**Updated**: 2026-05-27

BSIL turns pose and runtime lineage into review-oriented semantic and temporal
records. Outputs are observations or review candidates, never automatic
findings of wrongdoing. PostgreSQL is the durable authority for BSIL state,
lineage, review data, and accepted evidence.

## Implemented API Surfaces

Behavior routes are mounted below `/api/v1/behavior/`. The implemented list
views use `IsAdminUser`, are read-only, and accept available combinations of
`session_id`, `camera_source_id`, and `canonical_track_id` filters.

| Endpoint | Record exposed |
| --- | --- |
| `GET /api/v1/behavior/bsil/semantic-states/` | Semantic pose states and confidence lineage |
| `GET /api/v1/behavior/bsil/temporal-windows/` | Ordered reasoning windows |
| `GET /api/v1/behavior/bsil/behavioral-states/` | Temporal behavioral state outputs |
| `GET /api/v1/behavior/bsil/episodes/` | Review-oriented behavioral episodes |
| `GET /api/v1/behavior/bsil/episode-review-tasks/` | Episodes with a review task reference |
| `GET /api/v1/behavior/bsil/interaction-edges/` | Identity-scoped interaction edges |
| `GET /api/v1/behavior/bsil/decision-lineage/` | Reconstructable decision lineage records |
| `GET /api/v1/behavior/bsil/review-labels/` | Governed review label records |

Anomaly-side BSIL records are routed through `/api/v1/anomalies/`:

| Endpoint | Behavior |
| --- | --- |
| `/api/v1/anomalies/bsil/baselines/` | Read-only baseline snapshots |
| `/api/v1/anomalies/bsil/threshold-shifts/` | Read-only threshold shift lineage |
| `/api/v1/anomalies/bsil/review-labels/` | Review-label record view set |

`GET /api/v1/behavior/bsil/access-audit/` is currently an explicit
`501 Not Implemented` query surface with `truth_state="unavailable"`.
Acceptance does not use that query as proof: it requires a PostgreSQL-backed,
digest-addressed access-audit evidence manifest containing recorded audit
events.

## Truth and Confidence

Every accepted semantic state, behavioral state, episode, interaction edge, or
anomaly candidate exposes one band: `advisory`, `review-needed`,
`high-confidence`, `degraded`, or `invalid`. A `high-confidence` episode may
create a human review task; it does not authorize automatic enforcement.
API, UI, WebSocket, and exported evidence must render the same band for the
same lineage record.

## Evidence Acceptance

Accepted evidence retains raw media or live-capture manifests, frame and pose
references, temporal windows, runtime/queue telemetry, replay inputs,
decision lineage, review labels, audit events, and derived artifacts. Published
evidence is immutable and digest-addressed; placeholder, fallback, stale,
synthetic production, or SQLite-backed evidence is rejected.

The final acceptance command is implemented by
`backend/apps/behavior/management/commands/run_bsil_acceptance.py`. It checks:

| Gate | Acceptance condition |
| --- | --- |
| Evidence manifest | Fresh immutable raw-trace-backed evidence with PostgreSQL authority |
| Runtime reconciliation | Task, database, queue, artifact, telemetry, and frontend components converge |
| GPU/queue causality | Raw non-synthetic trace, one mode, mode-correct queue, and Triton/NVIDIA attribution |
| Access audit | PostgreSQL-backed valid audit manifest with at least one digest-addressed event collection |

Production acceptance evaluation runs through:

```powershell
.\tools\prod\prod-bsil-final-evidence.ps1 -RuntimeMode offline
.\tools\prod\prod-bsil-final-evidence.ps1 -RuntimeMode live
```

Run only the mode selected for the production validation session. The output
manifest is written beneath `ci_evidence/bsil/production/final/`.

## Source Contracts

- [evidence_contract.md](../../../specs/011-bsil-semantic-runtime/evidence_contract.md)
- [runtime_governance.md](../../../specs/011-bsil-semantic-runtime/runtime_governance.md)
- [behavioral_lineage_contract.md](../../../specs/011-bsil-semantic-runtime/behavioral_lineage_contract.md)
- [observability-runbook.md](observability-runbook.md)
- [deployment-topology.md](deployment-topology.md)
