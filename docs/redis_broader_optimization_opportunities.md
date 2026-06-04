# Redis Broader Optimization Opportunities

**Last updated:** 2026-06-04

**Status:** Investigation and roadmap staging only. No Redis-backed strategy in
this document is accepted, rejected, skipped, or closed until a completed
production Linux RTX 5090 benchmark on `combined.mp4` compares it against the
latest accepted baseline.

## Problem Statement

Redis is already present in the runtime, but Cycle 7 proved that Redis work can
be overestimated. Caching Redis clients improved the embedding stage only
modestly because the client factory cost was much lower than the original
hypothesis. Broader Redis use is still worth investigating, but only where it
can move a benchmark metric that the optimization plan already records:

| Metric | Redis can plausibly affect? | Required proof before implementation |
|---|---:|---|
| Total wall time | Yes | Stage-level wall shows Redis command cost or DB/progress contention is material |
| Embedding wall | Yes | Redis write/read command count and wall time during `generate_embeddings` are material |
| Persistence wall | Possible | DB writes or progress heartbeats are blocking on non-authoritative side effects |
| Benchmark observability | Yes | Redis stream/status snapshots improve sampling without adding DB pressure |
| Step 2 inference wall | Unlikely | Redis does not remove Triton compute, gRPC wait, or Python crop orchestration |
| GPU utilization | Indirect only | Redis must reduce CPU/post-stage blocking enough to expose more GPU work |

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| External | <https://redis.io/docs/latest/develop/using-commands/pipelining/> | Official Redis explanation that pipelining batches commands and reduces per-command RTT/socket overhead. |
| External | <https://redis.io/docs/latest/develop/data-types/streams/> | Official Redis Streams reference for append-only event logs and consumer groups. |
| External | <https://redis.io/docs/latest/develop/pubsub/> | Official Redis Pub/Sub delivery semantics; Pub/Sub is at-most-once and unsuitable for benchmark evidence authority. |
| File | `backend/apps/tracking/embeddings.py` | Existing Redis embedding helpers, including `cache_embedding`, `cache_job_track_embedding`, and a per-track pipeline. |
| File | `backend/apps/video_analysis/tasks.py` | Offline embedding loop, Redis status helpers, frame batching cache, job status writes, and benchmark-stage control flow. |
| File | `backend/apps/tracking/tracker.py` | Redis-backed identity lifecycle store for aliases and lifecycle snapshots. |
| File | `backend/config/settings/base.py` | Redis cache/channel/Celery configuration source. |
| Doc | `docs/crop_frame_optimization_execution.md` | Cycle 7 and Cycle 8 evidence for Redis client caching and embedding-stage redesign. |
| Doc | `docs/production_inference_benchmark.md` | Production benchmark authority and Cycle 7 measured result. |
| Doc | `docs/cycle_12_single_inflight_overlap_results.md` | Latest accepted baseline before the Redis roadmap was staged. |
| Doc | `docs/cycle_13c_redis_db_side_effect_measurement_results.md` | Completed Cycle 16.A command-cost measurement and next-candidate decision. |
| Doc | `docs/cycle_16b_redis_side_effect_coalescing_results.md` | Accepted Cycle 16.B production benchmark and decision. |

## Current Repo Evidence

Redis is already used for:

| Area | Existing behavior | Evidence |
|---|---|---|
| Celery broker/result backend | Redis-backed task dispatch and task results | `backend/config/celery.py`, `backend/config/settings/base.py` |
| Channels/cache | Django Channels and Django cache backend | `backend/config/settings/base.py` |
| Embedding cache | Latest and per-job track embeddings | `backend/apps/tracking/embeddings.py` |
| Embedding stage side effects | `generate_embeddings` flushes DB rows and then writes Redis embedding histories | `backend/apps/video_analysis/tasks.py` |
| Tracking lifecycle | Alias and lifecycle snapshots with TTL | `backend/apps/tracking/tracker.py` |
| Job status / visibility / telemetry snapshots | Status, model toggles, tracking-visibility, frame-batching cache | `backend/apps/video_analysis/tasks.py`, `backend/apps/video_analysis/views.py` |

Cycle 7 is the cautionary precedent: replacing repeated client construction
with cached clients was measured and accepted, but the gain was much smaller
than projected. Therefore every broader Redis cycle must first measure Redis
command count, command wall time, bytes, and Redis memory impact on production.

## Redis Strategies Worth Adding To The Roadmap

### Cycle 16.A — Redis Command-Cost Instrumentation

