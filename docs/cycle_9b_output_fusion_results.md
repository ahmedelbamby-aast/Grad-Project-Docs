# Cycle 9b Output Fusion Results

**Status:** **B.2.b exact server-side slice ACCEPTED** and **B.2.c exact
slice + Top-K ACCEPTED WITH CAVEAT.** The first standalone TensorRT
output-slice variant (`gaze2`) is **NOT ACCEPTED**.

This document records the Cycle 9b output-fusion candidates:

- `gaze_horizontal_gaze2_model`, a separate TensorRT plan that gathers legacy
  horizontal output channels `[0,1,2,3,8,9]` and returns `[6,2100]`.
- `gaze_horizontal_slice_model`, an exact server-side TensorRT gather that slices
  the already-executed legacy `gaze_horizontal_model.output0` inside Triton.
- `behavior_ensemble_gaze_slice_topk`, which keeps exact slicing and adds FP32
  Top-K adapter models after each behavior child so Python receives `[C,100]`
  instead of `[C,2100]`.

## Candidate

| Item | Value |
|---|---|
| Candidate commits | `c915ba78dc51f2ee804df2b0836027c6a4624a09`, parity fix `49932a22bfb429a74075e6952788af63eb007810` |
| Deployed SHA during proof | `49932a22bfb429a74075e6952788af63eb007810` |
| Production helper | `tools/prod/prod_enable_gaze_horizontal_gaze2.sh` |
| Variant child | `gaze_horizontal_gaze2_model` |
| Variant ensemble | `behavior_ensemble_gaze2` |
| Env flag | `GAZE_HORIZONTAL_HEAD_VARIANT=gaze2` |
| Rollback flag | `GAZE_HORIZONTAL_HEAD_VARIANT=coco80` |

## Validation Evidence

Production built the candidate TensorRT plan successfully on the RTX 5090 host:

| Build item | Result |
|---|---|
| Source export | `right_left.pt` -> `right_left.onnx`, input `320`, output `[1,84,2100]` |
| Slice export | `right_left.onnx` -> `right_left_gaze2.onnx` |
| TensorRT plan | `right_left_gaze2.engine`, FP16, `53 MB` |
| Triton deploy | `backend/models/triton_repository_cuda12/gaze_horizontal_gaze2_model/1/model.plan` |
| Startup validator | `behavior_ensemble_gaze2` validator passed |

Production parity probes:

| Item | Value |
|---|---|
| First probe artifact | `backend/logs/gaze_horizontal_gaze2_parity_20260601T230548.json` |
| Final rebuilt-engine probe artifact | `backend/logs/gaze_horizontal_gaze2_parity_20260601T231503_postrebuild.json` |
| Probe command | `backend/.venv/bin/python tools/prod/prod_gaze_horizontal_gaze2_parity.py --iterations 5 --batch-size 16 --json ...` |
| Compared tensors | legacy `gaze_horizontal_model.output0[:, [0,1,2,3,8,9], :]` vs candidate `gaze_horizontal_gaze2_model.output0` and `behavior_ensemble_gaze2.gaze_h_out` |
| Expected shape | `[16,6,2100]` |
| Candidate shape | `[16,6,2100]` |
| First max abs diff | `7.125` |
| Final rebuilt-engine max abs diff | `9.5` |
| Required tolerance | `1e-6` |
| Result | `fail` |

## Decision

**NOT ACCEPTED.**

The candidate passed shape/config/build validation, but it failed the
pre-benchmark tensor parity gate. A full `combined.mp4` benchmark was not run
because correctness evidence failed before the benchmark gate.

Mechanism observed: the candidate is a separate TensorRT plan, so TensorRT can
choose different FP16 tactics once the graph output is changed to a final
channel gather. That produces non-identical raw outputs even though the ONNX
graph mathematically selects the intended channels.

## Rollback

Production was rolled back immediately:

```bash
GAZE_HORIZONTAL_HEAD_VARIANT=coco80
MODEL_ROUTE_GAZE_HORIZONTAL_MODEL_NAME=gaze_horizontal_model
MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble
LPM_ENABLED=0
```

Triton and Celery workers were restarted after rollback. Production is back on
the legacy `behavior_ensemble` route.

## Next Recommendation

