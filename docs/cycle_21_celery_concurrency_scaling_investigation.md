# Cycle 21 Celery Concurrency Scaling Investigation

**Last updated:** 2026-06-04

**Status:** Phase A governance claimed by Agent 20. No production worker count,
thread count, pool type, queue concurrency, or GPU cap is changed by this
document. The only valid decision state is
`NO_DECISION_PRODUCTION_BENCHMARK_REQUIRED` until a completed production Linux
RTX 5090 benchmark on `combined.mp4` proves the candidate.

**Streaming compatibility:** `stream-safe-with-config`. Any future concurrency
change must be profile-specific: offline and live worker knobs are benchmarked
separately, live queues keep bounded per-camera behavior, and no live profile
inherits an offline worker or GPU-cap change by default.

## Question Answered

Increasing Celery workers can help only when the workload has independent work
for the extra workers to consume. The current accepted single-video offline
pipeline still has large monolithic sections:

| Stage | Current shape | Worker scaling implication |
|---|---|---|
| `process_video_upload` | One Celery task owns the offline video frame loop. | Extra `video_inference` workers are mostly idle for one job. |
| Triton model dispatch | In-process orchestration and Triton batching control request flow. | Extra Celery workers do not automatically increase one task's model parallelism. |
| Step 3 persistence | Runs after the frame aggregation inside the same task. | Extra workers help only after streaming/split persistence exists. |
| `generate_embeddings` | One Celery task loops through persisted rows and writes DB/Redis side effects. | Extra workers help only if embeddings are sharded or batched into independent tasks. |

Therefore a blind worker increase is not justified as a current default. It is
acceptable as a benchmarked cycle after a candidate creates parallel work or if
a controlled matrix proves the current assumptions wrong.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| File | `tools/prod/prod_start_celery_workers.sh` | Production worker launcher and active concurrency env knobs. |
| File | `tools/prod/prod_stop_celery_workers.sh` | Required rollback and duplicate-worker cleanup helper. |
| File | `tools/prod/prod_enable_parallel_flow.sh` | Current optimized production defaults for worker concurrency and GPU cap. |
| File | `backend/apps/video_analysis/tasks.py` | Single-job task boundaries for inference, persistence, and embedding. |
| File | `backend/config/celery.py` | Celery pool/concurrency config source. |
| Doc | `docs/cycle_13c_redis_db_side_effect_measurement_results.md` | Latest benchmark proving current post-stage bottleneck is inside one embedding task's Redis side effects. |
| Doc | `docs/cycle_16b_redis_side_effect_coalescing_investigation.md` | Next cycle that may reduce single-task Redis wall before worker scaling. |
| Doc | `docs/cycle_20_streaming_persistence_embedding_overlap_investigation.md` | Future architecture that could create parallel persistence/embedding work. |
| Doc | `.specify/memory/constitution.md` § 8.1.1 | Concurrency Scaling Authority gate. |
| Turn ledger | `docs/agent_20_remaining_lanes_turn.md` | Agent 20 governance ownership and non-overlap boundaries. |

## Existing Production Knobs

The production launcher reads these values from `backend/.env`:

| Env key | Current optimized default | Scope |
|---|---:|---|
| `CELERY_DEFAULT_WORKER_CONCURRENCY` | `2` | Default queue worker. |
| `CELERY_LIVE_WORKER_CONCURRENCY` | `2` | Live-mode control/person/pose/behavior workers. |
| `CELERY_OFFLINE_WORKER_CONCURRENCY` | `4` | Offline control/person/pose/behavior workers. |
| `CELERY_GPU_CONCURRENCY_CAP` | `0` | Optional cap applied to per-worker concurrency. |
| `TRITON_OFFLINE_BATCH_QUEUE_MAX_CONCURRENCY` | `2` | In-process offline frame-batch dispatch concurrency. |

## Current Metric-Based Assessment

Cycle 13.C / 16.A completed on production with replay
`cycle13c-redis-command-profile-20260603T020723Z`, job
`aa246a4e-e0f9-471a-9ce3-74f343bbd1fb`:

| Metric | Value |
|---|---:|
| DB-completed FPS | `5.024795` |
| Step 2 frame wall | `458.532 s` |
| Behavior RTT mean / p95 | `86.203 ms` / `131.898 ms` |
| GPU avg / peak util | `10.840 %` / `50.000 %` |
| Embedding wall | `152.771 s` |
| Redis flush wall | `92.397 s` |
| Estimated Redis pipeline executes | `72578` |
| Redis server command wall | `530.485 ms` |

