# Cycle 15.B Video Sharding Design-Proof Investigation

**Last updated:** 2026-06-03

**Status:** PHASE A STARTED. No video-sharding implementation, production
benchmark, acceptance, rejection, skip, or closure exists yet.

## Problem Statement

Cycle 15 Phase A selected video sharding for design proof, not implementation.
Sharding is attractive because it can expose independent work and keep the RTX
5090 busier than the current single-video loop, but it can also corrupt the
project's strongest correctness contract: every frame must keep every signal,
tracking continuity, embeddings, DB rows, and final lifecycle state.

This sub-cycle exists to prove the sharding safety contract before any
multi-process code is written.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Cycle 15 result | `docs/cycle_15_cuda_shared_memory_vs_sharding_results.md` | Promotes sharding to design proof and does not select shared memory. |
| Accepted baseline | `docs/cycle_14b_rtmpose_scenario_results.md` | Current accepted batch `16` baseline for correctness and throughput. |
| Benchmark history | `docs/production_inference_benchmark.md` | Stores production decision metrics and replay keys. |
| Execution log | `docs/crop_frame_optimization_execution.md` | Owns the sorted roadmap and next-task status. |
| Offline task code | `backend/apps/video_analysis/tasks.py` | Current single-job frame loop, finalization, and DB persistence lifecycle. |
| Tracking code | `backend/apps/tracking/` | Owns track continuity and embedding side effects that sharding can break. |
| Pose runtime | `backend/apps/pipeline/services/pose_runtime.py` | Current accepted cross-frame RTMPose batching. |
| Metrics collector | `tools/prod/prod_collect_benchmark_metrics.py` | Required evidence schema for future candidate benchmarks. |
| Live watcher | `tools/prod/prod_watch_benchmark_metrics.sh` | Live benchmark metrics display that future sharding runs must support. |

## Candidate Split

The first implementation benchmark, if design proof passes, must stay narrow.

| Sub-scenario | Candidate | Why this order |
|---|---|---|
| 15.B1 | Two-shard offline inference with overlap/stitching | Lowest duplicate-frame overhead: `0.704691 %` in Phase A sizing. |
| 15.B2 | Four-shard offline inference with overlap/stitching | Higher potential parallelism but more boundary risk: `2.114072 %` duplicate-frame overhead. |
| 15.B3 | Six/eight-shard stress case | Only after B1/B2 preserve correctness; overhead and lifecycle risk increase. |

## Design Proof Required Before Code

| Requirement | Proof needed |
|---|---|
| Deterministic shard boundaries | Same input video and env produce identical shard intervals and overlap windows. |
| Overlap ownership | Exactly one shard owns each non-overlap frame; overlap rows are either suppressed or marked non-authoritative before persistence. |
| Track stitching | StudentTrack continuity is preserved across boundaries or boundary tracks are deterministically merged. |
| DB idempotency | `Frame`, `Detection`, `BoundingBox`, `FrameEmbedding`, and tracking rows cannot duplicate if a shard retries. |
| Terminal-state coordination | Parent job cannot complete until every shard has terminal success or forced failure. |
| Evidence stitching | Benchmark collector can report parent and shard metrics without hiding a failed shard. |
| Rollback | Env/config resets to accepted single-job batch `16` flow without model changes. |

## Acceptance Criteria

No 15.B scenario can be accepted until it has:

| Gate | Requirement |
|---|---|
| Production benchmark | Completed Linux RTX 5090 `combined.mp4` benchmark. |
| Correctness | DB parity, model-agreement parity, embedding parity, detection parity, and StudentTrack continuity against accepted batch `16`. |
| Performance | DB FPS, total elapsed, Step 2 frame wall, Step 2 through-pose wall, model RTT, and GPU utilization improve without resource contention. |
| Lifecycle | Every shard and parent job reaches terminal status; timeout or stuck processing is a failure. |
| Resource safety | VRAM, RSS, CPU, PostgreSQL, Redis, Triton RTT, and worker queues remain within recorded budgets. |
| Reproducibility | Wrapper, deployed SHA, replay key, parent job id, shard job ids, JSON/Markdown evidence, and rollback proof are recorded. |

## Initial Hypothesis

Start with 15.B1 two-shard design because Phase A estimated only `0.704691 %`
extra overlap frames. Four-shard or larger scenarios must wait until the
two-shard contract proves exact DB/model parity and deterministic stitch
behavior.

## Next Step

Create a design-proof script or dry-run helper that emits shard intervals,
overlap windows, expected authoritative frame ownership, and duplicate-row
suppression rules for `combined.mp4`. This helper must be benchmark-neutral and
must not submit shard jobs yet.
