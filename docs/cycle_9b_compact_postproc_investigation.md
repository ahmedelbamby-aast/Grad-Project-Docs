# Cycle 9b B.1 Compact Postprocessing Investigation

**Last updated:** 2026-06-02

**Status:** **INVESTIGATION OPEN — NO CODE IMPLEMENTED, NO OPTIMIZATION ACCEPTED.**

This document starts Cycle 9b B.1, the server-side compact postprocessing
candidate. It exists because the current accepted baseline has changed since
the original B.1 idea was written: exact slice + Top-K already removed most
dense behavior output bytes. B.1 must therefore be evaluated against the current
Top-K baseline, not against the older Cycle 9 dense-output baseline.

## Problem Statement

The accepted production baseline is Cycle 9b B.2.c exact slice + Top-K:

| Item | Value |
|---|---|
| Replay key | `cycle9b-topk-crop-frame-20260602T041900` |
| Job ID | `be4ba9ee-4786-48e9-8334-28feb237a1fb` |
| Behavior route | `behavior_ensemble_gaze_slice_topk` |
| Behavior input size | `320x320` |
| Batch window | `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=2` |
| Step 2 frame wall | `540.399 s` |
| Step 2 through pose upload | `767.589 s` |
| DB-completed elapsed | `1022.952 s` |
| DB-completed FPS | `4.439` |
| Behavior ensemble RTT mean / p95 | `84.865 ms / 128.056 ms` |
| Avg / peak GPU util | `9.344 % / 53.0 %` |

The current path still returns behavior tensors to Python and decodes them in
`backend/apps/video_analysis/tasks.py:_decode_yolo_output0`, but those tensors
are already Top-K packed:

| Output path | Structural bytes per average 17-crop frame |
|---|---:|
| Pre-Top-K dense behavior outputs | `~17.1 MB/frame` |
| Accepted Top-K behavior outputs | `~0.33 MB/frame` |
| Ideal compact detections | `~2 KB/frame` |

B.1 can still remove the remaining Python-side decode/NMS work and almost all
remaining behavior output bytes, but the absolute byte-saving opportunity is
now much smaller than the original B.1 premise.

## Root Cause Being Tested

The current accepted flow decodes behavior outputs client-side:

```text
Triton behavior_ensemble_gaze_slice_topk
  -> Top-K tensors returned over gRPC
  -> Python response parsing
  -> tasks.py:_decode_task_response(...)
  -> tasks.py:_decode_yolo_output0(...)
  -> Python NMS
  -> DetectionBox objects
```

The B.1 hypothesis is:

```text
Triton behavior graph
  -> server-side decode/NMS
  -> compact detections only
  -> Python parses compact tuples
  -> DetectionBox objects
```

The lever is **client-side postprocessing and remaining output transfer**, not
behavior input upload, model child compute, or frame batching.

## Current Evidence

### Production Baseline Metrics

Refreshed from production with:
`tools/prod/prod_collect_benchmark_metrics.py --replay-key cycle9b-topk-crop-frame-20260602T041900`.

| Metric | Value |
|---|---:|
| Processed frames | `4541/4541` |
| Step 2 frame wall | `540.399 s` |
| Step 2 through pose upload | `767.589 s` |
| Step 2 aggregate inference timing | `451.744 s` |
| Step 2 aggregate postprocess timing | `889.823 s` |
| Behavior ensemble model calls | `3597` |
| Behavior ensemble RTT mean | `84.865 ms` |
| Behavior ensemble RTT p95 | `128.056 ms` |
| GPU avg util | `9.344 %` |
| GPU peak util | `53.000 %` |
| Detection rows | `72762` |
| BBox rows | `72762` |
| Frame embeddings | `72596` |
| Student tracks | `53` |

The aggregate `postprocess_s` field is not additive with Step 2 wall because
stages overlap and accumulate per-frame work, but it is enough to justify
instrumenting the decode/NMS sub-stage before implementing a backend.

