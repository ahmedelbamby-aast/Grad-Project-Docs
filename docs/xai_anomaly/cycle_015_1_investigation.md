# Cycle 015.1 Investigation: Versioned Evidence Envelope And Signal Registry

**Created:** 2026-06-11
**Last updated:** 2026-06-11
**Status:** `implemented_local; awaiting production benchmark`
**Streaming compatibility:** `stream-safe` (envelope normalization/persistence
is bounded; no live-unbounded state)

## Hypothesis

Normalizing every active signal through one governed contract (registry +
bounded envelope + idempotent persistence) adds bounded overhead and unlocks
all later XAI/anomaly cycles, without critical-path regression.

## Implemented (2026-06-11)

| Piece | Location |
|---|---|
| Contracts (truth states, payload bounds, digests, idempotency) | `backend/apps/behavior/explainability/contracts.py` |
| Registry (one active definition per key/version; fast-explainer mandatory; unregistered refused) | `backend/apps/behavior/explainability/registry.py` |
| Normalization (timestamps/identity/quality; missing-is-not-zero; truth-state derivation) | `backend/apps/behavior/explainability/signals.py` |
| 34 governed signal-family definitions (catalog sections 1-10 + interaction graph) | `backend/apps/behavior/explainability/signal_definitions.py` |
| PostgreSQL models + migration 0007 | `backend/apps/behavior/models.py` |
| Idempotent repository (replay returns existing row; bounded queries) | `backend/apps/behavior/repositories.py` |
| Bounded read serializers | `backend/apps/behavior/serializers.py` |
| Tests: 9 unit + 4 integration, all passing | `backend/tests/unit/behavior/test_xai_evidence_registry.py`, `backend/tests/integration/behavior/test_xai_evidence_persistence.py` |
| Read-only production registry/route/persistence coverage probe | `tools/prod/prod_probe_xai_signal_registry.py` |

Config: `XAI_EVIDENCE_MAX_PAYLOAD_BYTES` (default 16384) bounds every JSON
field at the contract layer (FR-001/FR-008).

## Atomic acceptance gates (open until the production benchmark)

- complete active-signal registry coverage on the production route snapshot;
- idempotent replay proof on production media;
- envelope payload-size distribution within bound; explicit missingness rates;
- normalization latency + DB write cost within the cycle budget (constitution
  7.1.1 families), measured AFTER Cycle 015.17 restores the throughput budget;
- rollback: feature consumes no critical-path time when unused (it is a
  library + tables; no orchestration hook is enabled by default).

## Rollback

No flag is required for inertness: nothing calls the registry/repository from
the critical path yet. Rollback = do not call; tables remain empty.
