# Cycle 9b B.1 Compact Postprocessing Results

**Last updated:** 2026-06-02

**Status:** **PHASE A PROBE COMPLETE — NO OPTIMIZATION IMPLEMENTED OR ACCEPTED.**

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
| Probe JSON | `backend/logs/cycle9b_b1_decode_cost_topk_20260602T185559Z.json` |
| Probe Markdown | `backend/logs/cycle9b_b1_decode_cost_topk_20260602T185559Z.md` |
| Probe mode | accepted Top-K baseline real crops |
| Replay key | `cycle9b-topk-crop-frame-20260602T041900` |
| Total sampled crops | `340` |
| Batches | `20` |
| Batch size | `17` |
| Mean behavior RTT with parse | `62.082 ms` |
| Mean gRPC/Triton wait | `59.651 ms` |
| Mean serialization time | `2.317 ms` |
| Mean `as_numpy` time | `0.114 ms` |
| Mean Python decode/NMS time | `3.125 ms/batch` |
| Decode ms per crop | `0.183823 ms` |
| Behavior output bytes per crop | `19,200 bytes` |
| Estimated compact bytes per crop | `11.2 bytes` |
| Estimated output reduction | `99.942 %` |

Per-model decode/NMS timing from the accepted Top-K route:

| Model | Decode mean ms/batch | Decode p95 ms/batch | Boxes mean/batch |
|---|---:|---:|---:|
| `posture_detection` | `1.475` | `2.488` | `5.600` |
| `gaze_horizontal` | `0.498` | `0.885` | `0.000` |
| `gaze_vertical` | `0.665` | `1.026` | `2.050` |
| `gaze_depth` | `0.487` | `0.751` | `0.300` |

Upper-bound implication: the accepted production benchmark has `3597`
`behavior_ensemble_gaze_slice_topk` calls. At `3.125 ms` decode/NMS per
17-crop call, the observed client decode/NMS work is about `11.24 s`, or only
`~2.08 %` of the accepted `540.399 s` Step 2 frame wall. `as_numpy` parsing is
about `0.41 s` across the same call count. A compact backend may still be useful
for architecture cleanliness and future LPM placement, but Phase A evidence says
client-side Top-K decode/NMS alone cannot plausibly satisfy the `>=10 %` Step 2
acceptance gate.

## Decision

No B.1 implementation is selected yet. B.1 remains unimplemented and
unaccepted; selecting Python BLS, C++ custom backend, or TRT plugin still
requires a real production candidate benchmark.
