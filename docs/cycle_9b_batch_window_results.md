# Cycle 9b B.4 Batch Window Results

**Last updated:** 2026-06-02

**Status:** **STAGED — PRODUCTION BENCHMARK PENDING.**

This file records the Cycle 9b B.4 result once the production benchmark has run.
Until the tables below are filled with production evidence, the accepted runtime
baseline remains Cycle 9b exact slice + Top-K at
`TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=2`.

## Baseline

| Item | Value |
|---|---|
| Baseline replay key | `cycle9b-topk-crop-frame-20260602T041900` |
| Baseline job ID | `be4ba9ee-4786-48e9-8334-28feb237a1fb` |
| Baseline route | `behavior_ensemble_gaze_slice_topk` |
| Baseline batch window | `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=2` |
| Baseline Step 2 frame wall | `540.399 s` |
| Baseline DB-completed FPS | `4.429` |

## Candidate

| Item | Value |
|---|---|
| Candidate replay key | pending |
| Candidate job ID | pending |
| Candidate route | `behavior_ensemble_gaze_slice_topk` |
| Candidate batch window | `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=4` |
| RSS guardrail | `< 4096 MiB` peak sampled Celery worker RSS |

## Production Evidence

| Artifact | Path |
|---|---|
| Bench summary JSON | pending |
| GPU CSV | pending |
| Inference audit | pending |
| RSS TSV | pending |
| Metrics JSON | pending |
| Metrics Markdown | pending |
| Model agreement JSON | pending |
| Model agreement Markdown | pending |

## Before / After Metrics

| Metric | 320 Top-K baseline | B.4 candidate | Delta |
|---|---:|---:|---:|
| Status | `completed` | pending | pending |
| DB-completed FPS | `4.429` | pending | pending |
| DB-completed elapsed | `1022.952 s` | pending | pending |
| Step 2 frame wall | `540.399 s` | pending | pending |
| Step 2 through pose upload | `767.589 s` | pending | pending |
| Behavior RTT mean | `84.865 ms` | pending | pending |
| Behavior RTT p95 | `128.056 ms` | pending | pending |
| GPU avg util | `9.344 %` | pending | pending |
| GPU peak util | `53.0 %` | pending | pending |
| Worker RSS peak | pending | pending | pending |

## Correctness

| Counter | 320 Top-K baseline | B.4 candidate | Delta |
|---|---:|---:|---:|
| Frames | `4541` | pending | pending |
| Detections | `72762` | pending | pending |
| Bounding boxes | `72762` | pending | pending |
| Frame embeddings | `72596` | pending | pending |
| Student tracks | `53` | pending | pending |
| `attention_tracking` boxes | `11781` | pending | pending |
| `hand_raising` boxes | `8809` | pending | pending |
| `person_detection` boxes | `19162` | pending | pending |
| `sitting_standing` boxes | `33010` | pending | pending |

## Decision

Pending. This cycle is not complete and not accepted until the production
benchmark evidence is recorded here and the cycle map is updated.
