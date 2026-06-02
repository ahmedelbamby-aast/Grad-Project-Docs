# Cycle 9b B.4 Batch Window Investigation

**Last updated:** 2026-06-02

**Status:** **INVESTIGATION / HYPOTHESIS STAGED — NO OPTIMIZATION ACCEPTED.**

This document starts Cycle 9b B.4, the larger ensemble batch-window candidate.
It does not accept a new runtime default. Acceptance is blocked until the
production RTX 5090 benchmark completes on `combined.mp4`, proves a Step 2
improvement, keeps worker RSS below the guardrail, and preserves correctness
against the accepted 320 Top-K baseline.

## Problem Statement

The accepted production baseline is Cycle 9b exact slice + Top-K:

| Item | Value |
|---|---|
| Replay key | `cycle9b-topk-crop-frame-20260602T041900` |
| Job ID | `be4ba9ee-4786-48e9-8334-28feb237a1fb` |
| Behavior route | `behavior_ensemble_gaze_slice_topk` |
| Behavior input size | `320x320` |
| Batch window | `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=2` |
| Step 2 frame wall | `540.399 s` |
| DB-completed FPS | `4.429` |
| Avg / peak GPU util | `9.344 % / 53.0 %` |

Cycle 11.A proved that reducing behavior input size can improve Step 2 speed,
but it is not acceptable because it regressed persisted behavior signals. B.4
therefore tests a lower-risk lever: keep the exact accepted 320 Top-K model
graph unchanged and only raise the offline frame batch window.

## Root Cause Being Tested

The current accepted flow batches at most two offline frames into one dispatch
window. Classroom frames have roughly 17 person crops on the benchmark video, so
the active behavior ensemble often sees true batches near the engine cap of 32.
Raising the frame window from `2` to `4` can reduce per-window Python
orchestration and allow the crop fanout to keep Triton busier when per-frame
crop counts fluctuate.

This is **not** a dense-output-byte optimization and **not** an accuracy/model
change. The candidate keeps:

- `TRITON_CROP_BEHAVIOR_INPUT_SIZE=320`
- `GAZE_HORIZONTAL_HEAD_VARIANT=slice`
- `TRITON_BEHAVIOR_TOP_K_ENABLED=1`
- `TRITON_BEHAVIOR_TOP_K_VALUE=100`
- `MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble_gaze_slice_topk`
- `LPM_ENABLED=0`

## Named Lever

**Single-process Python orchestration / batch formation.**

The hypothesis is that a larger frame window reduces frame-synchronous
orchestration overhead and improves batch packing without changing TensorRT
plans, output tensors, or persisted signal semantics.

## Candidate

| Knob | Baseline | Candidate |
|---|---:|---:|
| `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES` | `2` | `4` |
| `TRITON_OFFLINE_BATCH_QUEUE_MAX_CONCURRENCY` | `2` | `2` |
| `OFFLINE_TRIM_EVERY_N_BATCHES` | `8` | `8` |
| Pose cap | `16` | unchanged |

## Expected Gain

Expected gain is marginal and must be measured. The useful signal is not merely
overall FPS; the primary target is Step 2 frame wall under the exact same 320
Top-K route. A successful run should show:

- lower Step 2 frame wall than `540.399 s`;
- lower or unchanged behavior RTT;
- no correctness drift;
- worker RSS below `4096 MiB`.

## Risk Assessment

Risk is medium because early crop-frame runs hit very high RSS with large frame
windows. That history came before the current no-copy NumPy split, job-scoped
gRPC loop, Top-K outputs, and trim cadence. The candidate must still carry an
RSS monitor because memory safety is part of the B.4 acceptance gate.

Primary failure modes:

- worker RSS grows above `4096 MiB`;
- larger windows increase waiting time and regress Step 2 wall;
- queueing delays lower GPU utilization despite bigger windows;
- correctness counters drift because the changed window alters timing or
  tracking order.

## Rollback Strategy

Rollback is an env-only restore:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_enable_parallel_flow.sh --profile per-frame-signals
bash tools/prod/prod_enable_behavior_topk.sh --input-size 320 --topk 100
bash -l tools/prod/prod_start_triton.sh
bash tools/prod/prod_start_celery_workers.sh
```

The accepted baseline value is:

```text
TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=2
```

## Acceptance Criteria

B.4 is accepted only if all gates pass on the production RTX 5090:

| Gate | Required result |
|---|---|
| Production benchmark | `combined.mp4` completes with DB `status=completed` |
| Step 2 frame wall | improves versus `540.399 s` |
| DB-completed FPS | improves or stays within noise while Step 2 improves |
| Worker RSS | peak sampled RSS `< 4096 MiB` |
| Correctness | frame/detection/bbox/embedding counters within established baseline tolerance |
| Per-model agreement | no model agreement regression versus accepted 320 Top-K baseline |
| Evidence | bench summary JSON, inference audit, GPU CSV, RSS TSV, metrics JSON/Markdown, model agreement JSON/Markdown |

If any gate fails, the result is **NOT ACCEPTED** and production stays on
`TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=2`.

## Reproduction Plan

The B.4 helper will run the canonical benchmark path, override only the existing
batch-window env knob, sample Celery worker RSS, collect metrics, compare to the
accepted baseline, and restore the baseline value on exit:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_run_batch_window_benchmark.sh \
  --max-frames 4 \
  --rss-limit-mib 4096 \
  --tag cycle9b-b4-maxframes4-$(date -u +%Y%m%dT%H%M%SZ) \
  --timeout 7200
```

Expected evidence directory:

```text
backend/logs/<tag>/
  batch_window_metrics.json
  batch_window_metrics.md
  batch_window_rss.tsv
  model_agreement_320_topk_vs_candidate.json
  model_agreement_320_topk_vs_candidate.md
  baseline_topk_metrics.json
  baseline_topk_metrics.md
```

## Decision

Pending production benchmark. No optimization is accepted by this investigation
document.