### Code-Path Evidence

| Code path | Evidence |
|---|---|
| Decode/NMS implementation | `backend/apps/video_analysis/tasks.py:_decode_yolo_output0` |
| Caller in crop-frame behavior path | `backend/apps/video_analysis/tasks.py:_decode_task_response` |
| Current accepted model route | `MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble_gaze_slice_topk` in production `.env` |
| Top-K flags | `TRITON_BEHAVIOR_TOP_K_ENABLED=1`, `TRITON_BEHAVIOR_TOP_K_VALUE=100` |
| Compact parser gap | No `BEHAVIOR_COMPACT_BACKEND` setting or compact `[num_dets, 6]` parser exists yet |
| Decode-cost probe | `tools/prod/prod_probe_behavior_decode_cost.py` |

### Triton Backend Capability Evidence

Pinned production Triton install:
`/home/bamby/services/triton_build_r2502/tritonserver/install/backends`

```text
distributed_addsub
dyna_sequence
implicit_state
iterative_sequence
query
sequence
tensorrt
```

The pinned production install does **not** expose a `python` backend or a
generic `custom` backend today. A different non-pinned Triton tree exists with
`python`, `pytorch`, `onnxruntime`, and `tensorrt` backends, but switching to it
is a production runtime change and must not be done implicitly.

## Candidate Approaches

| ID | Candidate | Current feasibility | Expected gain source | Main risk |
|---|---|---|---|---|
| B.1.a | Triton Python BLS backend | Blocked until pinned Triton is rebuilt with Python backend or runtime is intentionally switched | Remove client decode/NMS and return compact detections; can host future LPM Phase 2 | Server Python overhead may eat the gain; backend rebuild/switch risk |
| B.1.b | C++ custom backend | Blocked until a custom backend build path exists in production | Lower-overhead server-side decode/NMS | Highest engineering effort; custom binary lifecycle |
| B.1.c | TensorRT EfficientNMS plugin in behavior engines | Feasible only through engine re-export/rebuild | GPU-side NMS with no extra backend scheduling | Accuracy/parity risk; output contract changes per engine |

## Expected Gain

The old B.1 estimate was based on removing `~17.1 MB/frame` of dense behavior
outputs. Against the current accepted Top-K route, the remaining output transfer
is closer to `~0.33 MB/frame`; the expected gain must be proven by a production
micro-benchmark before any full implementation is accepted.

Minimum proof needed before selecting an implementation:

1. Measure Python `_decode_yolo_output0` wall time per behavior model and per
   frame on the accepted Top-K route.
2. Measure compact tuple parsing overhead locally and in production.
3. Verify whether a Python backend rebuild/switch changes Triton stability,
   startup time, model readiness, and TensorRT execution timing.
4. If B.1.c is attempted, run exact decoded parity on the same crop batches
   before full production benchmark.

### Measurement Harness

The first B.1 code artifact is a probe only:

```bash
cd /home/bamby/grad_project
TS="$(date -u +%Y%m%dT%H%M%SZ)"
PYTHONPATH=backend DJANGO_SETTINGS_MODULE=config.settings APP_ENV=prod \
  backend/.venv/bin/python tools/prod/prod_probe_behavior_decode_cost.py \
    --mode real \
    --replay-key cycle9b-topk-crop-frame-20260602T041900 \
    --video "/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4" \
    --batches 20 \
    --batch-size 17 \
    --output "backend/logs/cycle9b_b1_decode_cost_topk_${TS}.json" \
    --markdown-output "backend/logs/cycle9b_b1_decode_cost_topk_${TS}.md"
```

It samples real `person_detection` boxes from the accepted Top-K baseline job,
recreates 320x320 crop tensors, calls `behavior_ensemble_gaze_slice_topk`,
times gRPC serialization/wait/`as_numpy`, then times the existing
`_decode_yolo_output0` path per behavior model. The probe estimates compact
bytes as `decoded_boxes * 6 floats * 4 bytes`. It does not change production
env, model routing, Triton configs, or database rows.

