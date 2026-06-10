# Configuration Ownership And Fingerprint Contract

**Last updated:** 2026-06-10

## Source-of-truth references

| Kind | Reference |
|---|---|
| Doc | `specs/015-xai-anomaly-score/plan.md` |
| Doc | `specs/015-xai-anomaly-score/spec.md` |
| Doc | `specs/015-xai-anomaly-score/contracts/anomaly-score-contract.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |
| File | `.env.example` |
| File | `scripts/ci/verify_no_hardcode.py` |
| Workflow | `.github/workflows/xai-anomaly.yml` |

## Contract

Every operational value used by Cycle 015 must resolve through one of two
provenance paths:

- `configured`: read from a named `.env` or settings key with a stable
  fingerprintable source; or
- `learned`: derived from a versioned baseline/calibration/profile artifact and
  recorded with a reconstructable reference.

Inline magic numbers in scoring, graph, gating, rendering, or promotion logic
are forbidden.

## Ownership

| Configuration family | Primary owner | Examples |
|---|---|---|
| Global enablement/schema | `apps.behavior` + `apps.anomalies` | `XAI_ENABLED`, `ANOMALY_SCORE_ENABLED`, `XAI_SCHEMA_VERSION` |
| Fast-path scoring gates | `apps.anomalies.scoring` | coverage, uncertainty, weight-profile keys |
| Deep-XAI lifecycle | `apps.behavior.explainability` | deep queue, deadlines, size caps, method allowlist |
| Calibration/conformal | `apps.anomalies.scoring` | calibration age/sample gates, conformal alpha |
| WebGL/render budgets | frontend shared WebGL core | context count, upload caps, tile sizes |
| Interaction-graph bounds | `apps.behavior.explainability` + frontend graph view | node/edge caps, live update latency, render FPS |
| Promotion lifecycle | `apps.pipeline.services` | shadow duration, latency SLO, approver roles |
| Retention/access | `apps.behavior` + `apps.audit` | artifact retention, role allowlist, audit toggles |

## Fingerprint requirements

Any persisted route/score/explanation artifact that depends on configured
values must carry enough information to reconstruct the applied config set:

| Field | Requirement |
|---|---|
| config key | Canonical environment/settings key name |
| config source | `.env`, Django settings, frontend env, or learned artifact |
| value | Resolved value at execution time |
| fingerprint | Stable digest of the contributing config bundle |
| schema/version | XAI/anomaly schema version active at execution time |
| compatible scope | Route/model/profile/baseline scope the value applies to |

## Minimum setup keys

The setup slice does not define runtime defaults beyond `.env.example`, but it
does require these categories to exist and stay fingerprintable:

- global enablement;
- scoring/profile gates;
- deep-XAI bounds;
- calibration/conformal gates;
- WebGL budgets;
- graph budgets;
- promotion lifecycle;
- retention/access;
- benchmark toggles.

## Enforcement

- `scripts/ci/verify_no_hardcode.py` rejects inline operational constants that
  are not mapped to configured or learned provenance.
- `.github/workflows/xai-anomaly.yml` runs that verifier on focused changes.
- Future Cycle 015 persistence models must store configuration provenance rather
  than reconstructing it from mutable defaults after the fact.
