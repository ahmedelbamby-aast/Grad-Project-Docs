# Cycle Result Template

**Last updated:** 2026-06-10

## Source-of-truth references

| Kind | Reference |
|---|---|
| Doc | `specs/015-xai-anomaly-score/contracts/benchmark-evidence-contract.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |
| Doc | `specs/015-xai-anomaly-score/quickstart.md` |
| Doc | `.specify/memory/constitution.md` |

## Purpose

Use this template for every `docs/xai_anomaly/cycle_015_X_results.md` file so
the decision state, missing evidence, and rollback boundary are reported
consistently.

## Template

```md
# Cycle 015.X Results

**Last updated:** YYYY-MM-DD
**Cycle:** `015.X`
**Status:** `<staged_local_only | ACCEPTED | NOT ACCEPTED | NEEDS FURTHER ITERATION | PROBE_ONLY | HYPOTHESIS_ONLY | ROLLBACK>`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `<benchmark runner>` |
| File | `<probe or collector>` |
| File | `<figure generator>` |
| File | `<rollback helper>` |
| Doc | `docs/xai_anomaly/cycle_015_X_investigation.md` |
| Doc | `docs/xai_anomaly/cycle_015_X_figure_plan.md` |
| Doc | `docs/xai_anomaly/cycle_015_X_figure_implementation.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |

## Current decision state

- what is implemented:
- what was validated locally:
- whether a production decision exists:
- what still blocks acceptance or closure:

## Verified local evidence

```text
<commands and short results>
```

## Production evidence

| Artifact | Status | Path / reason |
|---|---|---|
| stride-1 benchmark | pending/pass/fail | `<path or reason>` |
| figures + manifest | pending/pass/fail | `<path or reason>` |
| rollback proof | pending/pass/fail | `<path or reason>` |
| ledger entry | pending/pass/fail | `<path or reason>` |

## Knowledge limits

- no behavioral ground truth:
- unavailable metrics and reasons:
- reviewer-feedback boundary:

## Rollback boundary

- runtime state:
- DB state:
- Redis state:
- Triton state:
- artifact cleanup:
```
