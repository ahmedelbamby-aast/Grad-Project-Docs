# Cycle 9b Exact Server-Side Slice Investigation

**Last updated:** 2026-06-02

**Status:** PHASE A/B COMPLETE - IMPLEMENTATION NOT STARTED.

This document is the pre-code investigation and hypothesis for a second Cycle
9b B.2.b output-fusion candidate: keep the legacy `gaze_horizontal_model`
TensorRT plan unchanged, then slice its dense output server-side before the
ensemble response crosses the Triton gRPC boundary.

## Problem Statement

The first B.2.b candidate, `gaze_horizontal_gaze2_model`, failed production
raw-tensor parity. It changed the horizontal model graph output before TensorRT
engine build, so TensorRT selected a different FP16 plan and the candidate no
longer matched the legacy output slice:

| Probe | Artifact | Max abs diff | Tolerance | Result |
|---|---|---:|---:|---|
| Initial candidate | `backend/logs/gaze_horizontal_gaze2_parity_20260601T230548.json` | `7.125` | `1e-6` | fail |
| Rebuilt current engines | `backend/logs/gaze_horizontal_gaze2_parity_20260601T231503_postrebuild.json` | `9.5` | `1e-6` | fail |

The dense-output problem still exists. `gaze_horizontal_model` returns
`[84,2100]` per crop even though the runtime only consumes legacy class IDs `4`
and `5`.

## Feasibility Finding

Production Triton backends currently installed under
`/home/bamby/services/triton_build_r2502/tritonserver/install/backends` are:

```text
distributed_addsub
dyna_sequence
implicit_state
iterative_sequence
query
sequence
tensorrt
```

There is no `python` backend installed, so B.1.a BLS Python compact
postprocessing is not the lowest-risk next implementation without a production
Triton rebuild.

The available backend is `tensorrt`. A TensorRT slice model can be built as a
small graph with:

```text
input:  legacy gaze_horizontal_model output0  [N,84,2100]
op:     Gather(axis=1, indices=[0,1,2,3,8,9])
output: compact horizontal output0            [N,6,2100]
```

Then the ensemble can run:

```text
images
  -> gaze_horizontal_model.output0 [N,84,2100]
  -> gaze_horizontal_slice_model.output0 [N,6,2100]
  -> behavior_ensemble_gaze_slice.gaze_h_out
```

This differs from the rejected `gaze_horizontal_gaze2_model`: the legacy
horizontal model plan remains the producer of the dense values, so TensorRT
cannot change the horizontal model's FP16 tactics.

## Hypothesis

**Named lever:** dense output bytes.

An exact server-side slice model should preserve raw tensor parity while still
reducing horizontal-gaze bytes returned to app Python:

| Quantity | Legacy response | Exact-slice response |
|---|---:|---:|
| Horizontal output / crop | `84*2100*4 = 705 600 B` | `6*2100*4 = 50 400 B` |
| Horizontal output / frame at ~17 crops | `~11.4 MB` | `~0.82 MB` |
| Saved over gRPC per frame | - | `~10.6 MB` |

It will not reduce the legacy horizontal model's GPU compute, because that
model still executes unchanged. The expected improvement is therefore limited
to response transfer, `result.as_numpy(...)`, and client decode cost for the
horizontal child.

## Implementation Scope

Only this exact-slice candidate is in scope:

- Add a `gaze_horizontal_slice_model` TensorRT plan/config that accepts
  `[84,2100]` and returns `[6,2100]`.
- Add `behavior_ensemble_gaze_slice` so only the horizontal branch routes
  through the slice model.
- Extend `GAZE_HORIZONTAL_HEAD_VARIANT` to support:
  - `coco80`: legacy `[84,2100]`
  - `gaze2`: rejected standalone sliced horizontal plan, kept for rollback and
    evidence only
  - `slice`: exact server-side slice after legacy horizontal output
- Reuse the existing app-side compact class remap for `slice`.
- Keep production rollback as a route/env flip back to `coco80`.

Out of scope:

- Python BLS backend or compact NMS.
- Top-K anchor packing.
- Rebuilding or retraining the horizontal model head.
- LPM behavior changes.
- New intelligence models.

## Risk Assessment

| Risk | Level | Mitigation |
|---|---|---|
| Slice model build failure | Medium | Keep legacy path deployed; rollback env to `coco80` |
| Ensemble wire mismatch | Medium | Extend startup validator and unit tests for `slice` |
| Parity still fails due TensorRT Gather behavior | Medium | Run production raw-tensor parity before any full benchmark |
| Small/no throughput gain | Medium | Mark NOT ACCEPTED if Step 2 wall does not improve |
| `gaze2` rejected route accidentally used | Low | Prod helper must set `slice`, not `gaze2`; docs retain `gaze2` as rejected |

## Rollback Strategy

```bash
GAZE_HORIZONTAL_HEAD_VARIANT=coco80
MODEL_ROUTE_GAZE_HORIZONTAL_MODEL_NAME=gaze_horizontal_model
MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble
LPM_ENABLED=0
```

Then restart Triton and Celery workers. No database migration is involved.

## Acceptance Criteria

This candidate is not accepted unless all are true on production RTX 5090:

1. `gaze_horizontal_slice_model.output0` equals
   `gaze_horizontal_model.output0[:, [0,1,2,3,8,9], :]` within `1e-6`.
2. `behavior_ensemble_gaze_slice.gaze_h_out` equals the same slice within
   `1e-6`.
3. The canonical `combined.mp4` benchmark completes with DB
   `status=completed`.
4. Step 2 wall improves by at least `5 %` vs Cycle 9 (`858.1 s`) or the result
   is documented as NOT ACCEPTED.
5. Detection, bbox, embedding, StudentTrack, and per-class bbox counters remain
   within the established parity gate.
6. Results are recorded in `docs/cycle_9b_output_fusion_results.md`,
   `docs/production_inference_benchmark.md`,
   `docs/inference_parallelization_plan.md`, and `AGENTS.md`.