**Status:** measurement complete / `HYPOTHESIS_ONLY`.

**Purpose:** measure Redis command volume and wall time during a real
`combined.mp4` benchmark without changing behavior.

**Mechanism:** wrap the existing Redis clients used by the offline job with an
optional instrumentation layer behind a flag such as
`OFFLINE_REDIS_COMMAND_PROFILING=1`. Record per-stage counters:

| Counter | Why |
|---|---|
| `redis_command_count` | Shows whether Redis is still a meaningful command source after Cycle 8. |
| `redis_pipeline_execute_count` | Distinguishes one-command-at-a-time patterns from batched writes. |
| `redis_wall_ms` | Directly bounds the maximum possible performance gain. |
| `redis_bytes_written_estimate` | Detects oversized embedding/history payloads. |
| `redis_errors` | Ensures profiling does not hide degraded Redis behavior. |
| `redis_peak_memory` / `INFO memory` | Prevents hidden memory growth from stream/pipeline candidates. |

**Production result:** replay `cycle13c-redis-command-profile-20260603T020723Z`
/ job `aa246a4e-e0f9-471a-9ce3-74f343bbd1fb` completed. Redis server command
wall was only `530.485 ms`, while profiled Redis flush wall was `92.397 s`.
The measurement authorized Cycle 16.B client-side side-effect coalescing and
de-ranked Redis server tuning as the first fix.

### Cycle 16.B — Redis Pipeline Coalescing For Embedding/Tracking Side Effects

**Purpose:** reduce embedding-stage and total-wall cost because Cycle 16.A
proved the material wall is client-side helper/payload/pipeline overhead, not
Redis server command execution.

**Mechanism:** replace per-item helper calls in the embedding flush path with a
job-scoped Redis pipeline writer. The existing code currently flushes DB rows
in batches, then iterates pending Redis side effects. `cache_job_track_embedding`
already uses a pipeline internally, but `cache_embedding` and
`_cache_student_embedding_for_analysis` still add commands separately per item.

**Candidate implementation:**

| Change | Expected effect |
|---|---|
| Add `cache_embedding_batch(...)` in `backend/apps/tracking/embeddings.py` | Fold many `SET` operations into one pipeline execute per flush. |
| Add a job-scoped `_flush_student_embedding_analysis_batch(...)` in `tasks.py` | Fold `SET` / `RPUSH` / `EXPIRE` / `SADD` operations into one pipeline execute per flush. |
| Keep batch sizes bounded | Follow Redis pipelining guidance to avoid unbounded queued replies and memory growth. |
| Record counters into benchmark evidence | Prove command count, wall time, and memory impact. |

**Metric gate:** must improve total wall or embedding wall in production and
preserve embedding-row parity, track count, DB row counts, and model agreement.

**Production result:** replay `cycle16b-redis-coalescing-20260603T025823Z`
/ job `b2dfa987-afc5-4b96-ab12-6799b149ac25` completed. Redis flush wall
improved `59.874 s -> 35.970 s`, embedding profile wall improved
`121.681 s -> 97.505 s`, DB FPS improved `5.205675 -> 5.347791`, and
DB/model parity was exact. Pipeline executes fell from the Cycle 13.C measured
`72578` shape to `146`. Cycle 16.B is **ACCEPTED**; Redis server tuning remains
de-ranked because candidate Redis server command wall was only `194.039 ms`.

### Cycle 17 — Redis Streams For Non-Authoritative Progress And Benchmark Sampling

**Status:** Accepted as observability-only / not throughput in
`docs/cycle_17_redis_streams_progress_sampling_investigation.md`.

**Purpose:** improve benchmark observability and possibly reduce DB polling /
progress-write pressure during long jobs.

**Mechanism:** write lightweight job progress snapshots to a capped Redis
Stream, such as `bench:{replay_key}:events`, while PostgreSQL remains the
authoritative job state. `prod_watch_benchmark_metrics.sh` can read the stream
for high-frequency live updates and fall back to PostgreSQL for authoritative
terminal status.

**Why Streams instead of Pub/Sub:** Redis Pub/Sub is at-most-once; lost messages
are not replayed. Streams are append-only logs with consumer-group support, so
they are a better fit for benchmark sampling while still remaining
non-authoritative.

**Metric result:** production replay
`cycle17-redis-streams-20260604T025328Z` produced bounded Redis Stream evidence
with exact DB/model parity and rollback verified. DB FPS was neutral, so the
cycle is observability-only and must not be used to claim inference performance
improvement.

### Cycle 18 — Redis Boundary-State Cache For Future Video Sharding

