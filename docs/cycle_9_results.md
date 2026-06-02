# Cycle 9 Results: Behavior Triton Ensemble

Date: 2026-06-01

## Decision

**NEEDS FURTHER ITERATION.**

Cycle 9 completed code implementation, production deployment, production
benchmark, tensor parity validation, row-parity validation, and documentation
updates. It is **not accepted** because the required Step 2 wall-time reduction
was not achieved.

## Production Evidence

| Item | Value |
|---|---|
| Replay key | `cycle9-behavior-ensemble-crop-frame-20260601T180847` |
| Job id | `c1651663-e08a-4e29-9ee3-fd0f09884b98` |
| Candidate SHA | `0fa847af43186017316cc11a8c76645ff463e574` |
| Video | `/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4` |
| Benchmark log | `backend/logs/parallel_flow_cycle9-behavior-ensemble-crop-frame-20260601T180847.log` |
| Benchmark summary | `backend/logs/bench_summary_20260601T180857.json` |
| GPU CSV | `backend/logs/gpu_monitor_bench_20260601T180857.csv` |
| Tensor parity | `backend/logs/behavior_ensemble_parity_cycle9_20260601T180827.json` |
| Final status | `completed` |

## Changes

- Added `backend/models/triton_repository_cuda12/behavior_ensemble/config.pbtxt`.
- Added `TRITON_BEHAVIOR_ENSEMBLE` default-off app flag.
- Added model route `behavior_all -> behavior_ensemble:v1`.
- Updated Triton client IO routing to request four ensemble outputs.
- Updated crop-frame behavior dispatch to call `behavior_all` once and split
  outputs back to legacy task keys.
- Added fallback to standalone behavior models when ensemble responses are
  invalid.
- Added production repository validator and startup check.
- Added production tensor parity helper.
- Updated optimized production helper to preserve Cycle 8 defaults and enable
  the ensemble candidate.
- Rebuilt the pinned production Triton binary with `TRITON_ENABLE_ENSEMBLE=ON`
  after production proved the existing build had `TRITON_ENABLE_ENSEMBLE=OFF`.

## Validation

Local validation:

- `backend\.venv\Scripts\python.exe -m pytest backend/tests/unit/pipeline/test_behavior_ensemble_dispatch.py -q --tb=short`
- Inference-parallelization gate subset: `119 passed, 4 warnings`
- Route regression subset: `2 passed`
- `py_compile` for changed runtime modules and parity script
- `bash -n` for production helpers
- `python scripts/ci/verify_docs_diagrams.py`
- `git diff --check`

Production validation:

- `prod_start_triton.sh` validator: `[behavior-ensemble-validator] OK`
- Triton readiness: `TRITON_READY`
- Tensor parity: max abs diff `0.0`
- Benchmark return code: `0`
- DB status: `completed`

## Metrics

| Metric | Cycle 8 Baseline | Cycle 9 Candidate | Delta |
|---|---:|---:|---:|
| Step 2 wall | `852.8 s` | `858.1 s` | `+0.6 %` |
| Step 2 FPS | `5.33` | `5.29` | `-0.8 %` |
| Telemetry session wall | `1138.8 s` | `923.1 s` | `-18.9 %` |
| DB-completed elapsed | `1312.3 s` | `1110.7 s` | `-15.4 %` |
| DB-completed FPS | `3.46` | `4.09` | `+18.1 %` |
| App-level model calls | `20 348` | `9 557` | `-53.0 %` |
| Behavior crop app calls | `14 391` | `3 597` | `-75.0 %` |
| Behavior mean RTT | `143-168 ms/model` | `107.9 ms ensemble` | improved |
| Behavior p95 RTT | `248-277 ms/model` | `173.9 ms ensemble` | improved |
| Avg GPU utilization | `9.65 %` | `9.36 %` | `-0.29 pp` |
| Peak GPU utilization | `36 %` | `43 %` | `+7 pp` |
| Avg VRAM | `15 663 MiB` | `15 663 MiB` | unchanged |

## Correctness

| Counter | Cycle 8 | Cycle 9 | Delta |
|---|---:|---:|---:|
| Frames | 4 541 | 4 541 | 0 |
| Detections | 72 749 | 72 749 | 0 |
| Bounding boxes | 72 749 | 72 749 | 0 |
| Frame embeddings | 72 583 | 72 583 | 0 |
| Student tracks | 53 | 53 | 0 |
| `attention_tracking` boxes | 11 776 | 11 776 | 0 |
| `hand_raising` boxes | 8 800 | 8 800 | 0 |
| `person_detection` boxes | 19 162 | 19 162 | 0 |
| `sitting_standing` boxes | 33 011 | 33 011 | 0 |

