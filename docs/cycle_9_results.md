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
