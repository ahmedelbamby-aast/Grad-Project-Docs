# Cycle 9b B.2.a Top-K Anchor Packing Investigation

Date: 2026-06-02  
Branch: `feature/phase7a-crop-frame-mode`  
Status: **PHASE A ONLY - NO OPTIMIZATION ACCEPTED**

## Problem Statement

Cycle 9b exact horizontal-gaze slicing is accepted and deployed, but the
crop-frame behavior path still returns dense YOLO grids from Triton to Python.
For every person-crop batch, `behavior_ensemble_gaze_slice` still emits:

| Output | Shape | Float32 bytes / crop |
|---|---:|---:|
| `posture_out` | `[14, 2100]` | `117,600` |
| `gaze_h_out` | `[6, 2100]` | `50,400` |
| `gaze_v_out` | `[14, 2100]` | `117,600` |
| `gaze_d_out` | `[14, 2100]` | `117,600` |

At the accepted benchmark's approximate `17` crops/frame, this is about
`6.5 MB/frame` of behavior output traffic before Python decoding. The app then
keeps only the selected candidates and final NMS boxes.

The remaining opportunity is B.2.a: pack only the top `K` anchors inside
Triton, where `K = TRITON_YOLO_MAX_DECODE_CANDIDATES = 100`, so the Python
decoder receives `[C, 100]` instead of `[C, 2100]`.

## Current Accepted Baseline

Latest accepted production baseline:

| Field | Value |
|---|---:|
| Replay key | `cycle9b-exactslice-crop-frame-20260601T233211` |
| Job id | `7933c1e5-a970-47a3-81c5-0c9bd01bd332` |
| Deployed SHA after docs/config normalization | `85bf200cdc31db8d2c21577e90fb6572b47d118b` |
| Step 2 wall | `573.927 s` |
| DB-completed FPS | `4.307` |
| Behavior RTT mean | `91.470 ms` |
| Behavior RTT p95 | `146.015 ms` |
| Avg / peak GPU utilization | `9.595 % / 45 %` |
| Parity | `gaze_horizontal_slice` raw tensor parity `max_abs_diff=0.0`; DB counters preserved |

Production runtime verification before this investigation:

```text
Triton backends:
distributed_addsub, dyna_sequence, implicit_state, iterative_sequence,
query, sequence, tensorrt
```

There is no `python` backend in the current production Triton build. Therefore
B.1.a Python BLS compact postprocessing is not an immediately safe
implementation path unless production Triton is rebuilt/reinstalled with the
Python backend. That is a separate infrastructure-risk task, not a small model
repository change.

## Root Cause Targeted

The targeted remaining waste is dense output movement and dense-grid Python
decode, not model execution.

Relevant code path:

1. `backend/apps/video_analysis/tasks.py:_run_crop_behaviour_for_items()`
   dispatches all person crops through `behavior_all` when
   `TRITON_BEHAVIOR_ENSEMBLE=1`.
2. `backend/models/triton_repository_cuda12/behavior_ensemble_gaze_slice/config.pbtxt`
   fans one crop batch into posture, horizontal-gaze, vertical-gaze, and depth.
3. `_split_behavior_ensemble_responses()` maps the four outputs back to the
   existing single-output decoder path.
4. `_decode_yolo_output0()` performs class argmax, confidence filter,
   class-filter masking, candidate cap, coordinate transform, and NMS.

The exact decoder ordering in current code is:

1. Compute `cls_id = argmax(cls_scores)` and `conf = max(cls_scores)` across
   the model output class channels.
2. Apply `conf >= behavior_confidence_threshold`.
3. Apply task-specific `class_filter`.
4. If survivors exceed `TRITON_YOLO_MAX_DECODE_CANDIDATES`, keep only the top
   `K` survivors by confidence.
5. Convert coordinates.
6. Run NMS.

This ordering matters: any Triton-side Top-K packing must compute the same
global class argmax/confidence as Python, mask anchors whose global class id is
outside the task filter, and then rank the survivors by global confidence.
Ranking only by the two accepted class-score channels would be incorrect for
the 14-channel behavior heads because Python excludes an anchor whenever any
unused class is the global argmax.

## Proposed Hypothesis

If Triton packs the top `K=100` task-relevant anchors before returning the
behavior ensemble response, Python receives about `100/2100 = 4.76 %` of the
current anchor axis while preserving the channels needed by `_decode_yolo_output0`.

Expected traffic reduction:

| Output | Accepted exact-slice bytes/frame | Top-K bytes/frame estimate |
|---|---:|---:|
| `posture_out` | `~1.999 MB` | `~0.095 MB` |
| `gaze_h_out` | `~0.857 MB` | `~0.041 MB` |
| `gaze_v_out` | `~1.999 MB` | `~0.095 MB` |
| `gaze_d_out` | `~1.999 MB` | `~0.095 MB` |
| **Total behavior output** | **`~6.854 MB/frame`** | **`~0.326 MB/frame`** |

This should reduce Triton response payload size by roughly `95 %` for the
behavior outputs. The hypothesis is that Step 2 wall and behavior RTT improve
if response transfer and Python dense-grid preparation remain material after
the accepted exact-slice win.

## Implementation Shape To Test

