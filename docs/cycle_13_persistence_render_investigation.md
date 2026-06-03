# Cycle 13 Persistence And Render Investigation

**Last updated:** 2026-06-03

**Status:** Phase A investigation started. No code has been changed for Cycle
13. No optimization decision exists until a production Linux RTX 5090 benchmark
on `combined.mp4` completes and is compared against the latest accepted
baseline.

## Problem Statement

Cycle 12.C accepted the single-inflight behavior-overlap path and reduced the
Step 2 inference wall without regressing behavior RTT or correctness. The next
ordered cycle is Cycle 13: persistence/render cleanup. This cycle targets
post-inference total-wall time, not the GPU inference hot path.

The known accepted baseline at the start of this investigation is:

| Metric | Cycle 9b Top-K baseline | Cycle 12.C accepted baseline |
|---|---:|---:|
| Replay key | `cycle9b-topk-crop-frame-20260602T041900` | `cycle12-single-inflight-overlap-20260602T225821Z` |
| Job ID | `be4ba9ee-4786-48e9-8334-28feb237a1fb` | `069a217f-fa43-48cc-bf18-c946d53bb3ee` |
| Total DB-completed elapsed | `1022.952 s` | `935.516 s` |
| DB-completed FPS | `4.439` | `4.854` |
| Step 2 wall | `540.399 s` | `459.461 s` |
| Behavior RTT mean | `84.865 ms` | `83.936 ms` |
| GPU avg util | `9.344 %` | `10.332 %` |
| Tracks | `53` | `53` |
| Model agreement | baseline | `>=99.716 % F1@IoU0.5` |

The SLA for `combined.mp4` remains `451 s` total wall. Cycle 13 can improve
total wall, but it is not expected to close the full SLA gap alone.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Doc | `docs/cycle_12_single_inflight_overlap_results.md` | Latest accepted production result and baseline. |
| Doc | `docs/production_inference_benchmark.md` | Production benchmark authority and Cycle 12.C comparison table. |
| Doc | `docs/cycles_9_to_12_implementation_playbook.md` | Historical Cycle 13 plan: render and persistence cleanup. |
| Doc | `docs/crop_frame_optimization_execution.md` | Per-cycle execution log and projected Cycle 13 placement. |
| File | `backend/apps/video_analysis/tasks.py` | Offline pipeline, Step 3 persistence, render, embedding orchestration. |
| File | `backend/apps/tracking/embeddings.py` | `persist_embeddings_bulk` and Redis embedding helpers used after inference. |
| Tool | `tools/prod/prod_collect_benchmark_metrics.py` | Captures benchmark metrics and correctness evidence. |
| Tool | `tools/prod/prod_watch_benchmark_metrics.sh` | Live benchmark watch tool that should surface Cycle 13 metrics. |

## Root Cause Hypothesis

Cycle 13 should target the post-inference wall that remains after Step 2.
The prior roadmap names two low-risk cleanup candidates:

| Candidate | Lever | Expected effect |
|---|---|---|
| Parallel render writers | Single-process post-stage orchestration | Reduce render wall without touching inference outputs |
| PostgreSQL `COPY FROM` or stronger bulk insert path for embeddings | Persistence write overhead | Reduce embedding/persistence stage wall if row insert overhead remains material |

This is a cleanup cycle. If the current production artifacts show that render
and embedding persistence are not material after Cycle 12.C, Cycle 13 must be
de-ranked rather than implemented by inertia.

## Required Phase A Measurements Before Code

Before implementation, collect the current Cycle 12.C post-stage decomposition
from production evidence:

| Measurement | Source |
|---|---|
| Step 2 wall | `inference_audit.json` and collected metrics bundle |
| Step 3 persistence wall | `inference_audit.json`, job metadata, logs |
| Render wall | `inference_audit.json`, render logs, benchmark summary |
| Embedding wall | `inference_audit.json`, embedding task logs |
| Frame/Detection/BoundingBox/Embedding rows | PostgreSQL via benchmark collector |
| Worker RSS and CPU | benchmark metrics / watcher output |
| GPU utilization | GPU CSV from benchmark wrapper |
| Redis command wall, if available | Cycle 16.A will add this later; not required for Cycle 13 start |

If these measurements cannot bound Cycle 13's maximum possible gain, the next
action is to improve the measurement script, not implement persistence/render
changes.

## Expected Gain

The existing roadmap estimates Cycle 13 as roughly a `~20 s` total-wall cleanup.
Against the Cycle 12.C baseline (`935.516 s`), that is about `2.1 %` total-wall
improvement. Therefore Cycle 13 should be judged as a cleanup and reliability
cycle, not as the primary SLA-closing cycle.

## Risk Assessment

| Risk | Level | Mitigation |
|---|---|---|
| DB row-count regression | High | Compare Frame, Detection, BoundingBox, FrameEmbedding, StudentTrack counts against baseline. |
| Render artifact mismatch | Medium | Verify output file exists, duration/frame count is stable, and no render errors appear. |
| PostgreSQL COPY transaction semantics | Medium | Keep PostgreSQL-only tests and rollback to `bulk_create` path. |
| Parallel render CPU contention | Medium | Record CPU/RSS and total wall; do not accept if inference or embedding regresses. |
| Redis side-effect confusion | Low | Redis is not part of Cycle 13 implementation unless Phase A proves it belongs here. |

## Rollback Strategy

Any Cycle 13 implementation must be guarded or trivially revertible:

| Change type | Rollback |
|---|---|
| Render parallelism | Env flag to restore serial render, or commit revert |
| COPY-based embedding persistence | Env flag to restore existing `persist_embeddings_bulk` path, or commit revert |
| Benchmark tooling | Keep additive only; revert if metrics are misleading |

## Acceptance Criteria

Cycle 13 may be accepted only if all conditions are true:

1. Production Linux RTX 5090 benchmark on `combined.mp4` completes.
2. Candidate is compared against Cycle 12.C, not the older Cycle 9b Top-K
   baseline.
3. Total wall or targeted post-stage wall improves measurably.
4. DB parity passes for frames, detections, bounding boxes, embeddings, and
   tracks.
5. Model-agreement/correctness stays within the active gate.
6. Render output exists and no render errors are recorded.
7. GPU utilization, CPU, RSS, and memory are recorded.
8. `docs/production_inference_benchmark.md`,
   `docs/crop_frame_optimization_execution.md`,
   `docs/inference_parallelization_plan.md`, and `AGENTS.md` are updated with
   the decision table.

## Immediate Next Action

Run a Cycle 13 Phase A baseline extraction against the Cycle 12.C evidence
bundle. If post-stage wall is below the expected gain threshold, de-rank Cycle
13 and move to the next planned inference-wall cycle. If post-stage wall is
material, implement only the highest measured Cycle 13 candidate and benchmark
it on production.