Do not accept the separate TensorRT output-slice plan as B.2.b. The next
output-fusion attempt must either:

1. slice the already-executed legacy horizontal output server-side so parity is
   exact, or
2. define and pass an explicit decoded-detection parity gate before any full
   benchmark.

B.1 compact postprocessing / BLS remains the cleaner path because it can also
host the future LPM probability logic before dense outputs leave Triton.

---

## B.2.b Follow-Up — Exact Server-Side Slice (ACCEPTED)

**Status:** **ACCEPTED.**

The follow-up candidate keeps the legacy `gaze_horizontal_model` TensorRT plan
unchanged and inserts `gaze_horizontal_slice_model` after its dense output inside
Triton:

| Item | Value |
|---|---|
| Variant flag | `GAZE_HORIZONTAL_HEAD_VARIANT=slice` |
| Direct slice model | `gaze_horizontal_slice_model`, input `dense_input [84,2100]`, output `output0 [6,2100]` |
| Full behavior ensemble | `behavior_ensemble_gaze_slice` |
| Standalone fallback adapter | `gaze_horizontal_slice_adapter` |
| Prod enable helper | `tools/prod/prod_enable_gaze_horizontal_slice.sh` |
| Prod parity probe | `tools/prod/prod_gaze_horizontal_slice_parity.py` |
| Local validation | `18 passed`; py_compile and shell syntax passed |
| Deployed SHA | `ca69f02a8ceb214d7ef55cd2ae4b7ec75549c257` |
| Replay key | `cycle9b-exactslice-crop-frame-20260601T233211` |
| Job ID | `7933c1e5-a970-47a3-81c5-0c9bd01bd332` |
| Telemetry session | `c0d59cb2-721e-419d-80a4-b2b1e4bffaa6` |
| Bench summary | `backend/logs/bench_summary_20260602T023450.json` |
| GPU CSV | `backend/logs/gpu_monitor_bench_20260602T023450.csv` |
| Inference audit | `backend/data/videos/7933c1e5-a970-47a3-81c5-0c9bd01bd332/inference_audit.json` |
| Post-benchmark parity | `backend/logs/gaze_horizontal_slice_parity_20260601T235623_postbench.json`, `max_abs_diff=0.0` |

### Production Result

| Metric | Cycle 9 Dense Ensemble | Exact Slice | Delta |
|---|---:|---:|---:|
| Step 2 wall | `858.1 s` | `573.927 s` | `-33.1 %` |
| Step 2 FPS | `5.29` | `7.912` | `+49.6 %` |
| DB-completed elapsed | `1110.7 s` | `~1054 s` | `-5.1 %` |
| DB-completed FPS | `4.09` | `4.307` | `+5.3 %` |
| Behavior RTT mean | `107.9 ms` | `91.470 ms` | `-15.2 %` |
| Behavior RTT p95 | `173.9 ms` | `146.015 ms` | `-16.0 %` |
| Horizontal dense output / frame | `~11.4 MB` | `~0.82 MB` | `~93 %` less |
| Avg GPU utilization | `9.36 %` | `9.595 %` | `+0.235 pp` |
| Peak GPU utilization | `43 %` | `45 %` | `+2 pp` |

Correctness stayed within measurement noise: `4541` frames persisted,
`attention_tracking` stayed at `11776`, `person_detection` stayed at `19162`,
detections and bounding boxes were `-2` (`-0.0027 %`), frame embeddings were
`-2`, and StudentTrack count stayed `53`.

### Decision

**ACCEPTED.** The candidate pulled the intended lever, dense output bytes, and
the production benchmark showed a `33.1 %` Step 2 wall reduction plus exact raw
tensor parity. Production remains on:

```bash
GAZE_HORIZONTAL_HEAD_VARIANT=slice
MODEL_ROUTE_GAZE_HORIZONTAL_MODEL_NAME=gaze_horizontal_slice_adapter
MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble_gaze_slice
LPM_ENABLED=0
```

---

## B.2.c Follow-Up — Exact Slice + Top-K Packing (ACCEPTED WITH CAVEAT)

**Status:** **ACCEPTED WITH CAVEAT.**

This candidate keeps the accepted exact-slice route and adds Top-K packing after
all four behavior children:

