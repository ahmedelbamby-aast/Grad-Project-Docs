# Cycle 17 Redis Streams Progress Sampling Investigation

**Last updated:** 2026-06-04

**Status:** ACCEPTED AS OBSERVABILITY-ONLY / NOT A THROUGHPUT OPTIMIZATION.
The completed production Linux RTX 5090 `combined.mp4` benchmark proved bounded
Redis Stream progress evidence with exact correctness parity and no material
throughput gain.

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

## Phase B Implementation

Implemented files:

| File | Change |
|---|---|
| `backend/config/settings/base.py` | Adds default-off `BENCHMARK_REDIS_STREAM_EVENTS`, bounded `BENCHMARK_REDIS_STREAM_MAXLEN`, and bounded `BENCHMARK_REDIS_STREAM_TTL_SECONDS`. |
| `backend/apps/video_analysis/tasks.py` | Adds capped Redis Stream progress mirror helper, process-local write/fallback counters, heartbeat/status hooks, terminal metadata/audit summary under `benchmark_redis_stream`. |
| `tools/prod/prod_collect_benchmark_metrics.py` | Adds `redis_stream` evidence block and comparison-table metrics. |
| `tools/prod/prod_watch_benchmark_metrics.sh` | Adds optional `--redis-stream` table with stream length, TTL, memory, latest event, metadata counters, and read errors. |
| `tools/prod/prod_run_cycle17_redis_streams_benchmark.sh` | Adds reproducible production wrapper that enables the stream flag for one run, captures watcher log, collects metrics, runs model agreement, and resets the flag. |
| `backend/tests/unit/video_analysis/test_redis_progress_stream.py` | Covers disabled no-op, Redis-unavailable fallback counter, and capped replay-key stream writes. |
| `backend/tests/unit/pipeline/test_prod_collect_benchmark_metrics.py` | Covers collector parsing for Redis Stream health fields. |

Current candidate flags:

| Flag | Default | Candidate wrapper value | Authority |
|---|---:|---:|---|
| `BENCHMARK_REDIS_STREAM_EVENTS` | `0` | `1` for one benchmark run | PostgreSQL remains terminal authority. |
| `BENCHMARK_REDIS_STREAM_MAXLEN` | `1000` | configurable | Bounds stream growth. |
| `BENCHMARK_REDIS_STREAM_TTL_SECONDS` | `86400` | configurable | Ensures keys expire. |
| `WATCH_REDIS_STREAM_EVENTS` | `0` | `1` in wrapper watcher | Read-only display only. |
| `WATCH_DB_FULL_POLL_EVERY_N` | `1` | configurable | Evidence knob; current watcher still treats DB as authoritative. |

Implementation state: `ACCEPTED_OBSERVABILITY_ONLY_NOT_THROUGHPUT`. The
production benchmark and decision table are recorded in
`docs/production_inference_benchmark.md` § 40.

Benchmark release:

```text
BENCHMARK_RELEASE
agent: Agent 18
cycle: Cycle 17 Redis Streams progress sampling
replay_key: cycle17-redis-streams-20260604T025328Z
job_id: a7cf6fc2-23fb-4e17-beac-42343ba8d634
status: completed
metrics_json: /home/bamby/grad_project/backend/logs/cycle17-redis-streams-20260604T025328Z/redis_streams_metrics.json
metrics_md: /home/bamby/grad_project/backend/logs/cycle17-redis-streams-20260604T025328Z/redis_streams_metrics.md
model_agreement_json: /home/bamby/grad_project/backend/logs/cycle17-redis-streams-20260604T025328Z/model_agreement_baseline_vs_redis_streams.json
model_agreement_md: /home/bamby/grad_project/backend/logs/cycle17-redis-streams-20260604T025328Z/model_agreement_baseline_vs_redis_streams.md
rollback_verified: yes; BENCHMARK_REDIS_STREAM_EVENTS=0, workers restarted, Django setting False, Triton ready
released_at_utc: 2026-06-04T03:13:00Z
```

Local validation:

| Validation | Result |
|---|---|
| `pytest -q backend/tests/unit/video_analysis/test_redis_progress_stream.py backend/tests/unit/pipeline/test_prod_collect_benchmark_metrics.py` | `9 passed` |
| `py_compile backend/apps/video_analysis/tasks.py backend/config/settings/base.py tools/prod/prod_collect_benchmark_metrics.py` | Passed |
| `bash -n tools/prod/prod_watch_benchmark_metrics.sh tools/prod/prod_run_cycle17_redis_streams_benchmark.sh tools/prod/prod_enable_parallel_flow.sh` | Passed |
| `python scripts/ci/verify_doc_dates_and_reading_order.py` | Passed |
| `python scripts/ci/verify_mermaid_diagrams.py --paths docs/four_agent_cycle_coordination_board.md docs/cycle_17_redis_streams_progress_sampling_investigation.md` | Passed |
| `git diff --check` | Passed |
| `bash tools/prod/prod_run_cycle17_redis_streams_benchmark.sh --dry-run --tag cycle17-dryrun-agent18 --baseline-replay-key cycle15b-pre-shard-baseline-20260603T193531Z --watch-db-full-poll-every-n 2` | Passed; final status `DRY_RUN_NO_BENCHMARK_RECORDED` |

Local validation alone proved only fallback/collector behavior. The production
benchmark above is the authority for the observability-only acceptance.

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

Cycle 17 acceptance gate result:

| Gate | Result |
|---|---|
| Production benchmark | Passed: replay `cycle17-redis-streams-20260604T025328Z`, job `a7cf6fc2-23fb-4e17-beac-42343ba8d634`, status `completed`, `4541/4541` frames. |
| Correctness | Passed: DB rows, embeddings, StudentTracks, and all model-agreement rows matched the baseline exactly. |
| Authority | Passed: PostgreSQL terminal status remained authoritative; Redis events were evidence-only. |
| Bounded Redis | Passed: `XLen=1002` with approximate `MAXLEN 1000`, TTL `86396 s`, memory `252242 bytes`, `4729/4729` writes, zero Redis unavailable/write/read errors. |
| Impact | Passed for evidence completeness only: Redis captured high-frequency progress and final completion evidence. Throughput was neutral (`DB FPS -0.15 %`) and must not be claimed as improved. |
| Rollback | Passed: wrapper reset `BENCHMARK_REDIS_STREAM_EVENTS=0`, workers were restarted, Django resolved the setting as `False`, and Triton was ready. |

## Phase A Decision

Start Cycle 17 as an investigation and measurement-design cycle. Do not change
runtime behavior until the implementation plan records exact stream keys,
bounded retention, watcher fallback behavior, collector metrics, and the
production benchmark wrapper.

## Phase B Decision

Phase B is **accepted as observability-only**. Keep
`BENCHMARK_REDIS_STREAM_EVENTS` default-off and enable it only for governed
benchmark/watcher evidence runs. Do not describe Cycle 17 as a throughput
optimization: DB-completed FPS moved `5.620 -> 5.611` (`-0.15 %`), while Step 2
frame wall moved `467.450 s -> 461.087 s` (`-1.36 %`). The accepted value is
bounded replayable progress evidence: stream key
`bench:cycle17-redis-streams-20260604T025328Z:events`, `4729` successful writes,
TTL/memory/length recorded, final completed event captured, and zero Redis
errors.
