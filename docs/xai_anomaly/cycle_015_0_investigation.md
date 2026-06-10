# Cycle 015.0 Investigation — Runtime Truth, Route Snapshot, And BSIL Activation

**Last updated:** 2026-06-10
**Cycle:** `015.0`
**Status:** `staged_local_only`
**Streaming compatibility:** `stream-safe-with-config`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `backend/apps/pipeline/services/model_route_snapshot.py` |
| File | `backend/apps/pipeline/services/model_route_service.py` |
| File | `backend/apps/pipeline/rule_engine.py` |
| File | `backend/apps/video_analysis/tasks.py` |
| File | `backend/apps/behavior/models.py` |
| File | `backend/tests/unit/pipeline/test_xai_runtime_truth.py` |
| Doc | `specs/015-xai-anomaly-score/plan.md` |
| Doc | `specs/015-xai-anomaly-score/production-inventory-20260608.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |

## Goal

Cycle 015.0 establishes the minimum runtime-truth boundary needed before later
XAI/anomaly state can claim reconstructability:

- resolved model routes must be snapshotable and fingerprintable;
- service-local route mutation must not bleed into future service instances;
- runtime language at the legacy rule-engine edge must not emit accusatory text;
- later persistence/API work must attach to immutable route truth rather than
  mutable default tables.

## Current local slice

This turn only covers the local contract and service layer:

- immutable route snapshot value object;
- route-service cloning and snapshot APIs;
- non-accusatory rule-engine wording; and
- focused unit coverage for digest stability, override handling, parity, and
  vocabulary.

PostgreSQL persistence, serializer exposure, BSIL activation wiring, and
production helper scripts remain open work in later Cycle 015.0 tasks.

## Hypothesis

If route resolution becomes immutable and fingerprinted at the service layer,
later score/explanation state can bind to stable route truth instead of shared
mutable defaults, and legacy rule-engine text will stop violating Cycle 015
language constraints at the first runtime boundary.

## Acceptance boundary for this local slice

Local completion for this turn requires:

- `ModelRouteService.register_route()` mutates only the current service instance;
- identical route inputs produce stable snapshot digests;
- env overrides appear explicitly in snapshot metadata; and
- rule-engine outputs avoid prohibited accusatory wording.

This is not a production decision and does not satisfy Cycle 015.0 benchmark,
rollback, or ledger requirements.
