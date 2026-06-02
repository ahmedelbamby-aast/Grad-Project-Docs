# Cycle 11 — Behavior Input Size Results

**Last updated:** 2026-06-02

**Status:** **11.A NOT ACCEPTED — REAL PRODUCTION BENCHMARK CORRECTNESS REGRESSION.**

Cycle 11.A attempted to reduce behavior/gaze crop input size from `320x320` to
`256x256` for the four base behavior children:

- `posture_model`
- `gaze_horizontal_model`
- `gaze_vertical_model`
- `gaze_depth_model`

The named lever was **GPU child compute across all four behavior children**.
The synthetic pre-benchmark parity probe produced a serious correctness warning,
but it was not treated as the final decision authority. The real `combined.mp4`
production benchmark was then executed. It confirmed a strong throughput win,
but it also showed a large DB/signal correctness regression and lower average
GPU utilization. Production was rolled back to the accepted Cycle 9b B.2.c
baseline (`320`, exact slice + Top-K).

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
| Full-video benchmark | `cycle11-input256-realbench-20260602T161641Z-input256` |
| Benchmark job | `822b0da4-fbf2-4186-a5a6-dd066f2eb571` |
| Benchmark summary | `backend/logs/bench_summary_20260602T192327.json` |
| Matrix directory | `backend/logs/cycle11-input256-realbench-20260602T161641Z/` |
| Candidate metrics JSON | `backend/logs/cycle11-input256-realbench-20260602T161641Z/input_256_metrics.json` |
| Candidate metrics Markdown | `backend/logs/cycle11-input256-realbench-20260602T161641Z/input_256_metrics.md` |
| Model agreement JSON | `backend/logs/cycle11-input256-realbench-20260602T161641Z/model_agreement_320_vs_256.json` |
| Model agreement Markdown | `backend/logs/cycle11-input256-realbench-20260602T161641Z/model_agreement_320_vs_256.md` |
| Reproducible matrix runner | `tools/prod/prod_run_behavior_input_size_matrix.sh` |
| Metrics collector | `tools/prod/prod_collect_benchmark_metrics.py` |
| Model agreement collector | `tools/prod/prod_compare_benchmark_accuracy.py` |

The candidate did show lower synthetic capture wall per child, which confirms
the performance hypothesis directionally:

| Model | 320 capture ms | 256 capture ms | Delta |
|---|---:|---:|---:|
| `posture_model` | `622.49` | `394.41` | `-36.6 %` |
| `gaze_horizontal_model` | `760.83` | `383.08` | `-49.6 %` |
| `gaze_vertical_model` | `586.83` | `271.43` | `-53.7 %` |
| `gaze_depth_model` | `593.77` | `307.81` | `-48.2 %` |

That speed was not sufficient evidence by itself. The real benchmark below
shows why the candidate does not ship.

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

## Production Benchmark Matrix Result

| Metric | 320 Top-K baseline | 256 candidate | Delta |
|---|---:|---:|---:|
| Replay key | `cycle9b-topk-crop-frame-20260602T041900` | `cycle11-input256-realbench-20260602T161641Z-input256` |  |
| Job ID | `be4ba9ee-4786-48e9-8334-28feb237a1fb` | `822b0da4-fbf2-4186-a5a6-dd066f2eb571` |  |
| Status | `completed` | `completed` |  |
| DB-completed FPS | `4.439` | `4.820` | `+8.58 %` |
| DB-completed elapsed | `1022.952 s` | `942.127 s` | `-7.90 %` |
| Step 2 FPS | `8.403` | `11.594` | `+37.97 %` |
| Step 2 frame wall | `540.399 s` | `391.673 s` | `-27.52 %` |
| Step 2 through pose upload | `767.589 s` | `615.070 s` | `-19.87 %` |
| Behavior RTT mean | `84.865 ms` | `51.529 ms` | `-39.28 %` |
| Behavior RTT p95 | `128.056 ms` | `77.574 ms` | `-39.42 %` |
| GPU avg util | `9.344 %` | `7.367 %` | `-21.16 %` |
| GPU peak util | `53.0 %` | `35.0 %` | `-33.96 %` |
| Peak VRAM | `16055 MiB` | `15295 MiB` | `-4.73 %` |
| Detection rows | `72,762` | `101,213` | `+39.10 %` |
| BBox rows | `72,762` | `101,213` | `+39.10 %` |
| Embedding rows | `72,596` | `101,047` | `+39.19 %` |
| Student tracks | `53` | `53` | `0` |
| `attention_tracking` agreement accuracy proxy | `100.000 %` | `31.195 %` | `-68.805 pp` |
| `hand_raising` agreement accuracy proxy | `100.000 %` | `38.032 %` | `-61.968 pp` |
| `person_detection` agreement accuracy proxy | `100.000 %` | `100.000 %` | `0 pp` |
| `sitting_standing` agreement accuracy proxy | `100.000 %` | `65.250 %` | `-34.750 pp` |

