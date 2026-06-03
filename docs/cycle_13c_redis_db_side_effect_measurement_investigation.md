# Cycle 13.C / 16.A Redis DB Side-Effect Measurement Investigation

**Last updated:** 2026-06-03

**Status:** Phase A started. This cycle is measurement-only until a completed
production Linux RTX 5090 benchmark on `combined.mp4` proves Redis command
count, Redis wall, Redis memory, DB flush wall, and correctness impact. No
Redis semantic optimization is accepted, rejected, skipped, or closed by this
document.

## Problem Statement

Cycle 13.B removed the measured embedding track-lookup bottleneck and left two
large post-stage side-effect buckets:

| Bucket | Cycle 13.B wall | Current interpretation |
|---|---:|---|
| Redis flush | `59.874 s` | Potential hotspot, but command count and command wall are not yet decomposed. |
| DB flush | `38.773 s` | Potential PostgreSQL bulk-write target after Redis side effects are bounded. |
| Existing checks | `13.964 s` | Lower-priority idempotency query target. |

Cycle 7 is the cautionary precedent: Redis client caching was accepted, but the
real gain was much smaller than projected. Therefore the next sorted cycle must
measure Redis command cost directly before implementing Redis pipeline,
Streams, scripting, or wider Redis coordination changes.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Doc | `docs/cycle_13_embedding_track_lookup_results.md` | Latest accepted Cycle 13.B result and remaining bottleneck table. |
| Doc | `docs/redis_broader_optimization_opportunities.md` | Redis roadmap and Cycle 16.A command-cost requirement. |
| Doc | `docs/production_inference_benchmark.md` | Production benchmark authority and accepted metrics history. |
| Doc | `docs/crop_frame_optimization_execution.md` | Sorted execution map after Cycle 13.B. |
| File | `backend/apps/video_analysis/tasks.py` | Embedding loop, `pending_redis` flush path, DB flush, and profiling metadata. |
| File | `backend/apps/tracking/embeddings.py` | Redis embedding cache helpers and job-track pipeline helper. |
| File | `tools/prod/prod_collect_benchmark_metrics.py` | Benchmark collector that must surface any new Redis/DB counters. |
| File | `tools/prod/prod_watch_benchmark_metrics.sh` | Watcher that must display the new counters during production runs. |
| File | `backend/config/settings/base.py` | Source for any guarded profiling-only flag added by this cycle. |

## Root Cause Hypothesis

The current `redis_flush_ms` metric measures the wall around all Redis
side-effect writes, not the actual Redis command mix. It does not prove whether
the time is caused by Redis socket/command latency, Python loops, JSON/vector
serialization, helper overhead, or interaction with the DB flush cadence.

The next measurement must split Redis side-effect wall into:

| Measurement | Purpose |
|---|---|
| Command count by helper | Determine whether the flush is many small commands or already batched. |
| Pipeline execute count | Bound the opportunity for additional pipelining. |
| Redis command wall | Bound maximum possible speedup from Redis-only changes. |
| Python serialization wall | Avoid blaming Redis for local vector/JSON formatting. |
| Estimated bytes written | Detect oversized embedding payloads. |
| Redis memory before/after | Prevent hidden memory regressions. |
| Redis errors/timeouts | Fail closed if profiling exposes degraded Redis behavior. |
| DB flush wall in same run | Keep Redis and PostgreSQL side effects comparable. |

## Expected Gain

This cycle has no expected performance gain yet because it is measurement-only.
The expected output is an upper bound for later candidates:

| Candidate after measurement | Required proof before implementation |
|---|---|
| Redis pipeline coalescing | Redis command wall or pipeline count is material. |
| PostgreSQL bulk/COPY path | DB flush wall remains material after Redis command cost is bounded. |
| Redis Streams for watcher/progress | DB polling/write overhead is measured and material. |
| Redis scripts | A measured read-compute-write Redis pattern remains after pipelining. |

## Risk Assessment

| Risk | Mitigation |
|---|---|
| Profiling changes runtime behavior | Guard all instrumentation behind a disabled-by-default flag. |
| Profiling overhead distorts metrics | Record profiler wall and compare candidate against Cycle 13.B baseline. |
| Redis failure hidden by profiling | Record Redis errors and keep PostgreSQL as the authoritative store. |
| Evidence over-interpreted | Probes may stage candidates only; decisions require full production benchmark. |

## Rollback Strategy

Rollback must be a single environment flag reset and Celery restart:

```text
OFFLINE_REDIS_COMMAND_PROFILING=0
```

No database schema, model, Triton config, or Redis key contract change is
allowed in this measurement cycle.

## Acceptance Criteria

This cycle is complete only when:

| Gate | Required evidence |
|---|---|
| Production benchmark | Completed `combined.mp4` run on the Linux RTX 5090. |
| Baseline comparison | Compared against accepted Cycle 13.B replay `cycle13-track-lookup-20260603T011324Z`. |
| Redis counters | Command count, pipeline execute count, command wall, bytes estimate, errors, and memory are recorded. |
| DB counters | DB flush wall and row counts are recorded in the same evidence bundle. |
| Correctness | DB rows, tracks, embeddings, and model agreement match the accepted baseline. |
| Decision table | Results classify the next implementation candidate, not this measurement as an optimization. |
| Documentation | `AGENTS.md`, `docs/production_inference_benchmark.md`, and the roadmap docs are updated. |

## Phase A Decision

Start with guarded Redis/DB command-cost instrumentation. Do not implement
pipeline coalescing, Streams, server-side scripts, or PostgreSQL bulk changes
until this cycle's production benchmark proves which side-effect bucket is
actually controllable.