## Why It Was Not Accepted

The ensemble removed app-side request fragmentation, but the ensemble still
executes the same four TensorRT behavior/gaze child models and returns the same
dense YOLO output tensors. The measured result is lower app RTT and fewer app
model-call rows, not lower Step 2 wall time.

The Cycle 9 acceptance gate required at least 10 % Step 2 wall reduction. The
measured Step 2 wall changed from `852.8 s` to `858.1 s`, so this cycle remains
open.

## Rollback

Rollback is:

```bash
TRITON_BEHAVIOR_ENSEMBLE=0
```

The standalone behavior models remain deployed and the fallback code path is
covered by unit tests. The production Triton rebuild with
`TRITON_ENABLE_ENSEMBLE=ON` is safe to keep because it preserves existing
TensorRT behavior and only adds ensemble capability. The pre-Cycle-9 binary is
saved at:

```text
/home/bamby/services/triton_build_r2502/tritonserver/install/bin/tritonserver.pre_cycle9_no_ensemble_20260601T180729
```

## Next Recommendation

Do not spend another cycle on a plain ensemble that returns dense model outputs.
The next evidence-driven option should reduce either:

- server-side four-child behavior critical path, or
- dense YOLO output movement, or
- frame-level parallelism outside the single Python process.

This points toward the planned Cycle 13 architecture decision: compare compact
server-side postprocessing/BLS against video sharding or multi-process
architecture using production measurements.

---

## Cycle 9 Post-Mortem — Why The Main Gate Failed

The improvement *was* real on three axes (app-side call count, behavior RTT,
overall DB-completed FPS). It failed on the gate we chose (Step 2 wall) because
the gate was upstream of where the savings landed.

**Old per-frame critical path (Cycle 8):**

```
max(posture_call, gaze_h_call, gaze_v_call, gaze_d_call)
```

These four calls already ran concurrently from Python (`TRITON_CONCURRENT_MODELS=1`
since Cycle 1–5), so the wall time was already `max(…)`, not `sum(…)`. The
ensemble removed the per-call gRPC fan-out tax (4 trips → 1 trip), which is
why call count dropped 75 % and RTT dropped from 143–168 ms/model to 107.9 ms
ensemble. But the **GPU still executed the same four TensorRT engines on the
same crop batches**, and the response still carried the same four dense YOLO
output tensors (`[14, 2100]` × 3 and `[84, 2100]` × 1). So:

- Step 2 wall is bounded by **server-side GPU compute + dense response transfer**,
  not by per-call request fragmentation. The fragmentation that Cycle 9 removed
  was already hidden behind concurrent dispatch.
- The Cycle 9 win shows up in `Telemetry session wall` (-18.9 %) and the
  `DB-completed elapsed` (-15.4 %) because the embedding / persistence loops
  *do* benefit from fewer telemetry rows and lower Python orchestration cost.

**Lesson formalized:** request count was no longer the dominant Step 2 limiter
when this cycle started. The remaining levers are inside the GPU work
(child-model critical path, dense output volume) or outside the single-process
ordering (multi-process or video sharding).

---

## Cycle 9b — Continuation Options (Status Updated 2026-06-02)

The next ensemble-adjacent change MUST attack one of these five levers. Each
candidate needs its own production benchmark before acceptance. Since this
post-mortem was written, Option 2 has partially landed: exact server-side
horizontal slicing is accepted, and exact slice + Top-K is accepted with caveat.
Option 3 Step 1 measurement also landed and identified `gaze_horizontal_model`
as the original dominant child.

### Option 1 — Server-side compact postprocessing (HIGHEST IMPACT)

Add a BLS Python backend or a C++ custom backend that consumes the four dense
behavior outputs **inside Triton** and emits only the post-NMS / argmax decisions
(class ID, confidence, optional bbox). Per-frame response shrinks from
`(14 + 84 + 14 + 14) × 2100 × 4 bytes ≈ 1.05 MB / crop` to roughly 32 bytes /
crop / model. With ~17 crops/frame × 4 models that is `~71 MB → ~2 KB` of dense
output movement removed per frame.

- **Mechanism placement**: new `behavior_compact_ensemble` ensemble whose final
  step is a BLS Python backend (`triton_repository_cuda12/behavior_postproc/`).
