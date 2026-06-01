# Cycle 9b Output Fusion Results

**Status:** **B.2.b TensorRT output-slice variant NOT ACCEPTED.**

This document records the production result for the first Cycle 9b B.2.b
candidate: `gaze_horizontal_gaze2_model`, a separate TensorRT plan that gathers
legacy horizontal output channels `[0,1,2,3,8,9]` and returns `[6,2100]`.

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

## B.2.b Follow-Up — Exact Server-Side Slice (STAGED)

**Status:** STAGED locally, not accepted.

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

This candidate is not accepted until production builds the slice plan, the
parity probe passes, and the canonical `combined.mp4` benchmark records
before/after metrics.