### Production Probe Result

The Phase A probe ran on production at deployed SHA `cd6780d0`:

| Item | Value |
|---|---|
| Probe JSON | `backend/logs/cycle9b_b1_decode_cost_topk_20260602T185559Z.json` |
| Probe Markdown | `backend/logs/cycle9b_b1_decode_cost_topk_20260602T185559Z.md` |
| Mode | accepted Top-K baseline real crops |
| Replay key | `cycle9b-topk-crop-frame-20260602T041900` |
| Sampled crops | `340` |
| Batches | `20` |
| Mean RTT with parse | `62.082 ms` |
| Mean gRPC/Triton wait | `59.651 ms` |
| Mean `as_numpy` parse | `0.114 ms` |
| Mean Python decode/NMS | `3.125 ms/batch` |
| Decode/NMS per crop | `0.183823 ms` |
| Output bytes per crop | `19,200` |
| Estimated compact bytes per crop | `11.2` |
| Estimated output reduction | `99.942 %` |

For the accepted production benchmark, telemetry recorded `3597` behavior
ensemble calls. Multiplying by the measured `3.125 ms/batch` decode/NMS cost
gives an approximate `11.24 s` client decode/NMS upper bound. That is only
`~2.08 %` of the accepted `540.399 s` Step 2 frame wall. The Phase A evidence
therefore does not justify selecting B.1 as a pure client decode/NMS removal
candidate for the `>=10 %` Step 2 acceptance gate.

## Risk Assessment

| Risk | Level | Mitigation |
|---|---|---|
| Python backend not available in pinned Triton | High | Treat B.1.a as blocked until backend availability is proven by a controlled deploy |
| Top-K already removed most output bytes | Medium | Measure decode/NMS CPU wall before coding; do not assume legacy B.1 byte savings |
| Compact contract changes persisted boxes | High | Require per-model F1@IoU0.5 and count parity versus accepted Top-K baseline |
| Engine/plugin rebuild changes class/box semantics | High | Gate B.1.c behind exact decoded parity and full production benchmark |
| LPM Phase 2 coupling expands scope | Medium | Keep B.1 focused on compact behavior detections; LPM migration is a later cycle |

## Rollback Strategy

Every B.1 implementation must keep the current dense/Top-K path compiled and
route-controlled:

```env
BEHAVIOR_COMPACT_BACKEND=
```

Rollback must restore:

```text
TRITON_CROP_BEHAVIOR_INPUT_SIZE=320
GAZE_HORIZONTAL_HEAD_VARIANT=slice
MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble_gaze_slice_topk
TRITON_BEHAVIOR_TOP_K_ENABLED=1
TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=2
LPM_ENABLED=0
```

## Acceptance Criteria

B.1 is not accepted until all of the following are true:

1. Investigation and decode/NMS micro-benchmark evidence are recorded with
   `tools/prod/prod_probe_behavior_decode_cost.py`.
2. Implementation is flag-gated by `BEHAVIOR_COMPACT_BACKEND`.
3. The compact-output parser preserves per-model boxes versus the accepted
   Top-K baseline.
4. Production benchmark on `combined.mp4` completes on the RTX 5090.
5. Before/after tables are recorded in
   `docs/cycle_9b_compact_postproc_results.md` and
   `docs/production_inference_benchmark.md`.
6. Correctness gates pass: frame rows, detections, bboxes, embeddings,
   StudentTracks, and per-model F1@IoU0.5 agreement remain within tolerance.
7. The selected approach improves Step 2 wall by at least `10 %` versus the
   accepted Top-K baseline or is explicitly rejected.

## Decision

No B.1 implementation is selected yet. The next safe step is a measurement
harness for client-side behavior decode/NMS cost on the accepted Top-K route,
followed by a backend feasibility check for B.1.a/B.1.b/B.1.c.
