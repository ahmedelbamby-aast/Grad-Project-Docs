# Cycle 14.A Pose-Tail Decomposition Investigation

**Last updated:** 2026-06-03

**Status:** Phase A investigation started. No runtime code, production env,
worker topology, Triton config, or benchmark decision is changed by this
document. The only valid state is
`NO_DECISION_PRODUCTION_BENCHMARK_REQUIRED` until a completed production Linux
RTX 5090 `combined.mp4` benchmark records the required evidence.

## Problem Statement

Cycle 16.B is accepted and removed the measured Redis side-effect execution
shape as the next Redis bottleneck. The largest remaining measured wall in the
accepted evidence is now the Step 2 through-pose tail over the frame-stage wall:

| Metric | Cycle 16.B value |
|---|---:|
| Step 2 frame wall | `460.698 s` |
| Step 2 through pose upload | `682.475 s` |
| Through-pose tail over frame wall | `221.777 s` |
| Embedding profile wall | `97.505 s` |
| Embedding DB flush | `37.737 s` |
| Redis payload serialization | `31.242 s` |

This cycle starts because `221.777 s` is larger than the remaining measured
post-stage embedding/Redis buckets. Cycle 20 remains valid, but its own
ordering rule says it should move earlier only when post-stage tail is the next
dominant limiter. Current accepted evidence points first to the pose tail.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Doc | `docs/production_inference_benchmark.md` | Section 31 records the Cycle 16.B accepted benchmark and remaining bottleneck table. |
| Doc | `docs/cycle_16b_redis_side_effect_coalescing_results.md` | Records accepted Cycle 16.B metrics, rollback, parity, and next bottleneck. |
| Doc | `docs/inference_parallelization_plan.md` | Governs active optimization-cycle ordering and production benchmark authority. |
| Doc | `docs/crop_frame_optimization_execution.md` | Holds the sorted pending cycle map that this cycle restages. |
| Doc | `docs/cycle_20_streaming_persistence_embedding_overlap_investigation.md` | Explains why streaming persistence/embedding stays behind this investigation unless post-stage tail becomes dominant. |
| Doc | `docs/cycle_21_celery_concurrency_scaling_investigation.md` | Explains why worker count changes require proof of independent work and cannot be blind. |
| File | `backend/apps/video_analysis/tasks.py` | Owns the offline crop-frame loop, pose dispatch/finalization, upload timing, Step 3, and task lifecycle. |
| File | `backend/apps/video_analysis/services/inference_orchestrator.py` | Owns model dispatch orchestration paths that may contribute to the pose tail. |
| File | `backend/apps/pipeline/services/triton_client.py` | Owns Triton request timing and client transport behavior. |
| Tool | `tools/prod/prod_collect_benchmark_metrics.py` | Collects production benchmark metrics that must include any new pose-tail fields. |
| Tool | `tools/prod/prod_watch_benchmark_metrics.sh` | Live bounded watcher that must surface new metrics during production runs. |
| Constitution | `.specify/memory/constitution.md` | Requires production Linux benchmark authority and forbids unbenchmarked acceptance/rejection. |

## Current Evidence

| Evidence item | Value | Interpretation |
|---|---:|---|
| Cycle 16.B replay | `cycle16b-redis-coalescing-20260603T025823Z` | Latest accepted production profile. |
| Cycle 16.B job | `b2dfa987-afc5-4b96-ab12-6799b149ac25` | Completed `4541/4541` frames. |
| DB FPS | `5.347791` | Current accepted throughput baseline. |
| Behavior RTT mean / p95 | `86.532 ms` / `132.870 ms` | Inference protection baseline. |
| GPU avg / peak | `11.030 %` / `52.000 %` | GPU baseline. |
| Step 2 frame wall | `460.698 s` | Frame loop wall after accepted overlap. |
| Step 2 through pose upload | `682.475 s` | Frame loop plus pose upload/drain wall. |
| Difference | `221.777 s` | The tail this cycle must decompose. |

The evidence does not yet prove whether the tail is pose model execution, pose
request queuing, pose postprocessing, upload/enrichment, task orchestration, or
a timing-boundary artifact. This cycle must measure before optimizing.

## Hypotheses To Test

These are hypotheses, not conclusions:

