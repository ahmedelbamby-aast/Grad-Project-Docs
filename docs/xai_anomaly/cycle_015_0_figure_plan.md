# Cycle 015.0 Figure Plan

**Last updated:** 2026-06-10
**Cycle:** `015.0`
**Role:** `Figure Planner`
**Owner:** `Codex Cycle 015.0 Figure Planner`
**Status:** `planned_and_implemented_locally`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `backend/apps/pipeline/services/model_route_snapshot.py` |
| File | `backend/tests/unit/pipeline/test_xai_runtime_truth.py` |
| File | `tools/prod/prod_xai_cycle015_0_preflight.py` |
| File | `tools/prod/prod_generate_xai_cycle015_0_figures.py` |
| Doc | `specs/015-xai-anomaly-score/contracts/benchmark-evidence-contract.md` |
| Doc | `docs/xai_anomaly/cycle_015_0_investigation.md` |

## Planner decision

Cycle 015.0 figures must prove:

- resolved route truth is explicit and stable across the candidate run; and
- job-scoped BSIL/lineage rows become nonzero without regressing source
  outputs, performance, resources, storage, or queues.

## Required raw inputs

| Input | Producer | Required for |
|---|---|---|
| baseline preflight JSON | `tools/prod/prod_xai_cycle015_0_preflight.py` | disabled route/config/BSIL truth |
| candidate preflight JSON | `tools/prod/prod_xai_cycle015_0_preflight.py` | job-scoped route/BSIL truth |
| baseline metrics JSON | `tools/prod/prod_collect_benchmark_metrics.py` | baseline and best-comparable values |
| candidate metrics JSON | `tools/prod/prod_collect_benchmark_metrics.py` | candidate performance and parity |
| rollback status JSON | `tools/prod/prod_rollback_xai_cycle015_0.sh` | rollback verification |

For this first XAI cycle, the current accepted disabled profile is both the
baseline and best-comparable run. The manifest records the shared input digest
rather than manufacturing a separate best run.

## Required figures

| Figure | Purpose | Raw fields consumed |
|---|---|---|
| `route_snapshot_matrix` | active task-to-model truth at benchmark time | task key, model name, version, source, digest |
| `route_digest_stability` | prove candidate run uses one coherent route truth | route digest, environment digest |
| `bsil_activation_delta` | show nonzero lineage/semantic activation | baseline vs candidate row counts |
| `runtime_truth_vocabulary` | show no prohibited wording leaked through runtime summaries | summary text, reason codes |
| `benchmark_metric_delta` | baseline/candidate/best FPS, wall, and latency | baseline and candidate metrics |
| `model_rtt_call_rate` | expose every model's RTT and call rate | per-model telemetry |
| `resource_delta` | compare GPU, VRAM, and worker RSS | GPU/RSS summaries |
| `storage_queue_delta` | compare PostgreSQL, Redis, and queue work | embedding/audit summaries |
| `source_output_parity` | compare source output counters | correctness summaries |
| `evidence_completeness` | show required artifact presence | all five raw inputs |
| `unavailable_summary` | count unavailable values by run role | manifest unavailable records |
| `rollback_verification` | prove route/profile state converged after rollback | rollback flags and reconciled digests |

## Unavailable-metric policy

- Missing values render as `unavailable` with a metric, run role, and reason;
  they are never plotted as zero.
- Missing baseline/candidate/best metrics render the whole cross-run panel as
  unavailable.
- The final manifest links every figure to the exact raw route, benchmark, and
  rollback artifacts used by the Cycle 015.0 decision table.

## Role separation

Only one implementation agent was active for this local cycle, so separate
agents were unavailable. Planning authority remains isolated in this file;
generator code, tests, manifests, and produced images remain isolated in the
Figure Implementer record.