- **Expected gain**: Step 2 wall reduction not just from PCIe / gRPC bytes but
  from removing the dense `result.as_numpy(output0).reshape(...)` cost in the
  Python `_decode_yolo_output0` path that runs 17 × 4 = 68 times per frame.
- **Risk**: high — first BLS Python backend in this repo. Production validation
  during Cycle 9b showed the active Triton build did **not** have the Python
  backend available, so BLS requires an intentional backend rebuild or a
  different compact-postprocessing backend. Accuracy contract is bit-identical
  *only if* the NMS settings match the Python-side `_decode_yolo_output0`.

### Option 2 — Fuse outputs before returning

If full BLS is too much, add a stage that concatenates / packs only the needed
logits (class scores per anchor) instead of returning the full dense grid.
Even partial compression (return top-K anchors per model, not all 2 100) cuts
output bytes 20-100× without changing the Python decoder contract.

**2026-06-02 status:** exact server-side horizontal slicing is **ACCEPTED** and
exact slice + FP32 Top-K is **ACCEPTED WITH CAVEAT**. The accepted Top-K run
(`cycle9b-topk-crop-frame-20260602T041900`, job
`be4ba9ee-4786-48e9-8334-28feb237a1fb`) reduced Step 2 frame wall
`573.927 → 540.399 s`, behavior RTT mean `91.470 → 84.865 ms`, and behavior
output traffic `~6.85 → ~0.33 MB/frame`, but average GPU utilization did not
improve. Do not continue optimizing response-byte volume alone.

- **Mechanism placement**: a thin C++ custom backend OR an ensemble step that
  invokes a tiny TensorRT engine wrapping a top-K op.
- **Expected gain**: less than Option 1 but with a much smaller engineering
  footprint.
- **Risk**: medium — the top-K choice must match Python-side
  `TRITON_YOLO_MAX_DECODE_CANDIDATES=100` exactly.

### Option 3 — Reduce the child model critical path

The ensemble still waits for the slowest sub-model. Cycle 9 telemetry shows the
ensemble mean RTT 107.9 ms; the standalone per-model RTTs were 143–168 ms
spread asymmetrically. Need server-side measurement of which sub-model dominates,
then optimize that one specifically:

**2026-06-02 status:** Step 1 measurement is complete. Production stats in
`docs/cycle_9b_child_critical_path_results.md` identified
`gaze_horizontal_model` as the original dominant child (`16.058 ms/exec` server
delta). After exact-slice + Top-K, this must be remeasured before any Step 2
engine variant is implemented.

- TensorRT profile / precision tuning (FP16 → INT8 on the dominant child).
- Engine batch profile mismatch (each engine has its own preferred batch).
- Output channel pruning if any single sub-model output is wider than needed.

- **Mechanism placement**: ONNX export + TRT rebuild for the dominant child
  only. Other three children untouched.
- **Expected gain**: lifts the floor of `max(…)` for behavior dispatch.
- **Risk**: medium — precision change requires accuracy parity gate.

### Option 4 — Batch larger at ensemble level

Cycle 9 frequently formed batches of 32 already (the engine cap). Behavior
crops formed by `_build_crop_payload` could be coalesced across **2 frames at
once** so the ensemble sees ~34 crops / call instead of 17. Helps Triton hit
the preferred batch shape more often.

- **Mechanism placement**: raise `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES` 2 → 4
  with the cycle 6 pose chunking still chunking pose at 16 to avoid the
  regression of cycles 1-5.
- **Expected gain**: marginal — Cycle 9 batch histogram already centered at 32.
  This is secondary unless the dominant-child analysis (Option 3) shows the
  engine is under-utilized.
- **Risk**: medium — the prior 100 GiB RSS spike was at `MAX_FRAMES = ∞`; the
  cycle 8 trim still bounds it but we must measure.

### Option 5 — Stop optimizing gRPC call count alone

Cycle 9 proved that **call-count reduction without GPU-work reduction is
insufficient**. Any new candidate must reduce one of:

- GPU-side child compute (Option 3, Cycle 11 smaller input),
- dense output bytes returned to Python (Options 1, 2),
- frame-level Python orchestration outside the single-process ordering (the
  pose-parallelization Cycle 10 already in the plan, or video sharding /
  multi-process for a future cycle).

This is a discipline rule, not an implementation: future cycle hypotheses
must name which of these three levers they pull, before any code is written.