**Current status:** Phase A contract only / runtime blocked. The current source
doc is
[`docs/cycle_18_redis_boundary_state_cache_investigation.md`](cycle_18_redis_boundary_state_cache_investigation.md).
Cycle 15.B1, Cycle 15.B1.C1, and Cycle 15.B1.C2 all failed production
identity/model-agreement gates, and Cycle 15.B2 is blocked. Redis must not be
used as a workaround for failed sharding correctness.

**Purpose:** support the future multi-process video sharding architecture by
storing short-lived segment boundary state for stitching.

**Mechanism:** if Cycle 15 selects video sharding, each segment worker writes
non-authoritative boundary snapshots to Redis:

| Snapshot | Use |
|---|---|
| first/last frame track IDs | Track stitching across segment boundaries |
| per-track latest embedding vector | ReID continuity between segment workers |
| segment completion marker | Stitcher waits for all segments without polling DB heavily |
| per-segment diagnostics | Reproducible evidence if stitching fails |

**Constraint:** Redis may only coordinate ephemeral state. PostgreSQL remains
the source of truth for final frames, detections, tracks, embeddings, and job
status.

### Cycle 19 — Redis Server-Side Scripts For Read/Compute/Write Hotspots

**Purpose:** evaluate Lua scripting only if Cycle 16.A finds a read-modify-write
Redis pattern that pipelining cannot remove.

**Mechanism:** Redis scripting can execute read/compute/write logic server-side.
The official pipelining docs distinguish this from plain pipelining: scripts
can avoid client round trips when a command depends on a previous read result.

**Metric gate:** only useful if the measured hotspot is Redis read/compute/write
latency, not Triton or PostgreSQL. Any script must be small, deterministic, and
covered by rollback to the non-script path.

## Strategies Not Worth Promoting Now

| Strategy | Decision |
|---|---|
| Cache model predictions in Redis across benchmark frames | Do not pursue. The project requires every frame signal/prediction; Redis prediction caching would invalidate inference benchmarks. |
| Replace PostgreSQL with Redis for final evidence | Forbidden. PostgreSQL is the authoritative relational store. |
| Redis Pub/Sub for benchmark evidence | Do not use as evidence authority because Pub/Sub is at-most-once. It is acceptable only for best-effort UI notifications. |
| Redis-only job completion status | Do not use. Redis may mirror status, but terminal job status must come from PostgreSQL. |

## Restaged Redis Roadmap

These cycles are now sorted by the completed Cycle 16.A measurement:

| Cycle | Title | Status | Primary metric target |
|---|---|---|---|
| 16.A | Redis command-cost instrumentation | MEASUREMENT COMPLETE / HYPOTHESIS_ONLY | Evidence quality; upper bound for Redis optimization |
| 16.B | Redis pipeline coalescing for embedding/tracking side effects | ACCEPTED | Embedding wall; total wall |
| 17 | Redis Streams for progress/benchmark sampling | ACCEPTED OBSERVABILITY-ONLY / NOT THROUGHPUT | Evidence quality; no throughput claim |
| 18 | Redis boundary-state cache for future sharding | PLANNED AFTER Cycle 15 decision | Sharding stitch stability; total wall if sharding is selected |
| 19 | Redis server-side scripts for measured read/compute/write hotspots | CONDITIONAL AFTER 16.B | Only a Redis hotspot that coalescing cannot remove |

## Acceptance Criteria For Any Redis Optimization

1. A production Linux RTX 5090 benchmark on `combined.mp4` completes.
2. The candidate is compared against the latest accepted baseline, currently
   Cycle 12.C single-inflight behavior overlap at the time this document was
   created.
3. The comparison table includes FPS, total wall, Step 2 wall, post-stage wall,
   Redis command counters, Redis wall time, Redis memory, GPU utilization,
   DB parity, tracks, embeddings, model agreement, and evidence paths.
4. PostgreSQL correctness is unchanged.
5. Redis failures degrade gracefully and do not hide benchmark failures.
6. Rollback is one env flag or one commit revert, and rollback proof is recorded.

## Immediate Recommendation

Do not tune Redis server execution next. Cycle 16.B reduced client-side pipeline
overhead and left seconds-scale costs in embedding DB flush (`37.737 s`) and
Redis payload serialization (`31.242 s`), while Redis server command wall was
only `194.039 ms`. The next Redis-specific idea should target serialization or
only proceed if a later production benchmark proves Redis Streams reduce DB
polling/write overhead without changing PostgreSQL authority.
