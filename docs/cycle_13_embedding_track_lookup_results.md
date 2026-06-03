# Cycle 13.B Embedding Track Lookup Results

**Last updated:** 2026-06-03

**Status:** ACCEPTED. Production benchmark
`cycle13-track-lookup-20260603T011324Z` completed on the Linux RTX 5090 using
`combined.mp4`, preserved DB/model parity, and reduced the measured embedding
track-lookup wall from `66.223 s` to `0.447 s`.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Baseline replay | `cycle12-single-inflight-overlap-20260602T225821Z` | Accepted Cycle 12.C production baseline. |
| Baseline job | `069a217f-fa43-48cc-bf18-c946d53bb3ee` | Accepted Cycle 12.C job. |
| Profiling replay | `cycle13-embedding-profile-20260603T003853Z` | Measurement-only run that selected track lookup as the first Cycle 13.B target. |
| Candidate replay | `cycle13-track-lookup-20260603T011324Z` | Cycle 13.B production benchmark. |
| Candidate job | `c9f75d55-6043-4f27-bf9e-b2826d299459` | Completed `4541/4541` frames. |
| Metrics JSON | `backend/logs/cycle13-track-lookup-20260603T011324Z/track_lookup_metrics.json` | Main production metrics and embedding profile. |
| Metrics Markdown | `backend/logs/cycle13-track-lookup-20260603T011324Z/track_lookup_metrics.md` | Human-readable evidence bundle. |
| Agreement JSON | `backend/logs/cycle13-track-lookup-20260603T011324Z/model_agreement_cycle12c_vs_track_lookup.json` | Correctness comparison against Cycle 12.C. |
| Benchmark summary | `backend/logs/bench_summary_20260603T041346.json` | Runtime benchmark summary. |
| GPU CSV | `backend/logs/gpu_monitor_bench_20260603T041346.csv` | GPU utilization evidence. |
| Wrapper | `tools/prod/prod_run_cycle13_track_lookup_benchmark.sh` | Reproducible production benchmark command. |

## Comparison

| Metric | Cycle 12.C baseline | Cycle 13.A profile | Cycle 13.B candidate | Delta vs Cycle 12.C | Delta vs Cycle 13.A |
|---|---:|---:|---:|---:|---:|
| DB-completed elapsed | `935.516 s` | `945.570 s` | `872.317 s` | `-6.756 %` | `-7.747 %` |
| DB-completed FPS | `4.854005` | `4.802395` | `5.205675` | `+7.245 %` | `+8.397 %` |
| Step 2 frame wall | `459.461 s` | `463.905 s` | `458.696 s` | `-0.167 %` | `-1.123 %` |
| Step 2 through pose upload | `680.619 s` | `688.003 s` | `680.422 s` | `-0.029 %` | `-1.102 %` |
| Run-complete wall | `746.193 s` | `755.494 s` | `749.186 s` | `+0.401 %` | `-0.835 %` |
| Behavior RTT mean | `83.936 ms` | `85.094 ms` | `86.545 ms` | `+3.108 %` | `+1.705 %` |
| Behavior RTT p95 | `130.200 ms` | `131.637 ms` | `132.448 ms` | `+1.727 %` | `+0.616 %` |
| GPU avg util | `10.332 %` | `9.709 %` | `11.003 %` | `+6.494 %` | `+13.328 %` |
| GPU peak util | `53.000 %` | `51.000 %` | `50.000 %` | `-5.660 %` | `-1.961 %` |
| Embedding created span | `187.139 s` | `187.884 s` | `121.384 s` | `-35.137 %` | `-35.394 %` |
| Embedding profile wall | n/a | `188.620 s` | `121.681 s` | n/a | `-35.489 %` |
| Track lookup wall | n/a | `66.223 s` | `0.447 s` | n/a | `-99.325 %` |
| Redis flush wall | n/a | `59.304 s` | `59.874 s` | n/a | `+0.961 %` |
| DB flush wall | n/a | `38.467 s` | `38.773 s` | n/a | `+0.794 %` |
| Existing-check wall | n/a | `14.527 s` | `13.964 s` | n/a | `-3.877 %` |
| Detection rows | `72,744` | `72,744` | `72,744` | `0.000 %` | `0.000 %` |
| BBox rows | `72,744` | `72,744` | `72,744` | `0.000 %` | `0.000 %` |
| Embedding rows | `72,578` | `72,578` | `72,578` | `0.000 %` | `0.000 %` |
| Student tracks | `53` | `53` | `53` | `0.000 %` | `0.000 %` |

Cycle 13.B profile proof:

| Metric | Value |
|---|---:|
| `prefetch_track_lookup_enabled` | `true` |
| `track_lookup_prefetch_hits` | `144,785` |
| `track_lookup_prefetch_misses` | `0` |
| `track_lookup_orm_fallbacks` | `0` |

Model agreement against Cycle 12.C was exact:

| Model | F1@IoU0.5 | Precision | Recall | Count delta |
|---|---:|---:|---:|---:|
| `attention_tracking` | `100.000 %` | `100.000 %` | `100.000 %` | `0.000 %` |
| `hand_raising` | `100.000 %` | `100.000 %` | `100.000 %` | `0.000 %` |
| `person_detection` | `100.000 %` | `100.000 %` | `100.000 %` | `0.000 %` |
| `sitting_standing` | `100.000 %` | `100.000 %` | `100.000 %` | `0.000 %` |

## Decision

| Gate | Evidence | Result |
|---|---|---|
| Production benchmark authority | Replay `cycle13-track-lookup-20260603T011324Z` completed on the Linux RTX 5090. | Pass |
| Targeted bottleneck improved | Track lookup `66.223 s -> 0.447 s` (`-99.325 %`). | Pass |
| Total throughput improved | DB FPS `4.854005 -> 5.205675` (`+7.245 %`); elapsed `935.516 s -> 872.317 s` (`-6.756 %`). | Pass |
| Embedding wall improved | Profile wall `188.620 s -> 121.681 s` (`-35.489 %`). | Pass |
| Correctness preserved | DB rows unchanged; all model F1 rows `100.000 %`. | Pass |
| RTT did not materially regress | Behavior RTT mean `+3.108 %`, p95 `+1.727 %`; this is below the prior rejection-level RTT regressions. | Pass |
| Rollback proof exists | Wrapper restored `EMBEDDING_STAGE_PROFILING=0` and `EMBEDDING_PREFETCH_TRACK_LOOKUP=0`; production accepted profile is re-enabled separately. | Pass |

**Decision:** ACCEPTED. `EMBEDDING_PREFETCH_TRACK_LOOKUP=1` is now part of
the optimized production profile. Rollback is setting
`EMBEDDING_PREFETCH_TRACK_LOOKUP=0` and restarting Celery workers.

## Next Bottleneck

After Cycle 13.B, the embedding-stage profile is dominated by Redis and DB
side effects:

| Remaining bucket | Cycle 13.B wall | Next action |
|---|---:|---|
| Redis flush | `59.874 s` | Stage Cycle 13.C Redis side-effect coalescing or defer to Cycle 16.A if command-level proof is required first. |
| DB flush | `38.773 s` | Keep PostgreSQL bulk/COPY candidate after Redis command-cost evidence. |
| Existing checks | `13.964 s` | Lower-priority idempotency query optimization. |

Because Cycle 7 overestimated Redis savings, the next Redis-related step must
measure command count and command wall before changing Redis semantics.