The agreement accuracy proxy is `F1@IoU0.5` against the accepted 320 Top-K
baseline, not human-labeled ground-truth accuracy. The benchmark has no manual
labels; therefore 320 is the reference by definition and scores `100 %`.

Per-class persisted signal changes:

| BBox model | 320 Top-K baseline | 256 candidate | Delta |
|---|---:|---:|---:|
| `attention_tracking` | `11,781` | `20,558` | `+74.50 %` |
| `hand_raising` | `8,809` | `15,944` | `+80.99 %` |
| `person_detection` | `19,162` | `19,162` | `0` |
| `sitting_standing` | `33,010` | `45,549` | `+37.99 %` |

The candidate improved speed by shrinking behavior input compute, but it changed
the persisted behavior signal distribution substantially. This violates the
correctness/parity requirement for per-frame signals.

Per-model baseline agreement:

| Model | 320 reference accuracy proxy | 256 agreement F1@IoU0.5 | Precision vs 320 | Recall vs 320 | Frame-count match | 320 boxes | 256 boxes | Count delta |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| `attention_tracking` | `100.000 %` | `31.195 %` | `24.535 %` | `42.815 %` | `13.213 %` | `11,781` | `20,558` | `+74.50 %` |
| `hand_raising` | `100.000 %` | `38.032 %` | `29.522 %` | `53.434 %` | `16.957 %` | `8,809` | `15,944` | `+81.00 %` |
| `person_detection` | `100.000 %` | `100.000 %` | `100.000 %` | `100.000 %` | `100.000 %` | `19,162` | `19,162` | `0.00 %` |
| `sitting_standing` | `100.000 %` | `65.250 %` | `56.269 %` | `77.643 %` | `9.866 %` | `33,010` | `45,549` | `+37.99 %` |

## Reproduction Commands

Use the matrix runner to produce comparable, reproducible evidence:

```bash
cd /home/bamby/grad_project

# Full 320-vs-256 matrix. Rolls back to 320 at exit.
bash tools/prod/prod_run_behavior_input_size_matrix.sh \
  --sizes "320 256" \
  --tag cycle11-input-size-realbench-$(date -u +%Y%m%dT%H%M%SZ) \
  --timeout 7200
```

To benchmark only the 256 candidate against the already accepted Cycle 9b Top-K
baseline, first collect the baseline metrics from the existing replay key, then
run only `256`:

```bash
cd /home/bamby/grad_project

backend/.venv/bin/python tools/prod/prod_collect_benchmark_metrics.py \
  --replay-key cycle9b-topk-crop-frame-20260602T041900 \
  --output backend/logs/cycle9b_topk_320_baseline_metrics.json \
  --markdown-output backend/logs/cycle9b_topk_320_baseline_metrics.md

bash tools/prod/prod_run_behavior_input_size_matrix.sh \
  --sizes "256" \
  --baseline-metrics backend/logs/cycle9b_topk_320_baseline_metrics.json \
  --tag cycle11-input256-realbench-$(date -u +%Y%m%dT%H%M%SZ) \
  --timeout 7200
```

The matrix produces:

- `matrix_runs.tsv`
- `input_<size>_metrics.json`
- `input_<size>_metrics.md`

To reproduce the per-model agreement table:

```bash
cd /home/bamby/grad_project

PYTHONPATH=backend DJANGO_SETTINGS_MODULE=config.settings.production \
  backend/.venv/bin/python tools/prod/prod_compare_benchmark_accuracy.py \
    --baseline-replay-key cycle9b-topk-crop-frame-20260602T041900 \
    --candidate-replay-key cycle11-input256-realbench-20260602T161641Z-input256 \
    --output backend/logs/cycle11-input256-realbench-20260602T161641Z/model_agreement_320_vs_256.json \
    --markdown-output backend/logs/cycle11-input256-realbench-20260602T161641Z/model_agreement_320_vs_256.md
```

## Decision

**11.A is NOT ACCEPTED by real production benchmark.**

The candidate is not skipped or neglected; it was benchmarked on production and
the benchmark explains the decision. It meets the Step 2 speed target, but fails
DB/signal correctness and reduces average GPU utilization. The accepted
production runtime remains Cycle 9b B.2.c exact slice + Top-K at `320x320`.

## Next Recommendation

The next smaller-input attempt must explain and fix the behavior over-detection
before rerunning 256. Follow-up work remains:

- implement the lower-risk B.3.b/B.3.d kernel-tactic tuning on the dominant
  child at `320x320`;
- build a real-crop parity harness from `combined.mp4` detections before
  reconsidering smaller behavior input sizes;
- move to compact server-side execution / BLS if the next cycle targets
  orchestration instead of per-crop compute.