| Item | Value |
|---|---|
| Variant flags | `GAZE_HORIZONTAL_HEAD_VARIANT=slice`, `TRITON_BEHAVIOR_TOP_K_ENABLED=1`, `TRITON_BEHAVIOR_TOP_K_VALUE=100` |
| Full behavior ensemble | `behavior_ensemble_gaze_slice_topk` |
| Top-K adapters | `posture_topk_model`, `gaze_horizontal_slice_topk_model`, `gaze_vertical_topk_model`, `gaze_depth_topk_model` |
| Prod enable helper | `tools/prod/prod_enable_behavior_topk.sh` |
| Prod parity probe | `tools/prod/prod_behavior_topk_parity.py` |
| Deployed SHA | `9f879affeb4478e63a09276b10a2d64844bcbc44` |
| Replay key | `cycle9b-topk-crop-frame-20260602T041900` |
| Job ID | `be4ba9ee-4786-48e9-8334-28feb237a1fb` |
| Telemetry session | `c4710435-4ec0-49e1-8ffb-60012fa878c9` |
| Bench summary | `backend/logs/bench_summary_20260602T042139.json` |
| GPU CSV | `backend/logs/gpu_monitor_bench_20260602T042139.csv` |
| Inference audit | `backend/data/videos/be4ba9ee-4786-48e9-8334-28feb237a1fb/inference_audit.json` |
| Decoded parity | `backend/logs/behavior_topk_parity_20260602T011830Z_fp32.json`, `failed_count=0`, `max_score_diff=0.0`, `max_box_diff=0.0` |

FP16 Top-K adapters failed decoded parity before the full benchmark
(`max_score_diff≈2.4e-4`, `max_box_diff≈0.138`), so production built the Top-K
adapters in FP32.

### Production Result

| Metric | Exact Slice Baseline | Exact Slice + Top-K | Delta |
|---|---:|---:|---:|
| Step 2 frame wall | `573.927 s` | `540.399 s` | `-5.84 %` |
| Step 2 through pose upload | `799.345 s` | `767.589 s` | `-3.97 %` |
| Audit `run.complete` wall | `865.419 s` | `833.810 s` | `-3.65 %` |
| DB-completed elapsed | `1052.281 s` | `1022.952 s` | `-2.79 %` |
| DB-completed FPS | `4.307` | `4.429` | `+2.8 %` |
| Behavior RTT mean | `91.470 ms` | `84.865 ms` | `-7.22 %` |
| Behavior RTT p95 | `146.072 ms` | `128.138 ms` | `-12.28 %` |
| Behavior output / frame | `~6.85 MB` | `~0.33 MB` | `~95 %` less |
| Avg GPU utilization | `9.595 %` | `9.3 %` | `-0.295 pp` |
| Peak GPU utilization | `45 %` | `53 %` | `+8 pp` |

Correctness stayed within the established tolerance: `4541` frames persisted,
StudentTrack count stayed `53`, `person_detection` boxes stayed `19162`,
`attention_tracking` boxes moved `11776 → 11781` (`+0.0425 %`), total
detections moved `72747 → 72762` (`+0.0206 %`), and the decoded parity probe
passed exactly.

### Decision

**ACCEPTED WITH CAVEAT.** The candidate pulled the same named lever as B.2.b,
dense output bytes, and production proved lower Step 2 wall, lower behavior RTT,
and correctness parity. The caveat is that average GPU utilization did not
improve. Production remains on:

```bash
GAZE_HORIZONTAL_HEAD_VARIANT=slice
TRITON_BEHAVIOR_TOP_K_ENABLED=1
TRITON_BEHAVIOR_TOP_K_VALUE=100
MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble_gaze_slice_topk
LPM_ENABLED=0
```

Detailed evidence is also recorded in
[`docs/cycle_9b_topk_anchor_packing_results.md`](cycle_9b_topk_anchor_packing_results.md)
and [`docs/production_inference_benchmark.md`](production_inference_benchmark.md)
§19.

The broader B.2 multi-approach checklist is still not fully closed because
standalone B.2.a Top-K without the accepted slice was not separately
benchmarked. It is lower priority now: the production baseline has already moved
to B.2.c, and the next cycle should target GPU occupancy / server-side execution
rather than response-byte trimming alone.
