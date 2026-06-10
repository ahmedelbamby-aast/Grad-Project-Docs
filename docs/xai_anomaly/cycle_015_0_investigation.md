# Cycle 015.0 Investigation — Runtime Truth, Route Snapshot, And BSIL Activation

**Last updated:** 2026-06-10
**Cycle:** `015.0`
**Status:** `staged_awaiting_production_evidence`
**Streaming compatibility:** `stream-safe`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `backend/apps/pipeline/services/model_route_snapshot.py` |
| File | `backend/apps/pipeline/services/model_route_service.py` |
| File | `backend/apps/pipeline/rule_engine.py` |
| File | `backend/apps/video_analysis/tasks.py` |
| File | `backend/apps/behavior/models.py` |
| File | `backend/tests/unit/pipeline/test_xai_runtime_truth.py` |
| File | `backend/tests/integration/behavior/test_xai_bsil_activation.py` |
| File | `tools/prod/prod_xai_cycle015_0_preflight.py` |
| File | `tools/prod/prod_run_xai_cycle015_0.sh` |
| File | `tools/prod/prod_rollback_xai_cycle015_0.sh` |
| Doc | `specs/015-xai-anomaly-score/atomic-cycles.md` |
| Doc | `specs/015-xai-anomaly-score/contracts/benchmark-evidence-contract.md` |
| Doc | `specs/015-xai-anomaly-score/plan.md` |
| Doc | `specs/015-xai-anomaly-score/production-inventory-20260608.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |

## Goal

Cycle 015.0 establishes the runtime-truth boundary required before later
XAI/anomaly state can use production evidence:

- resolved model routes must be snapshotable and fingerprintable;
- service-local route mutation must not bleed into future service instances;
- runtime language at the legacy rule-engine edge must not emit accusatory text;
- benchmark jobs must persist and reference the exact route snapshot;
- BSIL semantic, temporal, state, and lineage rows must reconcile to the
  benchmark job rather than to global table counts.

## Implemented scope

- `ModelRouteSnapshot` provides immutable per-task route values and digests.
- `ModelRouteService` isolates runtime registration from shared defaults.
- `XAIModelRouteSnapshot` persists route/config/repository/SHA truth.
- offline and live pose orchestration invoke bounded BSIL activation.
- benchmark preflight resolves the replay key, scopes PostgreSQL counts to its
  job/session, and verifies its persisted route snapshot against runtime truth.
- the runner enforces stride `1`, a same-SHA baseline/candidate pair, runtime
  conflict rejection, rollback, metrics collection, and figure generation.

## Hypothesis

Runtime-truth capture and BSIL activation can produce job-scoped,
reconstructable evidence without changing source model outputs or violating
the accepted runtime's latency, throughput, storage, or resource boundaries.

## Atomic acceptance gates

Cycle 015.0 remains indivisible. A production decision requires all gates:

1. Baseline and candidate complete on native Linux RTX 5090 with frame stride
   `1`, the same reviewed SHA, media, route, and profile.
2. The candidate benchmark job has nonzero job-scoped `SemanticPoseState` and
   `DecisionLineageRecord` rows; temporal/state counts are reported explicitly.
3. The benchmark job references a persisted `XAIModelRouteSnapshot` whose
   routes, config fingerprint, and SHA match the post-run preflight.
4. Mapping parity and prohibited-language tests pass.
5. Source frame, detection, bounding-box, embedding, track, pose, and scene
   outputs match the disabled baseline; any mismatch blocks acceptance.
6. FPS, phase wall, per-model RTT/call rate, GPU/VRAM, worker RSS,
   PostgreSQL/Redis, and queue metrics are available or carry a specific
   unavailable reason. A critical unavailable metric blocks acceptance.
7. Rollback disables XAI benchmark and all BSIL activation flags, restarts the
   governed workers, and produces a ready post-rollback reconciliation report.
8. Figures and their manifest digest the same raw artifacts used by the
   decision table, and every run is recorded in the benchmark ledger.

## Knowledge limits

- BSIL output is derived evidence, not behavioral ground truth.
- Cycle 015.0 does not train or fine-tune an anomaly model.
- Local and CI results establish implementation readiness only; they cannot
  accept or close the cycle without the production benchmark and rollback.
