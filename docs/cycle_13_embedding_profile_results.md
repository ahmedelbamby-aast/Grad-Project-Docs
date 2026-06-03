# Cycle 13.A Embedding Stage Profile Results

**Last updated:** 2026-06-03

**Status:** HYPOTHESIS_ONLY / MEASUREMENT COMPLETE. This production Linux
RTX 5090 run enabled only `EMBEDDING_STAGE_PROFILING=1` on the accepted
Cycle 12.C profile. It does not accept, reject, skip, or close a Cycle 13
optimization.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Production replay | `cycle13-embedding-profile-20260603T003853Z` | Candidate profiling run on `combined.mp4`. |
| Production job | `aa2fe7a9-b3fb-49d7-92a3-eca41c894dcd` | Completed `4541/4541` frames with profiling enabled. |
| Metrics JSON | `backend/logs/cycle13-embedding-profile-20260603T003853Z/embedding_profile_metrics.json` | Main benchmark and embedding profile evidence. |
| Metrics Markdown | `backend/logs/cycle13-embedding-profile-20260603T003853Z/embedding_profile_metrics.md` | Human-readable evidence bundle. |
| Agreement JSON | `backend/logs/cycle13-embedding-profile-20260603T003853Z/model_agreement_cycle12c_vs_embedding_profile.json` | Baseline-agreement correctness evidence. |
| Run manifest | `backend/logs/cycle13-embedding-profile-20260603T003853Z/run_manifest.tsv` | Wrapper status and artifact paths. |
| Baseline replay | `cycle12-single-inflight-overlap-20260602T225821Z` | Accepted Cycle 12.C comparison target. |
| Baseline job | `069a217f-fa43-48cc-bf18-c946d53bb3ee` | Accepted Cycle 12.C production job. |

## Production Comparison

| Metric | Cycle 12.C baseline | Cycle 13.A profile run | Delta |
|---|---:|---:|---:|
| DB-completed elapsed | `935.516 s` | `945.570 s` | `+1.075 %` |
| DB-completed FPS | `4.854005` | `4.802395` | `-1.063 %` |
| Step 2 frame wall | `459.461 s` | `463.905 s` | `+0.967 %` |
| Step 2 through pose upload | `680.619 s` | `688.003 s` | `+1.085 %` |
| Run-complete wall | `746.193 s` | `755.494 s` | `+1.246 %` |
| Behavior RTT mean | `83.936 ms` | `85.094 ms` | `+1.380 %` |
| Behavior RTT p95 | `130.200 ms` | `131.637 ms` | `+1.104 %` |
| GPU avg util | `10.332 %` | `9.709 %` | `-6.030 %` |
| GPU peak util | `53.000 %` | `51.000 %` | `-3.774 %` |
| Peak VRAM | `15,725 MiB` | `15,731 MiB` | `+0.038 %` |
| Detection rows | `72,744` | `72,744` | `0.000 %` |
| BBox rows | `72,744` | `72,744` | `0.000 %` |
| Embedding rows | `72,578` | `72,578` | `0.000 %` |
| Student tracks | `53` | `53` | `0.000 %` |
| Embedding created span | `187.139 s` | `187.884 s` | `+0.398 %` |

Model agreement against Cycle 12.C stayed exact for the persisted model
outputs:

| Model | F1@IoU0.5 | Precision | Recall | Boxes |
|---|---:|---:|---:|---:|
| `attention_tracking` | `100.000 %` | `100.000 %` | `100.000 %` | `11,770` |
| `hand_raising` | `100.000 %` | `100.000 %` | `100.000 %` | `8,799` |
| `person_detection` | `100.000 %` | `100.000 %` | `100.000 %` | `19,162` |
| `sitting_standing` | `100.000 %` | `100.000 %` | `100.000 %` | `33,013` |

## Embedding Sub-Stage Profile

The profiling run measured `188,619.901 ms` total embedding wall. The largest
sub-stage buckets were:

| Sub-stage | Wall | Share of embedding wall | Evidence |
|---|---:|---:|---|
| Track lookup | `66,223.225 ms` | `35.11 %` | `embedding.summary.profile.track_lookup_ms` |
| Redis flush | `59,304.070 ms` | `31.44 %` | `embedding.summary.profile.redis_flush_ms` |
| DB flush | `38,467.399 ms` | `20.39 %` | `embedding.summary.profile.db_flush_ms` |
| Existing-embedding checks | `14,526.763 ms` | `7.70 %` | `embedding.summary.profile.embedding_exists_check_ms` |
| Frame detection query | `8,884.434 ms` | `4.71 %` | `embedding.summary.profile.frame_detection_query_ms` |
| cv2 seek/read | `723.663 ms` | `0.38 %` | `embedding.summary.profile.cv2_seek_read_ms` |
| Redis read | `10.842 ms` | `0.01 %` | `embedding.summary.profile.redis_read_ms` |
| Vector compute | `6.750 ms` | `0.00 %` | `embedding.summary.profile.vector_compute_ms` |

The run also measured:

| Metric | Value |
|---|---:|
| Frames iterated | `4,541` |
| Frames with candidates | `4,541` |
| Detections seen | `72,744` |
| Detections to embed | `72,578` |
| Generated vectors | `53` |
| Process-cache hits | `72,525` |
| Redis-cache hits | `0` |
| DB flush count | `146` |
| Redis flush count | `146` |
| Redis side-effect rows | `72,578` |
| Bulk batch size | `500` |
| Track vector cache size | `53` |

## Decision

This run is not an optimization decision. It is the production measurement
needed to select the first Cycle 13.B candidate.

| Question | Evidence | Decision |
|---|---|---|
| Did the profiling run complete on production Linux RTX 5090? | Replay `cycle13-embedding-profile-20260603T003853Z`, job `aa2fe7a9-b3fb-49d7-92a3-eca41c894dcd`, status `completed`. | Evidence is valid. |
| Did instrumentation preserve correctness? | DB rows and all model-agreement rows matched Cycle 12.C exactly. | Correctness gate suitable for the next candidate. |
| Did profiling improve performance? | No optimization was deployed; elapsed and FPS drifted by about `1 %`. | No acceptance/rejection allowed. |
| Which measured bucket is the first candidate? | Track lookup is `66.223 s`, the largest single bucket. | Start Cycle 13.B: prefetch-aware embedding track resolution. |
| Which bucket is next after that? | Redis flush is `59.304 s` over `72,578` side-effect rows. | Keep Redis coalescing as the follow-up if Cycle 13.B preserves correctness. |

## Next Candidate

Cycle 13.B must investigate and benchmark a guarded track-lookup change before
shipping:

1. Reuse the already-prefetched `bounding_boxes__student_track` relation in
   `generate_embeddings` instead of issuing per-detection related-manager
   queries.
2. Preserve the current selected `StudentTrack` semantics.
3. Record track-lookup profile counters before and after.
4. Run a full `combined.mp4` production benchmark against Cycle 12.C.
5. Accept only if total wall or embedding wall improves and DB/model parity
   remains intact.