These metrics point to client-side Redis side-effect work inside one embedding
task, not to insufficient Celery parent workers. Raising worker count before
Cycle 16.B would not reduce the `72578` per-row Redis pipeline shape and could
increase PostgreSQL/Redis contention.

## When More Workers Could Help

| Condition | Why scaling may help |
|---|---|
| Multiple independent video jobs are intentionally accepted in parallel. | Extra workers can consume multiple `process_video_upload` tasks. |
| Video sharding is implemented. | Segment workers can process independent frame ranges. |
| Streaming persistence is implemented. | Persistence workers can consume frame packets while inference continues. |
| Embedding is sharded by frame range or track partition. | Multiple embedding tasks can consume independent persisted rows. |
| Render is split into a dedicated queue. | Render wall can overlap with other follow-up work. |

## When More Workers Probably Will Not Help

| Condition | Reason |
|---|---|
| One monolithic `process_video_upload` task processes one video. | Extra workers have no same-job task to consume. |
| One monolithic `generate_embeddings` task writes all embeddings. | Extra workers cannot split that Python loop. |
| Triton is already receiving the same in-process batch stream. | Extra Celery workers do not change request formation. |
| PostgreSQL or Redis is the measured limiter. | Extra workers can increase contention and regress wall time. |

## Candidate Benchmark Matrix

Cycle 21 should run only after Cycle 16.B or after a parallelizing architecture
candidate creates work for extra workers. The first matrix should keep all
other accepted flags fixed:

| Candidate | Env delta | Expected outcome |
|---|---|---|
| Baseline | `CELERY_OFFLINE_WORKER_CONCURRENCY=4`, `CELERY_GPU_CONCURRENCY_CAP=0`, `TRITON_OFFLINE_BATCH_QUEUE_MAX_CONCURRENCY=2` | Accepted optimized profile. |
| Worker-only 6 | `CELERY_OFFLINE_WORKER_CONCURRENCY=6` | Tests whether hidden queue parallelism exists; expected low gain for one job. |
| Worker-only 8 | `CELERY_OFFLINE_WORKER_CONCURRENCY=8` | Stress test for CPU/RSS/DB/Redis contention. |
| GPU-capped worker 6 | `CELERY_OFFLINE_WORKER_CONCURRENCY=6`, `CELERY_GPU_CONCURRENCY_CAP=4` | Tests extra non-GPU workers without increasing GPU-bound concurrency. |
| In-process dispatch 3 | `TRITON_OFFLINE_BATCH_QUEUE_MAX_CONCURRENCY=3` | Tests if accepted `2` is still the saturation knee after later cycles. |

Each candidate must stop workers, start exactly one governed worker set, verify
no duplicate consumers, run `combined.mp4`, collect evidence, then roll back to
the accepted profile.

## Agent 20 Governance Tasks

Agent 20 owns governance documentation only. These tasks do not authorize
production topology changes:

| Task | State | Required output |
|---|---|---|
| `A20-C21-01` | `COMPLETED` | Worker topology matrix fields for parent workers, queues, pool type, prefetch, and GPU cap. |
| `A20-C21-02` | `COMPLETED` | Duplicate-worker proof checklist before every candidate run. |
| `A20-C21-03` | `COMPLETED` | Resource-budget checklist for CPU, RSS, VRAM, PostgreSQL, Redis, and queue backlog. |
| `A20-C21-04` | `COMPLETED` | Rollback proof format for env reset plus clean worker stop/start. |

The matrix remains future-only until a separate benchmark-lock owner records the
lock and runs a full production comparison.

## Governance Packet V0

This packet defines the evidence a future Cycle 21 benchmark owner must collect
before changing worker topology.

### Required topology capture

