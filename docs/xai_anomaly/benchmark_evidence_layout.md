# XAI Benchmark Evidence Layout

**Last updated:** 2026-06-10

This document defines the feature-owned evidence layout for future
`specs/015-xai-anomaly-score/` benchmark runs. It does not claim that any Cycle
015 production benchmark has already been executed; it defines where the raw
artifacts, rendered figures, manifests, and decision records must live once a
cycle runs under production authority.

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `docs/BENCHMARK_RESULTS_LEDGER.md` |
| File | `docs/figures/benchmark_artifacts/` |
| File | `docs/benchmark_results/` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |
| Doc | `specs/015-xai-anomaly-score/contracts/benchmark-evidence-contract.md` |
| Doc | `specs/015-xai-anomaly-score/quickstart.md` |
| Doc | `docs/xai_anomaly/README.md` |

## Existing repo roots

The repo already has two benchmark-evidence roots:

| Path | Current role |
|---|---|
| `docs/figures/benchmark_artifacts/` | Rendered benchmark figure bundles keyed by replay/run label |
| `docs/benchmark_results/` | Feature/cycle-specific benchmark result summaries such as `cycle_014/` |

Cycle 015 reuses those roots instead of inventing a parallel evidence tree.

## Required layout

Each Cycle 015 benchmark run must produce one feature-owned run label and place
its evidence in the following structure:

```text
docs/
|-- benchmark_results/
|   `-- cycle_015/
|       `-- <run_label>/
|           |-- decision_summary.json
|           |-- metrics_snapshot.json
|           |-- unavailable_metrics.json
|           |-- rollback_summary.json
|           `-- manifest.json
`-- figures/
    `-- benchmark_artifacts/
        `-- <run_label>/
            |-- <run_label>__decision_delta.png
            |-- <run_label>__evidence_completeness.png
            |-- <run_label>__model_rtt.png
            |-- <run_label>__resource_tail.png
            |-- <run_label>__unavailable_summary.png
            `-- manifest.json
```

`<run_label>` must match the replay/benchmark identity used in the cycle
results doc and the benchmark ledger row.

## Required files

| File | Requirement |
|---|---|
| `decision_summary.json` | Cycle ID, decision, replay key, job ID, SHA, governed media path, stride, baseline/candidate/best labels |
| `metrics_snapshot.json` | Performance, model/runtime, storage/queue, correctness, XAI-quality, and lifecycle metrics from the authoritative run |
| `unavailable_metrics.json` | Every unavailable metric with its explicit reason; missing metrics must not be omitted silently |
| `rollback_summary.json` | Rollback command, execution status, reconciliation status, and proof paths |
| `manifest.json` | Input artifact paths + digests, output figure paths + digests, Markdown targets, and unavailable-metric disclosure |

The figure-bundle `manifest.json` may be identical to the decision-summary
manifest if one file already captures the full input/output digest graph, but a
manifest must exist at the figure root.

## Markdown targets

Every authoritative run must be linked from all three surfaces below:

| Surface | Required update |
|---|---|
| `docs/xai_anomaly/cycle_015_<id>_results.md` | Link the run label, raw result directory, figure directory, unavailable-metric file, and rollback summary |
| `docs/BENCHMARK_RESULTS_LEDGER.md` | Add one row with the run label, cycle ID, decision, stride `1`, job ID, and reason |
| `docs/xai_anomaly/README.md` | Keep the feature index current when a new recurring evidence-support doc is added |

## Naming contract

| Item | Rule |
|---|---|
| `cycle_015/` | Shared feature-level raw benchmark root for all Cycle 015 runs |
| `<run_label>` | Stable replay/run identifier reused across raw results, figures, results docs, and the ledger |
| `__decision_delta.png` etc. | Figure filenames keep the existing `docs/figures/benchmark_artifacts/*` naming style so cross-cycle tooling stays consistent |

## Current status

This layout document is a setup artifact only. It does not satisfy any cycle's
production benchmark requirement by itself. A Cycle 015 task may be checked
complete only after the cycle-specific raw results, figure bundle, manifest,
ledger row, and rollback proof exist in the working tree or attached production
evidence package.
