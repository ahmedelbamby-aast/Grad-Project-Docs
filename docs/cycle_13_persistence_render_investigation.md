# Cycle 13 Persistence And Render Investigation

**Last updated:** 2026-06-03

**Status:** Phase A baseline extraction and Cycle 13.A embedding profiling are
complete. Cycle 13.B prefetch-aware embedding track lookup is the next staged
candidate. No optimization decision exists until a production Linux RTX 5090
benchmark on `combined.mp4` completes and is compared against the latest
accepted baseline.

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
| Tool | `tools/prod/prod_run_cycle13_embedding_profile_benchmark.sh` | Reproducible production wrapper for Cycle 13 embedding profiling. |
| File | `backend/config/settings/base.py` | Defines `EMBEDDING_STAGE_PROFILING`, the measurement-only Cycle 13 flag. |

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

## Phase A Production Baseline Extraction

Cycle 13 Phase A was extracted from the accepted Cycle 12.C production run:

| Item | Value |
|---|---|
| Replay key | `cycle12-single-inflight-overlap-20260602T225821Z` |
| Job ID | `069a217f-fa43-48cc-bf18-c946d53bb3ee` |
| Evidence bundle | `backend/logs/cycle12-single-inflight-overlap-20260602T225821Z/` |
| Recollected metrics | `backend/logs/cycle12-single-inflight-overlap-20260602T225821Z/cycle13_phase_a_recollected_metrics.json` |
| Inference audit | `backend/data/videos/069a217f-fa43-48cc-bf18-c946d53bb3ee/inference_audit.json` |
| GPU CSV | `backend/logs/gpu_monitor_bench_20260603T015844.csv` |

Measured decomposition:

| Stage | Measurement | Evidence source |
|---|---:|---|
| DB-completed elapsed | `935.516 s` | Recollected metrics JSON |
| Run-complete wall | `746.193 s` | `run.complete` in inference audit |
| Step 2 frame wall | `459.461 s` | `step2.frame_stage_timings` |
| Step 2 through pose upload | `680.619 s` | `step2.pose_upload` |
| Step 3 persistence wall | `39.820 s` | `step3.persistence.start` -> `step3.persistence.complete` |
| Render wall | `25.692 s` | `step3.persistence.complete` -> `step4.render.complete` |
| Post-run-complete tail | `189.323 s` | DB elapsed minus run-complete wall |
| Embedding created-at span | `187.139 s` | PostgreSQL `FrameEmbedding.created_at` min/max |
| Embedding rows | `72,578` | PostgreSQL `FrameEmbedding` count |
| Embedding reused rows | `72,525` | `job.metadata.embedding_summary` |
| DB rows | `4,541` frames, `72,744` detections, `72,744` boxes, `53` tracks | PostgreSQL counters |
| GPU utilization | `10.332 %` avg, `53.000 %` peak, `15,725 MiB` peak VRAM | GPU CSV summary |

Decision from Phase A:

| Candidate | Evidence | Phase A decision |
|---|---|---|
| Render-only optimization | Render wall is `25.692 s`, or `2.75 %` of DB-completed elapsed. | De-rank as first Cycle 13 candidate. It cannot recover the main post-stage tail alone. |
| Step 3 persistence-only optimization | Persistence wall is `39.820 s`, or `4.26 %` of DB-completed elapsed. | Worth keeping, but not first without sub-stage proof. |
| Embedding/finalization optimization | Embedding created-at span is `187.139 s`; post-run-complete tail is `189.323 s`. | Start Cycle 13 with embedding sub-stage profiling. |

The first implementation is therefore measurement-only:

| Change | Purpose | Decision authority |
|---|---|---|
| `EMBEDDING_STAGE_PROFILING=1` | Record embedding sub-stage wall inside `embedding_summary.profile`. | Not an optimization and not acceptance evidence by itself. |
| `tools/prod/prod_run_cycle13_embedding_profile_benchmark.sh` | Reproduce a full `combined.mp4` production run against the Cycle 12.C baseline. | Required before selecting a Cycle 13 optimization candidate. |
| Collector/watcher updates | Surface embedding DB span, DB flush, Redis flush/read, existence checks, track lookup, vector compute, and cv2 read time. | Evidence collection only. |

No render, PostgreSQL `COPY`, Redis coalescing, or finalization optimization is
accepted, rejected, skipped, or closed by this Phase A extraction.

## Cycle 13.A Production Embedding Profile

The measurement-only production run completed:

| Item | Value |
|---|---|
| Replay key | `cycle13-embedding-profile-20260603T003853Z` |
| Job ID | `aa2fe7a9-b3fb-49d7-92a3-eca41c894dcd` |
| Metrics | `backend/logs/cycle13-embedding-profile-20260603T003853Z/embedding_profile_metrics.json` |
| Agreement | `backend/logs/cycle13-embedding-profile-20260603T003853Z/model_agreement_cycle12c_vs_embedding_profile.json` |
| Result doc | `docs/cycle_13_embedding_profile_results.md` |
| Status | `HYPOTHESIS_ONLY` |

Comparison against Cycle 12.C:

| Metric | Cycle 12.C baseline | Cycle 13.A profile run | Delta |
|---|---:|---:|---:|
| DB-completed elapsed | `935.516 s` | `945.570 s` | `+1.075 %` |
| DB-completed FPS | `4.854005` | `4.802395` | `-1.063 %` |
| Step 2 frame wall | `459.461 s` | `463.905 s` | `+0.967 %` |
| Behavior RTT mean | `83.936 ms` | `85.094 ms` | `+1.380 %` |
| GPU avg util | `10.332 %` | `9.709 %` | `-6.030 %` |
| Detection rows | `72,744` | `72,744` | `0.000 %` |
| BBox rows | `72,744` | `72,744` | `0.000 %` |
| Embedding rows | `72,578` | `72,578` | `0.000 %` |
| Student tracks | `53` | `53` | `0.000 %` |
| Model agreement | accepted baseline | `100.000 %` F1@IoU0.5 for all persisted models | no regression |

Measured embedding sub-stages:

| Sub-stage | Wall | Decision |
|---|---:|---|
| Track lookup | `66.223 s` | First Cycle 13.B candidate. |
| Redis flush | `59.304 s` | Follow-up only after track lookup is benchmarked. |
| DB flush | `38.467 s` | Keep as later persistence candidate. |
| Existing-embedding checks | `14.527 s` | Lower priority. |
| Frame detection query | `8.884 s` | Lower priority. |
| cv2 seek/read | `0.724 s` | Not material after Cycle 8. |
| Vector compute | `0.007 s` | Not a target. |

Cycle 13.A selected Cycle 13.B because `track_lookup_ms` is the largest
measured sub-stage and code inspection ties it to repeated per-detection
related-manager lookups in `generate_embeddings`.

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

Implement only the guarded Cycle 13.B track-lookup candidate described in
`docs/cycle_13_embedding_track_lookup_investigation.md`, then run:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_run_cycle13_track_lookup_benchmark.sh \
  --tag cycle13-track-lookup-$(date -u +%Y%m%dT%H%M%SZ)
```

Then watch it with:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_watch_benchmark_metrics.sh --latest --interval 10 --clear
```

Only after that production benchmark records the comparison table may Cycle
13.B be accepted, rejected, or marked as needing further iteration.
