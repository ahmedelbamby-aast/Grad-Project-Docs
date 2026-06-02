# Cycle 11 — Behavior Input Size Results

**Last updated:** 2026-06-02

**Status:** **11.A NOT ACCEPTED — FAILED PRE-BENCHMARK PARITY GATE.**

Cycle 11.A attempted to reduce behavior/gaze crop input size from `320x320` to
`256x256` for the four base behavior children:

- `posture_model`
- `gaze_horizontal_model`
- `gaze_vertical_model`
- `gaze_depth_model`

The named lever was **GPU child compute across all four behavior children**.
The candidate was not benchmarked on `combined.mp4` because the required
pre-benchmark parity probe failed. Production was rolled back to the accepted
Cycle 9b B.2.c baseline (`320`, exact slice + Top-K).

## Evidence

| Item | Value |
|---|---|
| Baseline profile | `TRITON_CROP_BEHAVIOR_INPUT_SIZE=320`, `behavior_ensemble_gaze_slice_topk` |
| Candidate profile | `TRITON_CROP_BEHAVIOR_INPUT_SIZE=256`, `behavior_ensemble_gaze_slice_topk` |
| Baseline capture | `backend/logs/parity_capture_320_20260602T123459.npz` |
| Candidate capture | `backend/logs/parity_capture_256_20260602T154826.npz` |
| Parity JSON | `backend/logs/parity_input_size_256_20260602T154842.json` |
| Candidate output shapes | `[200,14,1344]`, `[200,84,1344]`, `[200,14,1344]`, `[200,14,1344]` |
| Rollback proof | production env restored to `TRITON_CROP_BEHAVIOR_INPUT_SIZE=320`; active Top-K models READY |
| Runtime guard commit | `4bcc79a5a4ea7c4d452b6fcd3ae3a6ff064a3bb5` |

The candidate did show lower synthetic capture wall per child, which confirms
the performance hypothesis directionally:

| Model | 320 capture ms | 256 capture ms | Delta |
|---|---:|---:|---:|
| `posture_model` | `622.49` | `394.41` | `-36.6 %` |
| `gaze_horizontal_model` | `760.83` | `383.08` | `-49.6 %` |
| `gaze_vertical_model` | `586.83` | `271.43` | `-53.7 %` |
| `gaze_depth_model` | `593.77` | `307.81` | `-48.2 %` |

That speed is not usable until accuracy parity is proven.

## Parity Gate Result

| Model | Class agreement | Mean centroid drift at candidate scale | Gate result |
|---|---:|---:|---|
| `posture_model` | `0.6950` | `119.204 px` | **fail** |
| `gaze_horizontal_model` | `1.0000` | `0.276 px` | pass |
| `gaze_vertical_model` | `0.9550` | `142.669 px` | **fail** |
| `gaze_depth_model` | `1.0000` | `141.526 px` | **fail** |

Configured gates from the investigation:

| Gate | Threshold |
|---|---:|
| Top-1 class agreement | `>= 0.995` |
| Mean centroid drift | `<= 0.5 px` |
| Mean confidence delta | `<= 0.05` |

Failures recorded by the probe:

```text
posture_model: class_agreement 0.6950 < 0.9950
posture_model: centroid_drift_mean_px 119.204 > 0.500
gaze_vertical_model: class_agreement 0.9550 < 0.9950
gaze_vertical_model: centroid_drift_mean_px 142.669 > 0.500
gaze_depth_model: centroid_drift_mean_px 141.526 > 0.500
```

## Operational Recovery

The 256 switch exposed one runtime guard bug: `prod_start_triton.sh` validated
`TRITON_LOAD_MODEL` but did not pass it to Triton, so Triton scanned every model
directory and tried to load stale 320-only compatibility ensembles. The fix in
`4bcc79a5` enables explicit model loading when `TRITON_LOAD_MODEL` is set.

Rollback steps executed:

```bash
bash tools/prod/prod_set_behavior_input_size.sh --input-size 320 --topk 100
bash tools/prod/prod_stop_celery_workers.sh
bash tools/prod/prod_start_celery_workers.sh
```

Final verified production state:

```text
TRITON_CROP_BEHAVIOR_INPUT_SIZE=320
GAZE_HORIZONTAL_HEAD_VARIANT=slice
MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble_gaze_slice_topk
TRITON_BEHAVIOR_TOP_K_ENABLED=1
TRITON_BEHAVIOR_TOP_K_VALUE=100
LPM_ENABLED=0
Triton /v2/health/ready = 200
behavior_ensemble_gaze_slice_topk = READY
```

## Decision

**11.A is NOT ACCEPTED.**

No full `combined.mp4` benchmark was run because the candidate failed the
pre-benchmark correctness gate. The accepted production baseline remains Cycle
9b B.2.c exact slice + Top-K at `320x320`.

## Next Recommendation

Do not retry `256x256` on the production path without a better accuracy gate.
The current synthetic parity probe is strict enough to block the candidate, but
it is not a substitute for a labeled accuracy set or a real-crop teacher/student
comparison. The next optimization should choose one of:

- implement the lower-risk B.3.b/B.3.d kernel-tactic tuning on the dominant
  child at `320x320`;
- build a real-crop parity harness from `combined.mp4` detections before
  reconsidering smaller behavior input sizes;
- move to compact server-side execution / BLS if the next cycle targets
  orchestration instead of per-crop compute.
