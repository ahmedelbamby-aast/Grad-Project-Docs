# Cycle 9b B.4 Batch Window Results

**Last updated:** 2026-06-02

**Status:** **NOT ACCEPTED. Real production benchmark completed, but correctness
gates failed.**

Cycle 9b B.4 tested a single runtime lever: raising
`TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES` from `2` to `4` while keeping the
accepted `320x320` exact-slice + Top-K behavior route unchanged. The candidate
completed the full `combined.mp4` production benchmark and produced a modest
throughput win with bounded RSS, but it changed tracking continuity and
model-agreement metrics enough to fail the acceptance gate.

The accepted production baseline remains Cycle 9b B.2.c:
`TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=2`,
`TRITON_CROP_BEHAVIOR_INPUT_SIZE=320`,
`MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble_gaze_slice_topk`,
`TRITON_BEHAVIOR_TOP_K_ENABLED=1`, and `LPM_ENABLED=0`.

## Baseline

| Item | Value |
|---|---|
| Baseline replay key | `cycle9b-topk-crop-frame-20260602T041900` |
| Baseline job ID | `be4ba9ee-4786-48e9-8334-28feb237a1fb` |
| Baseline route | `behavior_ensemble_gaze_slice_topk` |
| Baseline batch window | `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=2` |
| Baseline Step 2 frame wall | `540.399 s` |
| Baseline Step 2 through pose upload | `767.589 s` |
| Baseline DB-completed FPS | `4.439` |
| Baseline behavior RTT mean | `84.865 ms` |
| Baseline average GPU utilization | `9.344 %` |

## Candidate

| Item | Value |
|---|---|
| Candidate replay key | `cycle9b-b4-maxframes4-20260602T175820Z` |
| Candidate job ID | `416efe8c-772c-442f-8e55-cf44c54fe261` |
| Candidate status | `completed` |
| Candidate route | `behavior_ensemble_gaze_slice_topk` |
| Candidate batch window | `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=4` |
| Candidate Step 2 frame wall | `512.445 s` |
| Candidate Step 2 through pose upload | `739.389 s` |
| Candidate DB-completed FPS | `4.572` |
| RSS guardrail | `< 4096 MiB` peak sampled Celery worker RSS |
| Sampled worker RSS peak | `1120.328 MiB` |

## Production Evidence

| Artifact | Path |
|---|---|
| Bench summary JSON | `backend/logs/bench_summary_20260602T210058.json` |
| GPU CSV | `backend/logs/gpu_monitor_bench_20260602T210058.csv` |
| Inference audit | `backend/data/videos/416efe8c-772c-442f-8e55-cf44c54fe261/inference_audit.json` |
| Telemetry session | `68c36ffd-c493-42d4-b27f-81aa47ba7fa6` |
| Evidence directory | `backend/logs/cycle9b-b4-maxframes4-20260602T175820Z/` |
| Baseline metrics JSON | `backend/logs/cycle9b-b4-maxframes4-20260602T175820Z/baseline_topk_metrics.json` |
| Baseline metrics Markdown | `backend/logs/cycle9b-b4-maxframes4-20260602T175820Z/baseline_topk_metrics.md` |
| Candidate metrics JSON | `backend/logs/cycle9b-b4-maxframes4-20260602T175820Z/batch_window_metrics.json` |
| Candidate metrics Markdown | `backend/logs/cycle9b-b4-maxframes4-20260602T175820Z/batch_window_metrics.md` |
| RSS TSV | `backend/logs/cycle9b-b4-maxframes4-20260602T175820Z/batch_window_rss.tsv` |
| Model agreement JSON | `backend/logs/cycle9b-b4-maxframes4-20260602T175820Z/model_agreement_320_topk_vs_candidate.json` |
| Model agreement Markdown | `backend/logs/cycle9b-b4-maxframes4-20260602T175820Z/model_agreement_320_topk_vs_candidate.md` |

## Before / After Metrics

| Metric | 320 Top-K baseline | B.4 candidate | Delta |
|---|---:|---:|---:|
| Status | `completed` | `completed` | pass |
| DB-completed FPS | `4.439` | `4.572` | `+3.00 %` |
| DB-completed elapsed | `1022.952 s` | `993.178 s` | `-2.91 %` |
| Step 2 frame wall | `540.399 s` | `512.445 s` | `-5.17 %` |
| Step 2 through pose upload | `767.589 s` | `739.389 s` | `-3.67 %` |
| Behavior RTT mean | `84.865 ms` | `99.251 ms` | `+16.95 %` |
| Behavior RTT p95 | `128.056 ms` | `128.731 ms` | `+0.53 %` |
| GPU avg util | `9.344 %` | `9.459 %` | `+1.23 %` |
| GPU peak util | `53.0 %` | `49.0 %` | `-7.55 %` |
| Peak VRAM | `16055 MiB` | `15725 MiB` | `-2.06 %` |
| Worker RSS peak | not sampled | `1120.328 MiB` | pass |

## Correctness

| Counter | 320 Top-K baseline | B.4 candidate | Delta |
|---|---:|---:|---:|
| Frames | `4541` | `4541` | `0` |
| Detections | `72762` | `72755` | `-0.01 %` |
| Bounding boxes | `72762` | `72755` | `-0.01 %` |
| Frame embeddings | `72596` | `72589` | `-0.01 %` |
| Student tracks | `53` | `47` | `-11.32 %` |
| `attention_tracking` boxes | `11781` | `11781` | `0` |
| `hand_raising` boxes | `8809` | `8806` | `-0.03 %` |
| `person_detection` boxes | `19162` | `19162` | `0` |
| `sitting_standing` boxes | `33010` | `33006` | `-0.01 %` |

The count-level parity is close, but the model-agreement probe showed tracking
and box identity instability. This agreement proxy is `F1@IoU0.5` against the
accepted 320 Top-K baseline, not human-labeled accuracy:

| Model | Baseline reference | Candidate agreement F1@IoU | Frame-count match | Baseline boxes | Candidate boxes |
|---|---:|---:|---:|---:|---:|
| `attention_tracking` | `100.000 %` | `24.531 %` | `98.635 %` | `11781` | `11781` |
| `hand_raising` | `100.000 %` | `26.648 %` | `99.185 %` | `8809` | `8806` |
| `person_detection` | `100.000 %` | `100.000 %` | `100.000 %` | `19162` | `19162` |
| `sitting_standing` | `100.000 %` | `17.217 %` | `97.666 %` | `33010` | `33006` |

## Decision

**NOT ACCEPTED.** The candidate improved Step 2 frame wall by `5.17 %`,
overall DB-completed FPS by `3.00 %`, and average GPU utilization by only
`1.23 %`. It also increased behavior RTT mean by `16.95 %`, lowered peak GPU
utilization, reduced persisted `StudentTrack` count from `53` to `47`, and
failed the model-agreement gate for three behavior models.

Do not enable `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=4` as the production
default. The benchmark helper remains useful for reproduction, but the accepted
runtime value is `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=2`.

## Reproduction

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_run_batch_window_benchmark.sh \
  --max-frames 4 \
  --rss-limit-mib 4096 \
  --tag cycle9b-b4-maxframes4-$(date -u +%Y%m%dT%H%M%SZ) \
  --timeout 7200
```
