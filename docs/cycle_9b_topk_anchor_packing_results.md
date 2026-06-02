# Cycle 9b Top-K Anchor Packing Results

**Last updated:** 2026-06-02

**Status:** **ACCEPTED WITH CAVEAT** for the combined exact-slice + Top-K route
(`B.2.c`). The dense-output-byte lever moved: behavior output traffic dropped by
about `95 %`, behavior RTT improved, Step 2 frame-inference wall improved, and
decoded parity passed exactly. Average GPU utilization did not improve, so this
is not the final GPU-saturation fix.

## Candidate

Cycle 9b B.2.b exact server-side slice is the accepted baseline. This candidate
keeps that route and adds Top-K packing after every behavior child:

| Item | Value |
|---|---|
| Baseline route | `GAZE_HORIZONTAL_HEAD_VARIANT=slice`, `behavior_ensemble_gaze_slice` |
| Candidate route | `TRITON_BEHAVIOR_TOP_K_ENABLED=1`, `behavior_ensemble_gaze_slice_topk` |
| Top-K value | `100` |
| Adapter models | `posture_topk_model`, `gaze_horizontal_slice_topk_model`, `gaze_vertical_topk_model`, `gaze_depth_topk_model` |
| Build precision | FP32 (`--no-fp16`) |
| Named lever | Dense output bytes |

FP16 Top-K adapters were rejected before the full benchmark: decoded parity
reported score and box drift (`max_score_diff≈2.4e-4`,
`max_box_diff≈0.138`). FP32 adapters passed decoded parity exactly.

## Production Evidence

| Item | Value |
|---|---|
| Replay key | `cycle9b-topk-crop-frame-20260602T041900` |
| Job ID | `be4ba9ee-4786-48e9-8334-28feb237a1fb` |
| Candidate / deployed SHA | `9f879affeb4478e63a09276b10a2d64844bcbc44` |
| Telemetry session | `c4710435-4ec0-49e1-8ffb-60012fa878c9` |
| Bench summary | `backend/logs/bench_summary_20260602T042139.json` |
| GPU CSV | `backend/logs/gpu_monitor_bench_20260602T042139.csv` |
| Bench run log | `backend/logs/bench_run_20260602T042139.log` |
| Inference audit | `backend/data/videos/be4ba9ee-4786-48e9-8334-28feb237a1fb/inference_audit.json` |
| Decoded parity | `backend/logs/behavior_topk_parity_20260602T011830Z_fp32.json` |
| Final status | `completed`, `4541/4541` frames |

Decoded parity result: `failed_count=0`, `comparison_count=192`,
`max_score_diff=0.0`, `max_box_diff=0.0`.

## Metrics

| Metric | Exact Slice Baseline | Exact Slice + Top-K | Delta |
|---|---:|---:|---:|
| Step 2 frame wall | `573.927 s` | `540.399 s` | `-33.528 s` / `-5.84 %` |
| Step 2 through pose upload | `799.345 s` | `767.589 s` | `-31.756 s` / `-3.97 %` |
| Audit `run.complete` wall | `865.419 s` | `833.810 s` | `-31.609 s` / `-3.65 %` |
| DB-completed elapsed | `1052.281 s` | `1022.952 s` | `-29.329 s` / `-2.79 %` |
| DB-completed FPS | `4.307` | `4.429` | `+2.8 %` |
| Behavior RTT mean | `91.470 ms` | `84.865 ms` | `-7.22 %` |
| Behavior RTT p95 | `146.072 ms` | `128.138 ms` | `-12.28 %` |
| App behavior calls | `3597` | `3597` | unchanged |
| Behavior output / frame | `~6.85 MB` | `~0.33 MB` | `~95 %` less |
| Avg GPU utilization | `9.595 %` | `9.3 %` | `-0.295 pp` |
| Peak GPU utilization | `45 %` | `53 %` | `+8 pp` |
| Approx. VRAM during run | `~16.02 GiB` | `~16.05 GiB` | flat |

## Correctness

| Counter | Exact Slice | Exact Slice + Top-K | Delta |
|---|---:|---:|---:|
| Frames | `4541` | `4541` | 0 |
| Detections | `72747` | `72762` | `+15` / `+0.0206 %` |
| Bounding boxes | `72747` | `72762` | `+15` / `+0.0206 %` |
| Frame embeddings | `72581` | `72596` | `+15` / `+0.0207 %` |
| Student tracks | `53` | `53` | 0 |
| `attention_tracking` boxes | `11776` | `11781` | `+5` / `+0.0425 %` |
| `hand_raising` boxes | `8801` | `8809` | `+8` / `+0.0909 %` |
| `person_detection` boxes | `19162` | `19162` | 0 |
| `sitting_standing` boxes | `33008` | `33010` | `+2` / `+0.0061 %` |

The DB deltas are small and non-negative on total persisted signals. The parity
probe proves the Top-K adapter preserves the current Python decoder output for
the compared dense tensors.

## Decision

**ACCEPTED WITH CAVEAT.**

Reasons accepted:

- The named lever, dense output bytes, moved by about `95 %`.
- Step 2 frame-inference wall improved by `5.84 %` versus the accepted
  exact-slice baseline.
- Behavior RTT mean and p95 improved.
- The production job completed and correctness counters stayed within the
  established parity tolerance.
- Decoded parity passed exactly on the FP32 Top-K adapters.

Caveat:

- Average GPU utilization did not improve (`9.595 % → 9.3 %`). Peak GPU
  utilization improved (`45 % → 53 %`), but the pipeline still does not saturate
  the RTX 5090. The next optimization must target GPU occupancy, server-side
  execution, or single-process orchestration rather than only trimming response
  bytes.

## Active Production Profile

```bash
GAZE_HORIZONTAL_HEAD_VARIANT=slice
TRITON_BEHAVIOR_TOP_K_ENABLED=1
TRITON_BEHAVIOR_TOP_K_VALUE=100
MODEL_ROUTE_GAZE_HORIZONTAL_MODEL_NAME=gaze_horizontal_slice_adapter
MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble_gaze_slice_topk
LPM_ENABLED=0
```

## Rollback

Return to accepted exact-slice without Top-K:

```bash
bash tools/prod/prod_enable_gaze_horizontal_slice.sh --input-size 320 --skip-build
bash -l tools/prod/prod_start_triton.sh
bash tools/prod/prod_start_celery_workers.sh
```

## Next Recommendation

Do not spend the next cycle on response-byte trimming alone. The remaining SLA
gap is too large and the Top-K benchmark shows average GPU utilization can stay
flat even after large output-byte reductions. The next Phase A should compare:

- server-side compact postprocessing / BLS availability and expected occupancy;
- dominant child execution after Top-K;
- larger ensemble batch windows with RSS guardrails;
- video sharding or persistent producer/consumer orchestration if the bottleneck
  remains Python frame sequencing.
