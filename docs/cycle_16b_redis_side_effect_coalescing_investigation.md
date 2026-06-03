# Cycle 16.B Redis Side-Effect Coalescing Investigation

**Last updated:** 2026-06-03

**Status:** Phase A investigation started. No code is implemented by this
document. The next valid implementation must be guarded and must run a
completed production Linux RTX 5090 benchmark on `combined.mp4` before any
decision.

## Problem Statement

Cycle 13.C / 16.A proved the remaining Redis embedding side-effect wall is not
Redis server execution. The production profile recorded `92.397 s` Redis flush
wall in the profiled run, but Redis `INFO commandstats` showed only
`530.485 ms` of server command execution. The larger cost is client-side:
helper calls, payload serialization, and per-row pipeline execution.

The current embedding loop writes Redis side effects for every persisted
embedding row. In the Cycle 13.C benchmark that meant:

| Measured item | Value |
|---|---:|
| Redis side-effect rows | `72578` |
| Estimated helper commands | `870936` |
| Estimated pipeline executes | `72578` |
| Redis server command calls | `1017733` |
| Redis server command wall | `530.485 ms` |
| Payload serialization wall | `32410.536 ms` |
| Redis helper wall, summed | `59722.430 ms` |
| Redis payload bytes estimate | `4446643934` |

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Doc | `docs/cycle_13c_redis_db_side_effect_measurement_results.md` | Production measurement that justifies this cycle. |
| Doc | `docs/cycle_13c_redis_db_side_effect_measurement_investigation.md` | Original measurement plan and acceptance gates. |
| Doc | `docs/cycle_13_embedding_track_lookup_results.md` | Latest accepted baseline before Redis-side work. |
| Doc | `docs/production_inference_benchmark.md` | Production benchmark authority and comparison table destination. |
| File | `backend/apps/video_analysis/tasks.py` | `generate_embeddings`, `pending_redis`, `_flush_pending`, and `_cache_student_embedding_for_analysis`. |
| File | `backend/apps/tracking/embeddings.py` | `cache_embedding`, `cache_job_track_embedding`, and `redis_client` helper contracts. |
| File | `tools/prod/prod_collect_benchmark_metrics.py` | Metrics collector must surface coalescing counters. |
| File | `tools/prod/prod_watch_benchmark_metrics.sh` | Watcher must show coalescing counters during production benchmark. |
| Doc | `.specify/memory/constitution.md` | Benchmark decision authority and PostgreSQL authority gates. |

## Root Cause

The current Redis side-effect path is row-oriented:

| Current helper | Current behavior |
|---|---|
| `cache_embedding` | Serializes one vector and writes one latest key per row. |
| `cache_job_track_embedding` | Serializes one vector and executes one Redis pipeline per row. |
| `_cache_student_embedding_for_analysis` | Builds JSON and issues multiple Redis commands per row. |

This shape repeats work even when the embedding vector is reused from a
process-local track cache. Cycle 13.C recorded `generated_vector_count=53` and
`process_cache_hits=72525`, but Redis side effects still wrote `72578` rows.

## Candidate Architecture

Cycle 16.B should implement the lowest-risk coalescing path first:

| Candidate | Description | Risk |
|---|---|---|
| 16.B.1 batch pipeline writer | Replace per-row helper execution with one Redis pipeline per embedding flush batch while preserving the same final keys and list/set contracts. | Low-medium |
| 16.B.2 serialize-once payload builder | Serialize each vector/payload once per side-effect row and reuse that payload across Redis commands. | Low-medium |
| 16.B.3 final-state compaction | For latest keys and capped history keys, write only the final latest value and the last retained history entries per track. | Medium-high; only if Redis key parity proves final-state equivalence |

The first implementation should prefer 16.B.1 plus 16.B.2. 16.B.3 can be a
separate candidate only if the investigation proves no runtime consumer depends
on intermediate Redis history entries during the same embedding task.

## Required New Metrics

| Metric | Purpose |
|---|---|
| `redis_coalescing_enabled` | Confirms candidate flag state. |
| `redis_pipeline_execute_count_actual` | Proves pipeline executes fell from the current `72578` estimate. |
| `redis_command_count_actual_or_estimated` | Confirms command volume did not unexpectedly grow. |
| `redis_payload_bytes_actual_or_estimated` | Confirms payload volume did not grow. |
| `redis_payload_serialize_ms` | Measures serialization reduction. |
| `redis_flush_ms` | Primary post-stage wall target. |
| `redis_server_commandstats_ms_total` | Confirms Redis server remains non-dominant. |
| `redis_key_contract_parity` | Verifies final latest/history/tracks keys match baseline when a compact path is tested. |

## Acceptance Criteria

Cycle 16.B can be accepted only if a real production benchmark proves:

| Gate | Required evidence |
|---|---|
| Production benchmark | Completed `combined.mp4` run on the Linux RTX 5090 with replay key, job ID, deployed SHA, GPU CSV, and metrics JSON. |
| Redis wall | `redis_flush_ms` improves materially against the accepted Cycle 13.B baseline and the Cycle 13.C measurement. |
| Pipeline reduction | Pipeline executes decrease from the measured `72578` shape without Redis helper errors. |
| Total wall | DB-completed elapsed and FPS improve against the latest accepted baseline. |
| Inference protection | Step 2 wall, behavior RTT mean/p95, and GPU utilization do not materially regress. |
| Correctness | DB rows, embeddings, StudentTracks, model agreement, and Redis key contract parity pass. |
| Rollback | `EMBEDDING_REDIS_SIDE_EFFECT_COALESCING=0` restores the accepted path after worker restart. |

## Rollback Strategy

The implementation must be disabled by default and controlled by a single env
flag:

```text
EMBEDDING_REDIS_SIDE_EFFECT_COALESCING=0
```

Rollback is setting the flag to `0` and restarting Celery workers. PostgreSQL
remains authoritative; Redis side effects must never determine job terminal
state or benchmark acceptance.

## Phase A Decision

Start Cycle 16.B after Cycle 13.C because the production benchmark proved a
material client-side Redis side-effect wall and ruled out Redis server tuning as
the first fix. Do not implement Redis Streams, Redis scripts, or streaming
persistence overlap before this coalescing candidate unless a later production
benchmark decision table reorders the roadmap.
