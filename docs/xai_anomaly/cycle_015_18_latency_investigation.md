# Cycle 015.18 Investigation: In-Loop Latency — Scene Lane Is The Lever

**Created:** 2026-06-11
**Status:** `implemented_local; awaiting phase-by-phase production benchmark`
**Goal:** raise DB-completed FPS (baseline 1.656) toward the ≥15 target by
attacking the measured in-loop bottleneck, one causal variable per run.

## Measured breakdown (r2 baseline, production path, scene+SRVL on)

Wall / stage (`backend/logs/cycle015_17/.../baseline_metrics.json`):

| Stage | Time | Note |
|---|---|---|
| step2 frame wall | 1626 s | the frame loop |
| step2 pose tail (over frame) | 890 s | pose inference serialized after the loop |
| step3 persistence | 33 s | **not a lever** (confirms 015.17 finding) |
| step4 render | 28 s | negligible |
| db_completed_fps | **1.656** | target ≥15 |

Per-frame postprocess phase totals (from `benchmark_live_rollup`):

| Phase | Aggregate | Share of postprocess |
|---|---|---|
| `scene_callback_ms` (run_scene_frame_lane) | 844 s | 73.6% |
| `scene_output_decode_ms` (mask decode) | 200 s | 17.4% |
| behavior_crop_payload_build | 43 s | 3.8% |
| behavior_response_decode | 24 s | 2.1% |
| tracking / interp / cadence | ~7 s | <1% |

**Scene lane = 93.4% of in-loop postprocess.** GPU sampled 0–29% utilization
throughout — compute is not the ceiling; CPU postprocess (scene) is.

## What already exists (NOT re-implemented)

Reconnaissance of `tasks.py` confirmed the pipeline is already optimized:
dynamic batching (`TRITON_DYNAMIC_BATCHING_ENABLED`, `run_inference_batch`),
concurrent model dispatch (`_dispatch_task_inputs`), a preprocess producer
thread (GPU/CPU overlap), per-model detection cadence
(`OFFLINE_DETECT_EVERY_N_FRAMES`), binary tensor uploads
(`TRITON_BINARY_TENSORS`), and async persistence (Cycle 015.17, now
parity-correct). None of these are the remaining bottleneck.

## Flag matrix (each independently togglable → benchmark one per run)

| Flag | Default | Lever | Status |
|---|---|---|---|
| `YOLOE_SCENE_EVERY_N_FRAMES` | 1 | run scene lane every Nth frame, append-only carry-forward | **implemented** |
| (collector) `step2_postprocess_phase_timings_s` | — | surface scene/decode/behavior phase split in metrics | **implemented** |
| pose-tail overlap | — | overlap the 890 s pose tail with persistence/embedding | **next cycle 015.19** |
| behavior crop batch | — | the 112 ms × headcount `behavior_ensemble` inference | **next cycle 015.20** |
| COMBINED grouped head | — | one forward pass replaces the behavior ensemble's many calls | **training track** |

## Primary lever: scene cadence (`YOLOE_SCENE_EVERY_N_FRAMES`)

The scene lane (`_on_scene_frame` → `run_scene_frame_lane`: YOLOE seg + mask
decode + per-person OSNet ReID embeddings) runs on every frame, but room
layout / zones change slowly. Three aligned gates (`_is_scene_frame`) skip
scene **payload prep**, **output decode**, and the **callback** on non-cadence
frames; the append-only scene evidence carries forward (consumers read the
latest scene row ≤ t). `N == 1` is a byte-for-byte no-op (default), so the
change is safe to land dark and benchmark by flipping N.

Expected at N=5: scene cost ~1044 s → ~210 s, removing ~830 s from a 1626 s
frame wall — the largest single lever available.

ReID safety: the r2 embedding stage already reused 126 382 / 126 519 track
embeddings, so sparse scene frames do not meaningfully change identity
coverage; correctness is verified by the cadence sweep benchmark.

## Benchmark plan (phase by phase, after 015.17 parity verdict)

1. Deploy on the reviewed SHA (clean of the 015.17 parity variable).
2. `YOLOE_SCENE_EVERY_N_FRAMES=1` → confirms no regression vs r2.
3. Sweep N = 2, 3, 5 → record db_completed_fps and the now-surfaced
   `step2_postprocess_phase_timings_s` scene share at each step.
4. Accept the largest N that holds scene-evidence correctness (track parity,
   contradiction/recovery coverage) within tolerance.
5. Then 015.19 (pose-tail overlap) and 015.20 (behavior batch), each its own
   one-variable cycle.