| Field | Source | Required value shape |
|---|---|---|
| Runtime profile | `TRITON_EXECUTION_MODE` and profile script output | `offline` or `live`; never mixed. |
| Worker names | `prod_start_celery_workers.sh` startup log | One row per launched worker name. |
| Queue bindings | Celery `-Q` arguments and `backend/config/celery.py` routes | Queue name per worker. |
| Parent PIDs | PID files plus `ps` output | Exactly one parent per intended worker name. |
| Child process count | `ps` grouped by worker nodename | Matches pool/concurrency expectation. |
| Pool type | `CELERY_WORKER_POOL` / Celery config | `prefork` on Linux unless explicitly benchmarked otherwise. |
| Concurrency | launcher-resolved concurrency | Candidate and baseline numeric values. |
| Prefetch | `--prefetch-multiplier` and Celery config | Must be captured; current launcher uses `1`. |
| Task limits | soft/hard time limits and `max_tasks_per_child` | Same between baseline and candidate unless matrix declares a delta. |
| GPU cap | `CELERY_GPU_CONCURRENCY_CAP` | Captured even when `0`. |
| In-process dispatch | `TRITON_OFFLINE_BATCH_QUEUE_MAX_CONCURRENCY` | Captured with worker topology. |

### Duplicate-worker proof checklist

Before and after every candidate run:

| Check | Pass condition |
|---|---|
| Stop workers first | `prod_stop_celery_workers.sh` ran before a new topology starts. |
| Parent count | Exactly one parent per intended worker name. |
| Queue duplication | No queue has unintended duplicate consumers. |
| Stale PID files | PID files match live processes. |
| Beat state | Beat is either intentionally running once or explicitly disabled. |
| Active task drain | No stale active task from a prior benchmark remains. |

A candidate with duplicate workers is invalid even if FPS improves.

### Resource-budget packet

| Budget item | Required evidence | Failure meaning |
|---|---|---|
| CPU | per-worker CPU summary during run | Oversubscription or idle extra workers. |
| RSS | per-worker RSS and peak memory | Memory contention or leak risk. |
| VRAM | GPU memory used/free samples | GPU-cap regression risk. |
| GPU util | avg/peak utilization from GPU CSV | Detects idle or saturated GPU. |
| PostgreSQL | connection count and query/write counters if available | DB contention invalidates a scaling claim. |
| Redis | command wall, errors, and queue side-effect counters | Redis contention invalidates a scaling claim. |
| Celery backlog | queue depth and wait samples | Proves whether extra workers consumed real work. |
| Behavior RTT | mean/p95/p99 per-model RTT | Triton contention blocks acceptance. |
| Correctness | DB row counts, tracks, embeddings, model agreement | Correctness regression blocks acceptance. |

Unavailable metrics must be marked `unavailable` with a reason in the decision
table. They must not be represented as measured zero.

### Benchmark and rollback format

Every candidate row must record:

| Field | Requirement |
|---|---|
| Baseline replay | Latest accepted non-sharded production replay key and job ID. |
| Candidate replay | Candidate replay key, job ID, deployed SHA, and env delta. |
| Topology delta | Only the worker/concurrency fields being tested. |
| Cleanup | Worker stop/start proof before and after the run. |
| Decision table | Constitution §12.6 table in `docs/production_inference_benchmark.md`. |
| Rollback proof | Env reset plus clean worker restart to accepted topology. |

## Acceptance Criteria

| Gate | Required evidence |
|---|---|
| Production benchmark | Completed `combined.mp4` run on native Linux RTX 5090. |
| Topology proof | Parent workers, child concurrency, queues, pool type, prefetch, GPU cap, and duplicate-worker check captured. |
| Throughput | FPS or total wall improves materially against the latest accepted baseline. |
| Inference protection | Step 2 wall and behavior RTT mean/p95 do not materially regress. |
| Resource safety | Worker RSS, CPU, VRAM, PostgreSQL connections, Redis wall, and queue backlog remain within budget. |
| Correctness | DB rows, StudentTracks, embeddings, and model-agreement F1 pass. |
| Rollback | Env reset plus clean worker stop/start restores the accepted worker topology. |

## Rollback

```text
CELERY_DEFAULT_WORKER_CONCURRENCY=2
CELERY_LIVE_WORKER_CONCURRENCY=2
CELERY_OFFLINE_WORKER_CONCURRENCY=4
CELERY_GPU_CONCURRENCY_CAP=0
TRITON_OFFLINE_BATCH_QUEUE_MAX_CONCURRENCY=2
```

Then run:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_stop_celery_workers.sh
bash tools/prod/prod_start_celery_workers.sh
```

## Phase A Decision

Do not change production worker counts now. Start Cycle 16.B first because the
latest production metrics identify client-side Redis side-effect coalescing as
the next concrete bottleneck. Keep Cycle 21 staged as the governed concurrency
matrix to run after Cycle 16.B or after Cycle 20/14b creates parallel work.
