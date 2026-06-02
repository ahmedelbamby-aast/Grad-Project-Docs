# Cycle 9b B.1 Compact Postprocessing Results

**Last updated:** 2026-06-02

**Status:** **PENDING — NO PRODUCTION BENCHMARK RUN, NO OPTIMIZATION ACCEPTED.**

This file will record the Cycle 9b B.1 compact-postprocessing comparison matrix.
It is intentionally empty of candidate metrics until a production benchmark is
run on the Linux RTX 5090 server.

## Baseline

| Item | Value |
|---|---|
| Baseline replay key | `cycle9b-topk-crop-frame-20260602T041900` |
| Baseline job ID | `be4ba9ee-4786-48e9-8334-28feb237a1fb` |
| Baseline route | `behavior_ensemble_gaze_slice_topk` |
| Baseline input size | `320x320` |
| Baseline batch window | `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=2` |
| Baseline Step 2 frame wall | `540.399 s` |
| Baseline DB-completed FPS | `4.439` |
| Baseline behavior RTT mean | `84.865 ms` |

## Candidate Matrix

| Metric | Baseline Top-K | B.1.a Python BLS | B.1.b C++ custom | B.1.c TRT plugin |
|---|---:|---:|---:|---:|
| Production replay key | `cycle9b-topk-crop-frame-20260602T041900` | pending | pending | pending |
| Job ID | `be4ba9ee-4786-48e9-8334-28feb237a1fb` | pending | pending | pending |
| Status | `completed` | pending | pending | pending |
| Step 2 frame wall | `540.399 s` | pending | pending | pending |
| Step 2 wall delta | baseline | pending | pending | pending |
| DB-completed FPS | `4.439` | pending | pending | pending |
| Behavior RTT mean | `84.865 ms` | pending | pending | pending |
| GPU avg util | `9.344 %` | pending | pending | pending |
| Detection rows | `72762` | pending | pending | pending |
| BBox rows | `72762` | pending | pending | pending |
| Student tracks | `53` | pending | pending | pending |
| Model agreement F1@IoU0.5 | `100.000 %` reference | pending | pending | pending |

## Decode-Cost Probe

The first B.1 measurement artifact is
`tools/prod/prod_probe_behavior_decode_cost.py`. It must be run on production
before any compact backend implementation is selected.

| Metric | Accepted Top-K probe |
|---|---:|
| Probe JSON | pending |
| Probe Markdown | pending |
| Total sampled crops | pending |
| Mean behavior RTT with parse | pending |
| Mean `as_numpy` time | pending |
| Mean Python decode/NMS time | pending |
| Decode ms per crop | pending |
| Behavior output bytes per crop | pending |
| Estimated compact bytes per crop | pending |
| Estimated output reduction | pending |

## Decision

Pending. B.1 remains unimplemented and unaccepted.
