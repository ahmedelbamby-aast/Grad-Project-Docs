# Cycle 13.C / 16.A Redis DB Side-Effect Measurement Results

**Last updated:** 2026-06-03

**Status:** Measurement complete / `HYPOTHESIS_ONLY`. This cycle implemented
profiling, not an optimization. It does not accept, reject, skip, close, or
neglect any Redis implementation candidate.

## Production Benchmark Authority

| Field | Value |
|---|---|
| Baseline replay | `cycle13-track-lookup-20260603T011324Z` |
| Baseline job | `c9f75d55-6043-4f27-bf9e-b2826d299459` |
| Candidate replay | `cycle13c-redis-command-profile-20260603T020723Z` |
| Candidate job | `aa246a4e-e0f9-471a-9ce3-74f343bbd1fb` |
| Candidate deployed SHA | `bea98cb9a0bce6875975712d1ed967569f6b8b05` |
| Video | `/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4` |
| Frames | `4541/4541` |
| Candidate status | `completed` |
| Metrics evidence | `backend/logs/cycle13c-redis-command-profile-20260603T020723Z/redis_command_profile_metrics.json` |
| Agreement evidence | `backend/logs/cycle13c-redis-command-profile-20260603T020723Z/model_agreement_cycle13b_vs_redis_command_profile.json` |
| GPU CSV | `backend/logs/gpu_monitor_bench_20260603T050745.csv` |
| Benchmark summary | `backend/logs/bench_summary_20260603T050745.json` |

## Before / After Metrics

| Metric | Cycle 13.B baseline | Cycle 13.C profiled run | Delta |
|---|---:|---:|---:|
| DB-completed FPS | `5.205675` | `5.024795` | `-3.47 %` |
| DB-completed elapsed | `872.317 s` | `903.718 s` | `+3.60 %` |
| Step 2 frame wall | `458.696 s` | `458.532 s` | `-0.04 %` |
| Step 2 through pose upload | `680.422 s` | `684.303 s` | `+0.57 %` |
| GPU avg util | `11.003 %` | `10.840 %` | `-1.48 %` |
| GPU peak util | `50.000 %` | `50.000 %` | `0.00 %` |
| Behavior RTT mean | `86.545 ms` | `86.203 ms` | `-0.40 %` |
| Behavior RTT p95 | `132.448 ms` | `131.898 ms` | `-0.42 %` |
| Embedding wall | `121.681 s` | `152.771 s` | `+25.55 %` |
| Redis flush wall | `59.874 s` | `92.397 s` | `+54.32 %` |
| DB flush wall | `38.773 s` | `37.348 s` | `-3.68 %` |
| Existing-embedding check | `13.964 s` | `13.600 s` | `-2.60 %` |
| Track lookup | `0.447 s` | `0.448 s` | `+0.28 %` |
| Detection rows | `72744` | `72744` | `0.00 %` |
| BBox rows | `72744` | `72744` | `0.00 %` |
| Embedding rows | `72578` | `72578` | `0.00 %` |
| Student tracks | `53` | `53` | `0.00 %` |

## Redis Decomposition

| Redis profile field | Measured value | Interpretation |
|---|---:|---|
| Estimated helper commands | `870936` | High command volume from per-row side effects. |
| Estimated pipeline executes | `72578` | One pipeline execute per side-effect row is the main round-trip shape to remove. |
| Redis server command calls | `1017733` | Confirms high command count at Redis. |
| Redis server command wall | `530.485 ms` | Redis server execution is only about `0.57 %` of the profiled `redis_flush_ms`. |
| Payload bytes estimate | `4446643934` | About `4.45 GB` of serialized embedding payload is built/sent. |
| Payload serialization wall | `32410.536 ms` | Serialization is material and client-side. |
| `cache_embedding` helper wall | `15614.956 ms` | Per-row latest-vector helper overhead. |
| `cache_job_track_embedding` helper wall | `11534.170 ms` | Per-row job-track pipeline helper overhead. |
| Analysis cache helper wall | `32572.304 ms` | Largest helper bucket; writes JSON payload/history per row. |
| Redis helper errors | `0` | No Redis correctness failure observed. |
| Redis memory delta | `10537392 bytes` | About `10.05 MiB` net increase during the profiled run. |
| Profiling overhead | `2.018 ms` | The measurement wrapper overhead itself is negligible. |

## Correctness

Model agreement against the accepted Cycle 13.B baseline was exact for persisted
models:

| Model | F1@IoU0.5 | Precision | Recall | Boxes |
|---|---:|---:|---:|---:|
| `attention_tracking` | `100.000 %` | `100.000 %` | `100.000 %` | `11770 -> 11770` |
| `hand_raising` | `100.000 %` | `100.000 %` | `100.000 %` | `8799 -> 8799` |
| `person_detection` | `100.000 %` | `100.000 %` | `100.000 %` | `19162 -> 19162` |
| `sitting_standing` | `100.000 %` | `100.000 %` | `100.000 %` | `33013 -> 33013` |

## Decision Explanation

| Required field | Result |
|---|---|
| Decision status | `MEASUREMENT COMPLETE / HYPOTHESIS_ONLY` |
| Why not accepted | The candidate was profiling only and intentionally added measurement surfaces; it was not an optimization. |
| Why not rejected/skipped | The production benchmark completed and produced the needed decomposition; rejection or skipping of implementation candidates would violate the benchmark decision gate because no implementation candidate ran. |
| Causal interpretation | Redis server command execution is not the limiter (`0.530 s`). The material buckets are client-side helper loops, payload serialization, and `72,578` estimated pipeline executions. |
| Remaining bottleneck | Redis side-effect write path inside embedding generation, followed by PostgreSQL DB flush. |
| Next candidate | Cycle 16.B Redis side-effect coalescing: reduce helper calls, payload serialization, and pipeline executes while preserving final Redis key contracts and PostgreSQL authority. |

## Next Action

Start Cycle 16.B with an investigation document before code. The candidate
should not tune Redis server settings first; the measured server command wall is
too small. The higher-leverage target is the client-side side-effect writer:
batch pipeline execution by flush batch, serialize vectors once, and consider a
final-state-preserving compaction for Redis latest/history keys only if Redis
key contract parity can be proven.
