# Production Cycle Runbook

**Last updated:** 2026-06-10

## Source-of-truth references

| Kind | Reference |
|---|---|
| Doc | `specs/015-xai-anomaly-score/quickstart.md` |
| Doc | `specs/015-xai-anomaly-score/contracts/benchmark-evidence-contract.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |
| Doc | `docs/production_inference_benchmark.md` |
| Doc | `docs/inference_parallelization_plan.md` |
| Doc | `docs/cycle_9_and_10_improvements_todo.md` |
| File | `tools/prod/prod_install_instrumentation_tools.ps1` |

## Purpose

This template defines the minimum preflight, benchmark, figure, and rollback
sequence for a real Cycle 015 production run. It is a template, not proof that
any specific cycle has already been benchmarked.

## Preflight checklist

1. Confirm pushed branch/SHA matches production checkout.
2. Confirm PostgreSQL, Redis, Celery, Triton, and instrumentation readiness.
3. Confirm one active route/profile and capture the route/config fingerprint.
4. Confirm no conflicting active job and no duplicate workers.
5. Confirm the cycle's benchmark runner, figure generator, and rollback helper
   exist and are executable.
6. Confirm the cycle's Figure Planner and Figure Implementer docs are present.

## Minimum command sequence

```text
1. Pull the committed SHA on production
2. Run cycle-specific preflight helper
3. Run stride-1 baseline if required by the cycle
4. Run stride-1 candidate benchmark with one causal delta
5. Collect raw metrics and artifact digests
6. Generate figures from those exact raw artifacts
7. Update the cycle results doc and benchmark ledger
8. Execute rollback and capture reconciliation proof
```

## Required artifact families

| Family | Requirement |
|---|---|
| Runtime truth | SHA, env/profile, route snapshot, model/version fingerprints |
| Benchmark metrics | FPS, latency, per-model RTT/call rate, resource metrics |
| Correctness | parity/identity/task-quality metrics or explicit unavailable reason |
| XAI/anomaly quality | reconstruction, invariants, coverage, withholding, stability |
| Figure bundle | images, manifest, digests, unavailable-metric disclosures |
| Rollback | command, output, reconciled state proof |
| Ledger | `docs/BENCHMARK_RESULTS_LEDGER.md` row for every run/decision |

## Rollback template

Each cycle-specific rollback helper should report at least:

- whether runtime flags changed;
- whether DB or Redis state changed;
- whether Triton route/profile changed;
- which artifacts were preserved or cleaned; and
- whether reconciler convergence was verified.

## Failure handling

- Missing production evidence means the cycle remains `staged_local_only` or
  another non-acceptance state.
- Probe-only outputs must not be restated as acceptance.
- Any unavailable metric must be recorded with a reason, not omitted.