| Candidate | What would prove it |
|---|---|
| Pose request queue tail | Pose queue wait or future-drain wall dominates after frame-loop completion. |
| Pose Triton/server execution tail | Pose RTT/server execution consumes most of the `221.777 s` tail. |
| Pose postprocess/upload tail | RTMPose parsing, keypoint enrichment, row preparation, or upload bookkeeping dominates after RTT. |
| Orchestration boundary artifact | The through-pose timer includes non-pose work that belongs in Step 3, render, or task finalization. |
| Backpressure from accepted overlap | Behavior overlap protects RTT but leaves pose work delayed behind frame preparation or tracking. |

## Required Measurements

Cycle 14.A must add guarded instrumentation before any optimization candidate:

| Measurement | Required field |
|---|---|
| Pose dispatch count | `pose_tail.pose_requests` |
| Pose crop/person count | `pose_tail.pose_subjects` |
| Pose enqueue wall | `pose_tail.enqueue_wall_s` |
| Pose queue wait | `pose_tail.queue_wait_s` |
| Pose Triton RTT | `pose_tail.rtt_wall_s`, mean, p95 |
| Pose server execution | `pose_tail.server_exec_s`, mean, p95 when Triton stats expose it |
| Pose response parse | `pose_tail.response_parse_s` |
| Pose postprocess | `pose_tail.postprocess_s` |
| Pose upload/enrichment | `pose_tail.upload_wall_s` |
| Future-drain wall after last frame | `pose_tail.drain_after_frame_loop_s` |
| Step-boundary timestamps | frame loop done, pose drain done, Step 3 start, render start, embedding handoff |
| Inference protection | behavior RTT mean/p95, Step 2 wall, GPU util |
| Correctness | DB rows, StudentTracks, embeddings, model agreement |

The production watcher must display the bounded fields without overflowing
table cells, following the existing watcher text-fit rule.

## Expected Gain

No gain is claimed in Phase A. If the full `221.777 s` tail is real and
optimizable, the theoretical total-elapsed ceiling is material. The first
implementation candidate must state a smaller evidence-backed expected gain
after the decomposition identifies the dominant sub-stage.

## Risk Assessment

| Risk | Level | Mitigation |
|---|---|---|
| Profiling overhead changes timings | Medium | Guard with a disabled-by-default env flag and compare against baseline. |
| Misattributing Step 3 or render time to pose | Medium | Add explicit step-boundary timestamps. |
| Inference RTT regression | Medium | Require behavior RTT and GPU metrics in every benchmark. |
| Correctness drift | High for any later optimization | Phase A is measurement-only; later candidates must pass DB/model parity. |
| Worker-count overcorrection | Medium | Do not raise workers until Cycle 21 matrix is justified by independent work. |

## Rollback Strategy

Phase A instrumentation must be controlled by a disabled-by-default flag, for
example:

```text
POSE_TAIL_PROFILING=0
```

Rollback for the profiling candidate is setting the flag to `0` and restarting
Celery workers. No schema migration, engine rebuild, or Triton route change is
allowed in Phase A.

## Acceptance Criteria

Cycle 14.A can only conclude after a real production benchmark:

| Gate | Required evidence |
|---|---|
| Production authority | Completed `combined.mp4` benchmark on native Linux RTX 5090. |
| Evidence bundle | Replay key, job ID, deployed SHA, env fingerprint, GPU CSV, metrics JSON/Markdown, wrapper log. |
| Tail decomposition | The `221.777 s` through-pose tail is split into measured sub-stages. |
| Inference protection | Step 2 wall, behavior RTT mean/p95, GPU util, and memory recorded. |
| Correctness | DB row parity, StudentTracks, embeddings, and model-agreement F1 recorded. |
| Decision | Results doc states `MEASUREMENT COMPLETE`; no optimization accepted unless a later candidate improves a real benchmark. |

## Ordering Decision

Cycle 14.A is now the next cycle to start. It moves ahead of compact
server-side postprocessing, video sharding, Redis Streams, Redis scripts, Cycle
20 streaming persistence, and Cycle 21 worker scaling because accepted Cycle
16.B evidence shows the Step 2 through-pose tail is the largest unresolved
measured bucket.

The older compact server-side postprocessing idea remains valid as a later
Cycle 14.B candidate only if Cycle 14.A proves the tail is caused by
pose/behavior client-server transfer or server execution that compact
postprocessing can reduce.
