# Cycle 16.B Redis Side-Effect Coalescing Results

**Last updated:** 2026-06-03

**Decision:** ACCEPTED by real production benchmark.

## Summary

Cycle 16.B reduced the embedding Redis side-effect wall by replacing per-row
Redis helper execution with one ordered Redis pipeline per embedding DB flush.
PostgreSQL remains the authoritative store; Redis remains non-authoritative
side-effect/cache state.

The candidate improved the targeted post-stage bottleneck and total throughput
without DB/model drift.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Benchmark JSON | `backend/logs/cycle16b-redis-coalescing-20260603T025823Z/redis_coalescing_metrics.json` | Candidate production metrics. |
| Benchmark Markdown | `backend/logs/cycle16b-redis-coalescing-20260603T025823Z/redis_coalescing_metrics.md` | Human-readable candidate comparison table. |
| Agreement JSON | `backend/logs/cycle16b-redis-coalescing-20260603T025823Z/model_agreement_cycle13b_vs_redis_coalescing.json` | Per-model baseline-agreement proof. |
| Wrapper log | `backend/logs/cycle16b-redis-coalescing-20260603T025823Z/wrapper.log` | Env, replay key, reset, and wrapper rc proof. |
| Baseline JSON | `backend/logs/cycle16b-redis-coalescing-20260603T025823Z/baseline_cycle13b_metrics.json` | Accepted Cycle 13.B baseline metrics used by the wrapper. |
| Measurement doc | `docs/cycle_13c_redis_db_side_effect_measurement_results.md` | Pre-implementation Redis command-cost measurement. |
| Investigation doc | `docs/cycle_16b_redis_side_effect_coalescing_investigation.md` | Cycle 16.B plan, risk, and acceptance criteria. |
| Benchmark history | `docs/production_inference_benchmark.md` | Canonical benchmark history destination. |

## Production Run

| Field | Value |
|---|---|
| Baseline replay | `cycle13-track-lookup-20260603T011324Z` |
| Baseline job | `c9f75d55-6043-4f27-bf9e-b2826d299459` |
| Candidate replay | `cycle16b-redis-coalescing-20260603T025823Z` |
| Candidate job | `b2dfa987-afc5-4b96-ab12-6799b149ac25` |
| Deployed SHA | `c458c44393f965b40ea98401399bf2efde2ea4b5` |
| Video | `grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4` |
| Status | `completed`, `4541/4541` frames |
| Candidate flag | `EMBEDDING_REDIS_SIDE_EFFECT_COALESCING=1` |
| Rollback flag | `EMBEDDING_REDIS_SIDE_EFFECT_COALESCING=0` and restart Celery workers |

## Before/After Metrics

| Metric | Cycle 13.B baseline | Cycle 16.B candidate | Delta |
|---|---:|---:|---:|
| DB-completed FPS | `5.205675` | `5.347791` | `+2.73 %` |
| DB-completed elapsed | `872.317 s` | `849.136 s` | `-2.66 %` |
| Step 2 frame wall | `458.696 s` | `460.698 s` | `+0.44 %` |
| Step 2 through pose upload | `680.422 s` | `682.475 s` | `+0.30 %` |
| GPU avg util | `11.003 %` | `11.030 %` | `+0.25 %` |
| GPU peak util | `50.000 %` | `52.000 %` | `+4.00 %` |
| Behavior RTT mean | `86.545 ms` | `86.532 ms` | `-0.02 %` |
| Behavior RTT p95 | `132.448 ms` | `132.870 ms` | `+0.32 %` |
| Embedding profile wall | `121.681 s` | `97.505 s` | `-19.87 %` |
| Embedding created span | `121.384 s` | `97.226 s` | `-19.90 %` |
| Embedding Redis flush wall | `59.874 s` | `35.970 s` | `-39.92 %` |
| Embedding DB flush wall | `38.773 s` | `37.737 s` | `-2.67 %` |
| Detection rows | `72744` | `72744` | `0.00 %` |
| BBox rows | `72744` | `72744` | `0.00 %` |
| Embedding rows | `72578` | `72578` | `0.00 %` |
| Student tracks | `53` | `53` | `0.00 %` |

## Redis-Specific Proof

| Metric | Cycle 13.C measured shape | Cycle 16.B candidate | Interpretation |
|---|---:|---:|---|
| Estimated helper commands | `870936` | `870936` | Command contract preserved. |
| Estimated pipeline executes | `72578` | `146` | Per-row pipeline execution removed. |
| Redis server command wall | `530.485 ms` | `194.039 ms` | Redis server remains non-dominant. |
| Payload serialization wall | `32410.536 ms` | `31242.000 ms` | Serialization is still material. |
| Redis coalesced pipeline wall | n/a | `4084.293 ms` | Actual batched pipeline execution is small. |
| Redis helper errors | `0` | `0` | Error gate passed. |
| Redis unavailable count | n/a | `0` | Availability gate passed. |

## Model Agreement

Agreement is an accuracy proxy against the accepted Cycle 13.B baseline, not
human-labeled ground truth.

| Model | Candidate F1@IoU0.5 | Count delta |
|---|---:|---:|
| `attention_tracking` | `100.000 %` | `0.00 %` |
| `hand_raising` | `100.000 %` | `0.00 %` |
| `person_detection` | `100.000 %` | `0.00 %` |
| `sitting_standing` | `100.000 %` | `0.00 %` |

## Decision

| Gate | Evidence | Result |
|---|---|---|
| Production benchmark | Replay `cycle16b-redis-coalescing-20260603T025823Z` completed on production Linux RTX 5090. | Passed |
| Target bottleneck | Redis flush wall `59.874 s -> 35.970 s`. | Passed |
| Pipeline reduction | Estimated pipeline executes `72578 -> 146` compared with Cycle 13.C measured shape. | Passed |
| Total throughput | DB FPS `5.205675 -> 5.347791`, elapsed `872.317 s -> 849.136 s`. | Passed |
| Inference protection | Step 2 and behavior RTT moved less than `0.5 %`. | Passed |
| Correctness | DB rows unchanged; all model agreement rows `100.000 %`. | Passed |
| Redis safety | Coalesced errors `0`, unavailable count `0`. | Passed |

**Cycle 16.B is ACCEPTED.**

## Next Bottleneck

Cycle 16.B left two post-stage costs visible:

| Remaining item | Candidate wall |
|---|---:|
| Embedding DB flush | `37.737 s` |
| Embedding Redis payload serialization | `31.242 s` |
| Step 2 through pose upload tail over frame wall | `221.777 s` |

The next sorted cycle should not tune Redis server execution. The Redis server
wall is `194.039 ms`, while serialization and DB flush are still seconds-scale.
