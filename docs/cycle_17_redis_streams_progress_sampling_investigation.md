# Cycle 17 Redis Streams Progress Sampling Investigation

**Last updated:** 2026-06-04

**Status:** PHASE A ACTIVE / INVESTIGATION ONLY. No code, env flag, runtime
behavior, or benchmark result in this file is accepted, rejected, skipped, or
closed until a completed production Linux RTX 5090 `combined.mp4` benchmark
compares it against the latest accepted baseline.

**Streaming compatibility:** `stream-safe-with-config`. Redis Streams may be
used only as bounded, non-authoritative progress/evidence mirrors. PostgreSQL
remains the authoritative job state. Live profile use requires per-camera or
per-job capped streams (`MAXLEN`), bounded consumers, and no terminal-state
decision from Redis.

## Problem Statement

Cycle 15 sharding candidates improved throughput but failed identity/model
agreement. The sorted roadmap now moves to Cycle 17. This cycle investigates
whether Redis Streams can improve benchmark observability and reduce
non-authoritative DB polling/progress pressure while preserving PostgreSQL
authority.

Cycle 17 must not claim to optimize Triton inference wall by default. Redis does
not remove model execution, gRPC wait, crop preprocessing, or pose runtime. It
can be accepted only if the production benchmark proves one of these:

| Acceptable impact | Required proof |
|---|---|
| Lower DB polling/progress pressure | Before/after DB query/write counters, watcher sampling counters, and total-wall/post-stage comparison. |
| Better benchmark evidence quality | Live watcher obtains complete high-frequency samples from Redis Stream while DB remains authoritative. |
| Total-wall improvement | Completed `combined.mp4` benchmark shows elapsed/FPS gain with DB/model parity preserved. |

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Doc | `docs/redis_broader_optimization_opportunities.md` | Defines Cycle 17 and why Redis Streams are preferred over Pub/Sub for replayable progress events. |
| Doc | `docs/cycle_13c_redis_db_side_effect_measurement_results.md` | Production measurement proving Redis server execution was tiny while client-side Redis helper work mattered. |
| Doc | `docs/cycle_16b_redis_side_effect_coalescing_investigation.md` | Accepted Redis-side implementation pattern: guarded flag, PostgreSQL authority, production benchmark, rollback. |
| Doc | `docs/cycle_16b_redis_side_effect_coalescing_results.md` | Accepted Cycle 16.B production benchmark; establishes current Redis discipline and metric expectations. |
| Doc | `docs/production_inference_benchmark.md` | Production benchmark authority and required before/after tables. |
| File | `backend/config/settings/base.py` | Existing Redis, cache, and inference env settings are defined here. |
| File | `backend/apps/video_analysis/tasks.py` | Current job progress writes, Redis status helpers, and inference lifecycle live here. |
| File | `tools/prod/prod_watch_benchmark_metrics.sh` | Current live watcher polls benchmark/job metrics; candidate may add stream reads here. |
| File | `tools/prod/prod_collect_benchmark_metrics.py` | Evidence collector must surface stream counters if implementation proceeds. |
| Constitution | `.specify/memory/constitution.md` §12.5 / §12.6 | Real production benchmark is the only decision authority. |

## Current Evidence

| Evidence | Measurement | Interpretation |
|---|---:|---|
| Cycle 16.A Redis server command wall | `530.485 ms` | Redis server execution was not the main runtime bottleneck. |
| Cycle 16.A profiled Redis flush wall | `92.397 s` | Client-side helper/payload/pipeline work was material before 16.B. |
| Cycle 16.B Redis flush wall | `35.970 s` | Coalescing was accepted; remaining Redis server tuning is de-ranked. |
| Cycle 15.B1.C2 DB FPS | `7.833` | Sharding speedups exist but are blocked by correctness, so Cycle 17 must not assume sharding. |
| Cycle 15.B1.C2 model F1 range | `53.730 %` - `61.109 %` | Current sharding path is correctness-blocked; Redis Streams must not be used to hide correctness failures. |

## Hypothesis

The highest-value Cycle 17 candidate is measurement-first:

1. Mirror non-terminal job progress and benchmark samples into a capped Redis
   Stream.
2. Teach the live watcher to read the stream for high-frequency progress while
   continuing to query PostgreSQL for terminal state and final counters.
3. Record before/after DB polling/write counts and watcher sample completeness.

Expected impact is primarily evidence quality and possible DB load reduction.
Expected Step 2 inference-wall improvement is `0 %` unless the production
benchmark proves DB progress writes or watcher polling were blocking the job.

## Candidate Design

| Component | Candidate behavior | Guard |
|---|---|---|
| Stream key | `bench:{replay_key}:events` for benchmark runs. | Candidate flag required. |
| Event type | Lightweight JSON progress samples: frame counters, status, stage, FPS, GPU snapshot reference, and timestamp. | No model predictions or authoritative evidence rows. |
| Retention | `XADD ... MAXLEN ~ N` with bounded `N`. | Required for live safety. |
| Watcher | Read newest stream entries first, then DB fallback for authoritative status. | If Redis unavailable, watcher continues DB polling. |
| Collector | Capture stream counters: entries, read errors, bytes estimate, fallback count, DB polls avoided estimate. | Evidence only. |

## Risks

| Risk | Mitigation |
|---|---|
| Redis becomes a second authority | PostgreSQL remains terminal-state authority; stream data is explicitly non-authoritative. |
| Unbounded stream growth | Enforce `MAXLEN` and document live-profile bounds. |
| Missing Redis events mislead operators | Watcher always falls back to DB and reports Redis stream gaps/errors. |
| Extra writes increase total wall | Benchmark must include total wall, FPS, Redis command counters, and DB parity; reject if it regresses. |
| No meaningful performance gain | Accept only as observability improvement if evidence quality improves without throughput/correctness regression. |

## Rollback Strategy

Rollback must be a single env flag plus worker restart:

```text
BENCHMARK_REDIS_STREAM_EVENTS=0
```

No persisted PostgreSQL schema changes should be required for the first
candidate. Redis keys should expire or be capped, and PostgreSQL job state must
remain complete without Redis.

## Acceptance Criteria

Cycle 17 can be accepted only if all gates pass:

| Gate | Required evidence |
|---|---|
| Production benchmark | Completed Linux RTX 5090 `combined.mp4` benchmark with replay key, job ID, deployed SHA, metrics JSON/MD, model agreement, and GPU CSV. |
| Correctness | DB rows, embeddings, StudentTracks, and model agreement unchanged from the accepted baseline. |
| Authority | PostgreSQL terminal status remains correct when Redis stream data is ignored. |
| Bounded Redis | Stream `MAXLEN`, memory delta, write/read counters, and Redis errors recorded. |
| Impact | Either DB polling/progress pressure decreases, evidence completeness improves, or total wall/FPS improves. |
| Rollback | `BENCHMARK_REDIS_STREAM_EVENTS=0` plus worker restart restores the previous watcher/benchmark behavior. |

## Phase A Decision

Start Cycle 17 as an investigation and measurement-design cycle. Do not change
runtime behavior until the implementation plan records exact stream keys,
bounded retention, watcher fallback behavior, collector metrics, and the
production benchmark wrapper.
