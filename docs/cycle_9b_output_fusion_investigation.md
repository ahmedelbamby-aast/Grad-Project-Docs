# Cycle 9b Output Fusion Investigation

**Status:** PHASE A/B COMPLETE - IMPLEMENTATION NOT STARTED.

This document is the pre-code investigation and hypothesis for Cycle 9b
B.2.b: a `gaze_horizontal_model` narrow-output candidate. It does not mark any
optimization accepted.

## Problem Statement

Cycle 9 reduced app-side behavior calls from four calls to one
`behavior_ensemble` call, but Step 2 wall did not improve:

| Metric | Cycle 8 | Cycle 9 | Delta |
|---|---:|---:|---:|
| Step 2 wall | `852.8 s` | `858.1 s` | `+0.6 %` |
| DB-completed FPS | `3.46` | `4.09` | `+18.1 %` |
| Behavior crop app calls | `14 391` | `3 597` | `-75.0 %` |

The Cycle 9 post-mortem proves request-count reduction alone is no longer the
right lever. The four child TensorRT models still execute, and Python still
receives dense YOLO tensors.

## Root Cause Evidence

Cycle 9b Step-1 production measurement identified `gaze_horizontal_model` as
the dominant behavior child:

| Model | Direct server avg / exec |
|---|---:|
| `posture_model` | `12.133 ms` |
| `gaze_horizontal_model` | `16.058 ms` |
| `gaze_vertical_model` | `11.759 ms` |
| `gaze_depth_model` | `11.909 ms` |

The same model has the widest behavior output:

| Model | Dense output | Bytes / crop |
|---|---:|---:|
| `gaze_horizontal_model` | `[84, 2100]` | `~689 KB` |
| Other behavior models | `[14, 2100]` | `~115 KB` |

The pipeline only consumes horizontal-gaze class IDs `4` and `5`
(`right_left` class filter). Therefore the horizontal model returns:

- 4 bbox channels,
- 80 class channels,
- but only 2 class channels are consumed.

## Feasibility Finding

The existing `right_left.onnx` output is:

```text
output0: [batch, 84, anchors]
```

The model metadata still names the 80 classes as COCO classes, while the
runtime maps class IDs `4` and `5` to `right` / `left`. A true two-class head
retrain is out of scope for this cycle. The feasible low-risk implementation is
an ONNX output-slice variant:

```text
legacy output channels:  [0,1,2,3, 4..83]
gaze2 output channels:   [0,1,2,3, 8,9]
                         bbox + legacy classes 4 and 5
```

The compact decoder must remap compact class IDs back to legacy IDs:

| Compact class ID | Legacy class ID | Label |
|---:|---:|---|
| `0` | `4` | `right` |
| `1` | `5` | `left` |

This preserves DB semantics and makes correctness parity measurable.

## Hypothesis

**Named lever:** dense output bytes.

Creating a `gaze_horizontal_gaze2_model` TensorRT plan whose output is
`[6,2100]` instead of `[84,2100]`, plus a matching
`behavior_ensemble_gaze2` wrapper, should reduce horizontal-gaze response bytes
by about `92.9 %` without changing the model input, crop preprocessing, or
class semantics.

Expected per-frame behavior dense-output reduction at the observed ~17 crops:

| Quantity | Legacy | Candidate |
|---|---:|---:|
| Horizontal bytes / crop | `84*2100*4 = 705 600 B` | `6*2100*4 = 50 400 B` |
| Horizontal bytes / frame | `~11.4 MB` | `~0.82 MB` |
| Saved / frame | - | `~10.6 MB` |

The cycle targets lower behavior RTT and lower Step 2 wall by reducing response
transfer and Python `result.as_numpy(...)` / decode work for the largest child.
It may not reduce full TensorRT compute if TensorRT cannot eliminate unused
internal head channels from the sliced graph; the production benchmark is the
acceptance authority.

## Implementation Scope

Only B.2.b is in scope:

- Add an env-gated `GAZE_HORIZONTAL_HEAD_VARIANT=coco80|gaze2`.
- Add `gaze_horizontal_gaze2_model` Triton config and build support.
- Add `behavior_ensemble_gaze2` Triton config and route override support.
- Update the crop-frame decoder to expect 6 channels only for
  `gaze_horizontal` when `GAZE_HORIZONTAL_HEAD_VARIANT=gaze2`, then remap
  compact class IDs to legacy `4/5`.
- Keep the legacy dense path unchanged.

Out of scope:

- B.1 compact postprocessing / BLS.
- B.2.a top-K anchor packing.
- INT8 precision.
- New behavior, gaze, attention, cheating, or semantic models.
- Any LPM scope change.

## Risk Assessment

| Risk | Level | Mitigation |
|---|---|---|
| Class-ID drift (`0/1` persisted instead of `4/5`) | High | Unit test compact-to-legacy remap and production DB parity |
| Ensemble config mismatch | Medium | Startup validator variant support and Triton readiness check |
| TensorRT cannot build sliced ONNX | Medium | Roll back env; legacy model remains loaded |
| No Step 2 improvement | Medium | Mark NOT ACCEPTED if production benchmark misses gate |
| Accuracy regression from output slicing | Low-to-medium | Parity probe compares sliced output to legacy channels 4/5 before full benchmark |

## Rollback Strategy

Production rollback is:

```bash
GAZE_HORIZONTAL_HEAD_VARIANT=coco80
MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble
TRITON_LOAD_MODEL=person_detector,posture_model,gaze_horizontal_model,gaze_vertical_model,gaze_depth_model,rtmpose_model,behavior_ensemble
```

Then restart Triton and Celery workers. The legacy `gaze_horizontal_model` and
`behavior_ensemble` remain deployed.

## Acceptance Criteria

The candidate is not accepted unless all are true on the production RTX 5090
server using `combined.mp4`:

1. Sliced-output parity probe passes: compact output equals legacy bbox channels
   plus legacy class channels `4/5` within `1e-6`.
2. Job reaches DB `status=completed`.
3. Step 2 wall improves by at least `5 %` vs Cycle 9 (`858.1 s`) for this
   child-output cycle, or the result is documented as NOT ACCEPTED.
4. Overall DB-completed FPS improves.
5. Detection, bbox, embedding, StudentTrack, and per-class bbox counters remain
   within the existing parity gate.
6. Benchmark evidence is recorded in:
   - `docs/production_inference_benchmark.md`
   - `docs/inference_parallelization_plan.md`
   - `docs/cycle_9b_output_fusion_results.md`
   - `AGENTS.md`

Until then this cycle remains STAGED / NOT ACCEPTED.
