# Cycle 9b B.1 Compact Postprocessing Results

**Last updated:** 2026-06-02

**Status:** **PROBE-ONLY EVIDENCE — NO DECISION. PRODUCTION BENCHMARK REQUIRED.**

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

Hypothesis-only upper-bound implication: the accepted production benchmark has `3597`
`behavior_ensemble_gaze_slice_topk` calls. At `3.125 ms` decode/NMS per
17-crop call, the observed client decode/NMS work is about `11.24 s`, or only
`~2.08 %` of the accepted `540.399 s` Step 2 frame wall. `as_numpy` parsing is
about `0.41 s` across the same call count. This arithmetic is not a decision:
it only defines a hypothesis that a future production Linux benchmark must test.

## Probe Context Table — No Decision Authority

| Decision question | Evidence from last run | Decision impact |
|---|---|---|
| Is this a production candidate benchmark? | No. The run is a component probe over accepted Top-K crops; no compact backend, env change, or model route change was deployed. | Status is `PROBE_ONLY`; B.1 cannot be accepted, rejected, skipped, closed, or deprioritized from this run. |
| What accepted baseline anchors the comparison? | `cycle9b-topk-crop-frame-20260602T041900`, job `be4ba9ee-4786-48e9-8334-28feb237a1fb`, Step 2 wall `540.399 s`, DB FPS `4.439`, behavior RTT mean `84.865 ms`. | All B.1 candidates must beat this baseline and preserve correctness. |
| What component did the probe isolate? | Client response parse plus Python `_decode_yolo_output0`/NMS after `behavior_ensemble_gaze_slice_topk`. | The probe only bounds the value of moving Top-K decode/NMS out of Python. |
| How large is the removable measured component? | `3.125 ms` decode/NMS per 17-crop batch, or about `11.24 s` across `3597` accepted behavior calls. | Pure decode/NMS removal is bounded at `~2.08 %` of accepted Step 2 wall. |
| Does the measured component satisfy the target gate? | Gate satisfaction cannot be evaluated by a component probe. The cycle gate requires `>=10 %` Step 2 wall reduction plus correctness parity in a real production benchmark. | `NO_DECISION_PRODUCTION_BENCHMARK_REQUIRED`. |
| Why do the measured results look like this? | Top-K already reduced behavior output to `19,200 bytes/crop`; remaining Python decode/NMS is small. The dominant measured time is gRPC/Triton wait: `59.651 ms` of `62.082 ms` RTT-with-parse. | The earlier dense-byte bottleneck is mostly gone; the remaining limiter is wait/server execution rather than Python decode. |
| What bottleneck hypothesis should the benchmark test? | `infer_wait_ms`/server-side child execution and scheduling dominate the sample, while `as_numpy` parse is only `0.114 ms`. | The next benchmark must prove whether any B.1 candidate reduces wait/server execution, not only compact already-Top-K outputs. |
| Can B.1 be skipped or accepted now? | No full candidate benchmark exists. | B.1 remains open; only a real candidate benchmark can accept, reject, or close it. |

## Required End-to-End Re-Run

The probe has no decision authority. Repeat B.1 evidence through the full
production Linux path with:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_run_b1_decode_cost_full_benchmark.sh \
  --video "/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4" \
  --tag cycle9b-b1-fullbench-$(date -u +%Y%m%dT%H%M%SZ)
```

That wrapper runs the canonical full benchmark first, collects DB/GPU/RTT and
model-agreement metrics, then runs the decode-cost probe against the fresh
replay key. Only the documented full benchmark can support a B.1 decision.

## Production Full-Benchmark Repeat

The required end-to-end repeat was executed on the production Linux RTX 5090
server. It kept the accepted 320 exact-slice + Top-K route and did not deploy a
B.1 compact backend, so it records benchmark evidence only.

| Item | Value |
|---|---|
| Wrapper | `tools/prod/prod_run_b1_decode_cost_full_benchmark.sh` |
| Replay key | `cycle9b-b1-fullbench-20260602T192344Z` |
| Job ID | `00e0e1da-44b6-4198-ad39-39fd853e4e18` |
| Deployed SHA | `7556e84` |
| Video | `/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4` |
| Pipeline | `crop_frame`, accepted 320 exact-slice + Top-K |
| Final status | `completed`, `4541/4541` frames |
| Evidence directory | `backend/logs/cycle9b-b1-fullbench-20260602T192344Z/` |
| Decision status | `NO_DECISION_BENCHMARK_RECORDED` |

| Metric | Accepted Top-K baseline | B.1 full repeat | Delta |
|---|---:|---:|---:|
| Step 2 frame wall | `540.399 s` | `540.748 s` | `+0.06 %` |
| DB-completed FPS | `4.439` | `4.346` | `-2.11 %` |
| DB-completed elapsed | `1022.952 s` | `1044.988 s` | `+2.15 %` |
| Behavior RTT mean | `84.865 ms` | `85.201 ms` | `+0.40 %` |
| Behavior RTT p95 | `128.056 ms` | `128.792 ms` | `+0.57 %` |
| GPU avg util | `9.344 %` | `11.962 %` | `+28.02 %` |
| GPU peak util | `53.000 %` | `51.000 %` | `-3.77 %` |
| Peak VRAM | `16055 MiB` | `15725 MiB` | `-2.06 %` |
| Detection rows | `72762` | `72750` | `-0.02 %` |
| BBox rows | `72762` | `72750` | `-0.02 %` |
| Embedding rows | `72596` | `72584` | `-0.02 %` |
| Student tracks | `53` | `53` | `0.00 %` |

Model agreement against the accepted baseline remained within repeat-run noise:

| Model | Agreement F1@IoU0.5 | Count delta |
|---|---:|---:|
| `attention_tracking` | `99.724 %` | `-0.03 %` |
| `hand_raising` | `99.790 %` | `-0.08 %` |
| `person_detection` | `100.000 %` | `0.00 %` |
| `sitting_standing` | `99.979 %` | `-0.01 %` |

The post-benchmark decode probe ran against all sampled person crops from the
fresh completed job:

| Decode metric | Value |
|---|---:|
| Batches | `1127` |
| Total crops | `19146` |
| Mean RTT with parse | `45.160 ms` |
| Mean infer wait | `42.704 ms` |
| Mean serialization | `2.392 ms` |
| Mean `as_numpy` parse | `0.064 ms` |
| Mean decode/NMS | `2.040 ms/batch` |
| Decode/NMS per crop | `0.120 ms` |
| Total behavior output bytes | `367,603,200` |
| Estimated compact bytes | `222,936` |
| Estimated byte reduction | `99.939 %` |

### Decision Authority Result

This full production run still does not accept or reject B.1 because it did not
deploy a compact-postprocessing candidate. It proves the benchmark path and
refreshes the bottleneck measurement on `combined.mp4`: the accepted route's
Step 2 wall and behavior RTT are effectively unchanged, while direct all-crop
decode/NMS is a small measured component. A future B.1 candidate must change
code/config, run this same production benchmark, and then compare against the
accepted 320 Top-K baseline before any acceptance or non-acceptance decision.