The lowest-risk TensorRT-only path is a per-output Top-K packing adapter:

| Adapter | Input shape | Class score channels used for Top-K | Output shape |
|---|---:|---:|---:|
| `posture_topk_model` | `[14, 2100]` | global argmax in class ids `[7, 8]` | `[14, 100]` |
| `gaze_horizontal_slice_topk_model` | `[6, 2100]` | global argmax in compact class ids `[0, 1]` | `[6, 100]` |
| `gaze_vertical_topk_model` | `[14, 2100]` | global argmax in class ids `[2, 6]` | `[14, 100]` |
| `gaze_depth_topk_model` | `[14, 2100]` | global argmax in class ids `[1, 3]` | `[14, 100]` |

The channel indexes above are output-channel indexes, not class ids. YOLO output
channels are `[xywh(0..3), class_scores...]`, so class id `n` lives at output
channel `4+n`.

A new ensemble variant can then compose with the accepted slice:

```text
images
  -> posture_model                  -> posture_dense  -> posture_topk_model -> posture_out [14,100]
  -> gaze_horizontal_model          -> gaze_h_dense84
  -> gaze_horizontal_slice_model    -> gaze_h_dense6   -> gaze_horizontal_slice_topk_model -> gaze_h_out [6,100]
  -> gaze_vertical_model            -> gaze_v_dense    -> gaze_v_topk_model  -> gaze_v_out [14,100]
  -> gaze_depth_model               -> gaze_d_dense    -> gaze_d_topk_model  -> gaze_d_out [14,100]
```

The existing app decoder can continue to parse dynamic anchor counts because it
already transposes by observed tensor shape when `output_channels` matches.

## Required Validation Before Benchmark

Top-K cannot use raw tensor equality because output shape changes from
`[C,2100]` to `[C,100]`. It must pass decoded-detection parity:

1. Generate deterministic crop batches on production.
2. Run the accepted exact-slice ensemble and decode with the current Python path.
3. Run the Top-K ensemble and decode with the same Python path.
4. Compare decoded boxes per crop/model after remapping:
   - class id sequence
   - confidence values
   - `xyxy` values
   - NMS result count

Acceptance for the parity probe:

| Check | Gate |
|---|---:|
| Per-crop/model decoded count | exact match |
| Class ids after remap | exact match |
| Confidence max absolute difference | `<= 1e-6` |
| Box coordinate max absolute difference | `<= 1e-4` |

If decoded parity fails, do not run a full production benchmark.

## Production Benchmark Acceptance Criteria

Only after the parity probe passes:

| Metric | Gate |
|---|---:|
| Benchmark video | `/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4` |
| Step 2 wall | at least `5 %` better than accepted exact-slice baseline `573.927 s` |
| DB-completed FPS | measurable improvement over `4.307` |
| Behavior RTT mean | measurable improvement over `91.470 ms` |
| Correctness counters | no material regression; target `<= 0.05 %` per class |
| Completion | job reaches `4541/4541` and `completed` |
| Stale reconciler | no false `reconciled_stale_processing_state` |

The improvement gate is lower than Cycle 9's original `10 %` because exact
slicing already removed the largest horizontal-gaze output. If the measured
benefit is below `5 %`, this cycle should be rejected or treated as
needs-iteration, even if code works.

## Risk Assessment

| Risk | Level | Mitigation |
|---|---|---|
| Top-K ranks wrong channels and drops valid detections | High | Build per-task adapters with explicit class-score channel indexes; decoded parity probe is mandatory |
| ONNX `TopK` + `GatherElements` does not compile cleanly to TensorRT on the pinned CUDA 12.8 stack | Medium | Build on production with the active TensorRT stack before env routing; keep legacy route active until READY |
| Top-K before the existing confidence threshold changes tie behavior | Medium | The parity probe must include deterministic random batches and real video crop batches |
| Extra ensemble steps add enough scheduling overhead to offset transfer savings | Medium | Full production benchmark decides acceptance; rollback is an env flip |
| Validator and docs drift from shape changes | Medium | Extend `ensemble_validator.py`, docs, and focused unit tests in the implementation phase |

## Rollback Strategy

No existing accepted route should be modified destructively. The implementation
phase should add a new env-gated route:

```env
TRITON_BEHAVIOR_TOP_K_ENABLED=0
TRITON_BEHAVIOR_TOP_K_VALUE=100
MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble_gaze_slice
```

Candidate activation:

```env
TRITON_BEHAVIOR_TOP_K_ENABLED=1
TRITON_BEHAVIOR_TOP_K_VALUE=100
MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble_gaze_slice_topk
```

Rollback is:

```env
TRITON_BEHAVIOR_TOP_K_ENABLED=0
MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble_gaze_slice
```

Then restart Triton and Celery workers.

## Decision

Proceed to Phase B implementation of B.2.a only if the TensorRT Top-K adapter
can be generated with the production TensorRT stack and a decoded parity probe
can compare accepted exact-slice outputs against Top-K outputs before the full
benchmark.

This cycle remains **INCOMPLETE** until implementation, local validation,
production deployment, production parity probe, full production benchmark,
documentation updates, commit, push, deployed SHA verification, and benchmark
evidence are all complete.
