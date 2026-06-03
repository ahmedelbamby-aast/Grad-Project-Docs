# Cycle 13.B Embedding Track Lookup Investigation

**Last updated:** 2026-06-03

**Status:** STAGED / INVESTIGATION COMPLETE BEFORE CODE. No optimization is
accepted, rejected, skipped, or closed until a real production Linux RTX 5090
benchmark on `combined.mp4` completes against the accepted Cycle 12.C baseline.

## Problem Statement

Cycle 13.A profiling measured the embedding stage after the accepted Cycle
12.C inference profile. The largest measured embedding sub-stage was track
lookup: `66,223.225 ms` inside `188,619.901 ms` total embedding wall.

The code path in `backend/apps/video_analysis/tasks.py::generate_embeddings`
already loads each frame's detections with `prefetch_related(
'bounding_boxes__student_track')`, but then resolves each detection's track by
calling `detection.bounding_boxes.select_related('student_track').order_by(
'student_track__tracking_id').first()` in the need-pixels pass and again in
the persistence pass. That related-manager call bypasses the prefetched list
and issues repeated database work for detections that already have their
bounding boxes in memory.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Cycle 13.A results | `docs/cycle_13_embedding_profile_results.md` | Production profile that selected this candidate. |
| Metrics JSON | `backend/logs/cycle13-embedding-profile-20260603T003853Z/embedding_profile_metrics.json` | Contains `track_lookup_ms=66223.225`. |
| Code | `backend/apps/video_analysis/tasks.py` | `generate_embeddings` performs the measured track lookup work. |
| Code | `backend/apps/tracking/embeddings.py` | Bulk embedding persistence path used after track resolution. |
| Baseline replay | `cycle12-single-inflight-overlap-20260602T225821Z` | Accepted baseline for comparison. |
| Baseline job | `069a217f-fa43-48cc-bf18-c946d53bb3ee` | Accepted baseline job. |
| Candidate wrapper | `tools/prod/prod_run_cycle13_track_lookup_benchmark.sh` | Required production benchmark wrapper for this candidate. |

## Root Cause

The measured `track_lookup_ms` is high because the embedding loop repeats
small ORM lookups per detection even though the required relation is already
prefetched per frame.

Relevant measured scale:

| Metric | Value |
|---|---:|
| Frames iterated | `4,541` |
| Detections seen | `72,744` |
| Detections to embed | `72,578` |
| Unique generated vectors | `53` |
| Process-cache hits | `72,525` |
| Track lookup wall | `66.223 s` |

Since only `53` track vectors are generated and `72,525` embeddings reuse a
process-local vector, repeated track resolution is overhead around a successful
reuse strategy rather than model compute.

## Candidate

Add a guarded embedding-stage track resolver:

| Change | Guard | Expected effect |
|---|---|---|
| Resolve the first `BoundingBox` with `student_track` from the prefetched `detection.bounding_boxes.all()` list. | `EMBEDDING_PREFETCH_TRACK_LOOKUP=1` | Reduce per-detection ORM query wall while preserving selected track semantics. |
| Add profile counters for strategy and prefetch hits/misses. | Same guard and existing `EMBEDDING_STAGE_PROFILING=1` | Prove whether the measured `track_lookup_ms` moves. |
| Benchmark with the accepted Cycle 12.C profile. | Wrapper script sets and restores the flag. | Decision evidence only. |

This candidate does not change embedding vectors, Redis key schema, DB schema,
model inference, frame ordering, or render behavior.

## Expected Gain

Upper bound is the measured `66.223 s` track-lookup bucket. A realistic
accepted result should reduce embedding wall by a material fraction of that
bucket without increasing DB rows or changing model agreement. Against the
Cycle 12.C total elapsed (`935.516 s`), the full bucket is `7.08 %`; therefore
this is a cleanup candidate, not the final SLA-closing change.

## Risk Assessment

| Risk | Level | Mitigation |
|---|---|---|
| Wrong track selected from prefetched boxes | Medium | Preserve existing ordering by choosing the lowest `student_track.tracking_id`; compare StudentTrack count and model agreement. |
| Hidden query reintroduced when prefetch is absent | Low | Keep the fallback resolver and count prefetch misses in profiling. |
| Embedding row regression | High | Compare `FrameEmbedding`, `Detection`, `BoundingBox`, and `StudentTrack` counts against Cycle 12.C. |
| Redis semantics change | Low | Do not change Redis writes in Cycle 13.B. Redis coalescing is a later candidate. |

## Rollback Strategy

Set `EMBEDDING_PREFETCH_TRACK_LOOKUP=0` and restart Celery workers. The wrapper
must restore the env flag after the benchmark regardless of success or failure.

## Acceptance Criteria

Cycle 13.B can be accepted only if all conditions are true:

1. Production Linux RTX 5090 benchmark on `combined.mp4` completes.
2. Candidate is compared against Cycle 12.C and Cycle 13.A profile evidence.
3. Embedding wall, total wall, or DB-completed FPS improves measurably.
4. Track lookup wall decreases in the profiling table.
5. DB parity holds for frames, detections, bounding boxes, embeddings, and
   student tracks.
6. Model-agreement F1@IoU0.5 remains within the active gate.
7. Production env rollback proof is recorded.
8. `AGENTS.md`, `docs/production_inference_benchmark.md`,
   `docs/inference_parallelization_plan.md`,
   `docs/crop_frame_optimization_execution.md`, and
   `docs/cycle_9_and_10_improvements_todo.md` are updated.

