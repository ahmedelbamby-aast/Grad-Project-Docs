# Cycle 15 CUDA Shared Memory vs Video Sharding Investigation

**Last updated:** 2026-06-03

**Status:** PHASE A STARTED. No implementation, production benchmark,
acceptance, rejection, skip, or closure exists for Cycle 15 yet.

## Problem Statement

The accepted production profile after Cycle 14.B2 uses ordered cross-frame
RTMPose batching with batch cap `16`. Cycle 14.C/14.C3 showed that simply
raising or internally parallelizing RTMPose batches does not improve total
throughput. Cycle 14.D Phase A then showed server-side compact behavior
postprocessing has no selected implementation candidate in the current runtime.

Cycle 15 must decide the next architecture lever with evidence:

- reduce Python/Triton memory movement with CUDA shared memory; or
- split the video into independent shards/multiple processes so more GPU work
  is kept in flight without corrupting tracking or persistence.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Current accepted profile | `docs/cycle_14b_rtmpose_scenario_results.md` | Defines accepted batch `16` pose path and its metrics. |
| Batch-size matrix | `docs/cycle_14c_pose_batch_size_matrix_results.md` | Proves batch `8` and `32` are not accepted. |
| Batch-32 repair | `docs/cycle_14c3_batch32_parallel_chunks_results.md` | Proves internal chunk parallelism worsened latency/FPS. |
| Compact-output Phase A | `docs/cycle_14d_server_side_compact_postproc_results.md` | Explains why no 14.D implementation was selected before Cycle 15. |
| Benchmark history | `docs/production_inference_benchmark.md` | Stores before/after production metrics and decision tables. |
| Execution log | `docs/crop_frame_optimization_execution.md` | Owns the current sorted roadmap. |
| Plan | `docs/inference_parallelization_plan.md` | Owns the production benchmark authority rule. |
| Phase A wrapper | `tools/prod/prod_run_cycle15_phase_a_measurements.sh` | Reproducible read-only measurement wrapper for 15.A/15.B triage. |
| Live watcher | `tools/prod/prod_watch_benchmark_metrics.sh` | Shows live production metrics during any candidate benchmark. |
| Collector | `tools/prod/prod_collect_benchmark_metrics.py` | Captures benchmark JSON/Markdown evidence bundles. |
| Offline task code | `backend/apps/video_analysis/tasks.py` | Owns current single-video orchestration and post-stage lifecycle. |
| Triton client | `backend/apps/pipeline/services/triton_client.py` | Owns the Python/gRPC request path that shared memory would affect. |
| Pose runtime | `backend/apps/pipeline/services/pose_runtime.py` | Owns RTMPose provider batching and current tail behavior. |

## Candidate Split

Cycle 15 is split before code because the better architecture is not known.

| Sub-cycle | Candidate | Required Phase A proof |
|---|---|---|
| 15.A | CUDA shared memory for Triton tensors | Current input/output copy or serialization cost is large enough to move FPS/latency after Cycle 14.B2. |
| 15.B | Video sharding / multi-process offline inference | The job can be partitioned without losing every-frame signals, track continuity, DB parity, or deterministic finalization. |
| 15.C | Hybrid shared-memory plus sharding plan | Only allowed if 15.A and 15.B each prove independent bottlenecks and the combined risk is justified. |

## Required Measurements Before Code

| Measurement | Purpose |
|---|---|
| Per-model input bytes and output bytes | Quantify what shared memory can actually remove. |
| Serialization and `as_numpy` wall per model | Bound CUDA shared memory upside. |
| Step 2 frame wall vs Step 2 through-pose wall | Decide whether the current limiter is frame-loop work or post-loop pose drain. |
| GPU average/peak utilization and VRAM | Ensure extra concurrency has GPU headroom and memory budget. |
| Tracking window dependencies | Determine shard overlap needed to preserve identity continuity. |
| DB write/idempotency boundaries | Determine whether shards can persist safely without duplicate rows or terminal-state drift. |
| Worker/process topology | Size candidate workers/processes without violating constitution §8.1.1. |

## Acceptance Criteria

No Cycle 15 scenario can be accepted until it has:

| Gate | Requirement |
|---|---|
| Production benchmark | Completed Linux RTX 5090 `combined.mp4` benchmark. |
| Correctness | DB parity, model-agreement parity, embedding parity, detection parity, and StudentTrack continuity against accepted batch `16`. |
| Performance | DB FPS, total elapsed, Step 2 frame wall, Step 2 through-pose wall, latency percentiles, and GPU utilization improve or remain within documented gates. |
| Resource safety | VRAM, worker RSS, CPU, PostgreSQL, Redis, and Triton RTT do not regress into contention. |
| Reproducibility | Wrapper, replay key, job id, deployed SHA, metrics JSON/Markdown, and rollback proof recorded. |

## Initial Hypothesis

Cycle 15 should start measurement-only. The likely first decision is not
whether shared memory or sharding is theoretically faster; it is whether the
measured remaining cost is dominated by copy/serialization or by insufficient
parallel work in flight. If copy/serialization is small, 15.A must not be
implemented first. If shard boundaries cannot preserve tracks and every-frame
signals, 15.B must remain investigation-only until a deterministic stitching
contract exists.

## Next Step

Run `tools/prod/prod_run_cycle15_phase_a_measurements.sh` on production. It
captures copy/serialization, per-model bytes, current queue topology, shared
memory registration status, and shardability evidence against the accepted
batch `16` baseline. Do not implement CUDA shared memory or sharding until
that wrapper identifies one highest-ROI scenario.
