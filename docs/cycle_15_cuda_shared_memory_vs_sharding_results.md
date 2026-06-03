# Cycle 15 CUDA Shared Memory vs Video Sharding Results

**Last updated:** 2026-06-03

**Status:** PHASE A MEASURED / 15.B DESIGN-PROOF NEXT. No Cycle 15
optimization has been implemented, benchmarked, accepted, rejected, skipped,
or closed.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Investigation | `docs/cycle_15_cuda_shared_memory_vs_sharding_investigation.md` | Defines 15.A shared memory and 15.B sharding before code. |
| Phase A wrapper | `tools/prod/prod_run_cycle15_phase_a_measurements.sh` | Reproducible read-only production measurement wrapper. |
| Accepted baseline | `docs/cycle_14b_rtmpose_scenario_results.md` | Batch `16` is the accepted comparison profile. |
| Benchmark history | `docs/production_inference_benchmark.md` | Stores production metrics and decision history. |
| Plan | `docs/inference_parallelization_plan.md` | Owns the cycle order and production benchmark authority. |
| Execution log | `docs/crop_frame_optimization_execution.md` | Tracks current optimization status and next task. |
| Offline orchestration | `backend/apps/video_analysis/tasks.py` | Current single-video lifecycle and persistence boundary. |
| Triton client | `backend/apps/pipeline/services/triton_client.py` | Current Python/gRPC request boundary that shared memory would affect. |
| Pose runtime | `backend/apps/pipeline/services/pose_runtime.py` | Current RTMPose cross-frame batching and pose-tail behavior. |

## Production Evidence

| Field | Value |
|---|---|
| Evidence directory | `/home/bamby/grad_project/backend/logs/cycle15-phase-a-20260603T180125Z-r2` |
| Deployed SHA | `a69b431` |
| Baseline replay | `cycle14b-cross-frame-batch16-r2-20260603T150000Z` |
| Baseline job | `6b42a557-b954-4954-a2f8-de54634229eb` |
| Video | `/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4` |
| Production mutation | none; read-only probes and collector only |

## Baseline Snapshot

| Metric | Value |
|---|---:|
| Frames | `4541` |
| Video FPS | `30.0` |
| Baseline DB FPS | `5.680314` |
| Step 2 frame wall | `462.188348 s` |
| Step 2 through-pose wall | `633.939294 s` |
| GPU avg util | `12.168 %` |
| GPU peak util | `51.0 %` |

## 15.A CUDA Shared Memory Evidence

| Metric | Value |
|---|---:|
| CUDA shared-memory registered regions | `0` |
| System shared-memory registered regions | `0` |
| Behavior input bytes/probe batch | `20,889,600` |
| Behavior output bytes mean | `326,400` |
| Behavior serialize mean | `2.766 ms` |
| Behavior deserialize mean | `2.296 ms` |
| Behavior `as_numpy` mean | `0.115 ms` |
| Behavior infer wait mean | `49.950 ms` |
| Ensemble RTT mean | `64.85 ms` |
| Ensemble transport + server mean | `59.79 ms` |
| `behavior_ensemble_gaze_slice_topk` server avg | `30.289 ms/exec` |

**Decision boundary:** CUDA shared memory is not selected for immediate
implementation. The directly measured client-side behavior copy/parse window is
about `5.177 ms` per 17-crop probe batch (`serialize + deserialize + as_numpy`),
while wait/server time is much larger. Shared memory may still be useful later,
but Phase A does not prove it is the highest-ROI next implementation.

## 15.B Video Sharding Evidence

| Shards | Shard lengths | Assumed overlap | Extra frames pct |
|---:|---|---:|---:|
| `2` | `[2271, 2270]` | `32` | `0.704691 %` |
| `3` | `[1514, 1514, 1513]` | `32` | `1.409381 %` |
| `4` | `[1136, 1135, 1135, 1135]` | `32` | `2.114072 %` |
| `6` | `[757, 757, 757, 757, 757, 756]` | `32` | `3.523453 %` |
| `8` | `[568, 568, 568, 568, 568, 567, 567, 567]` | `28` | `4.316230 %` |

**Decision boundary:** video sharding is the next design-proof candidate, not an
implementation yet. It is the only Cycle 15 scenario with plausible large
throughput upside, but it changes lifecycle, tracking, duplicate-overlap
handling, DB idempotency, finalization, and evidence stitching.

## Phase A Decision

| Candidate | Phase A decision | Reason |
|---|---|---|
| 15.A CUDA shared memory | NOT SELECTED FOR NEXT IMPLEMENTATION | The measured copy/parse window is small relative to wait/server and Step 2 through-pose wall. |
| 15.B video sharding | PROMOTED TO DESIGN-PROOF SUB-CYCLE | It can expose independent work, but must first prove track stitching and DB idempotency before code. |
| 15.C hybrid | NOT ELIGIBLE | Hybrid work is blocked until 15.A and 15.B each independently prove value. |

## Next Step

Create a Cycle 15.B design-proof investigation for deterministic shard
boundaries, overlap windows, track stitching, duplicate suppression, DB
idempotency, terminal-state coordination, and benchmark rollback. Do not start
multi-process implementation until that proof exists.
