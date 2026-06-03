# Cycle 14.C3 Batch-32 Parallel Provider Chunks Results

**Last updated:** 2026-06-03

**Decision:** NOT ACCEPTED. Batch `32` remains rejected, and accepted
production stays on `POSE_CROSS_FRAME_BATCH_SIZE=16` with
`POSE_PROVIDER_CHUNK_PARALLELISM=1`.

## Summary

Cycle 14.C3 tested the only batch-32-specific repair suggested by Cycle 14.C2:
keep the outer cross-frame RTMPose batch at `32`, but allow the two internal
16-crop RTMPose provider chunks to run concurrently with
`POSE_PROVIDER_CHUNK_PARALLELISM=2`.

The production benchmark completed and preserved DB/model correctness exactly,
but it did not repair batch 32. It regressed DB FPS, Step 2 through-pose wall,
RTMPose mean/p95 RTT, provider async wall, and average GPU utilization versus
the prior rejected batch-32 run.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Investigation | `docs/cycle_14c3_batch32_parallel_chunks_investigation.md` | Defines the C3 hypothesis, rollback, and acceptance gates. |
| Candidate metrics | `backend/logs/cycle14c3-batch32-parallel2-20260603T170117Z/cross_frame_batch_metrics.json` | Production metrics for the C3 candidate. |
| Candidate agreement vs batch 16 | `backend/logs/cycle14c3-batch32-parallel2-20260603T170117Z/model_agreement_cycle16b_vs_cross_frame_batch.json` | Correctness evidence against accepted batch 16. |
| Candidate comparison vs prior batch 32 | `backend/logs/cycle14c3-batch32-parallel2-20260603T170117Z/parallel2_vs_prior_batch32_metrics.json` | Metrics comparison against rejected Cycle 14.C2 batch 32. |
| Candidate agreement vs prior batch 32 | `backend/logs/cycle14c3-batch32-parallel2-20260603T170117Z/model_agreement_prior_batch32_vs_parallel2.json` | Correctness evidence against prior rejected batch 32. |
| Wrapper log | `backend/logs/cycle14c3-batch32-parallel2-20260603T170117Z/wrapper.log` | Proves active env, benchmark launch, collection, and rollback. |
| Wrapper | `tools/prod/prod_run_cycle14c3_batch32_parallel_chunks_benchmark.sh` | Reproducible production runner. |
| Code | `backend/apps/pipeline/services/pose_runtime.py` | Guarded provider-chunk parallelism implementation. |

## Production Run

| Field | Value |
|---|---|
| Replay key | `cycle14c3-batch32-parallel2-20260603T170117Z` |
| Job ID | `d71802b2-4c20-4f31-9c06-9854ebbc4eed` |
| Deployed SHA | `6642046` |
| Video | `/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4` |
| Status | `completed`, `4541/4541` |
| Candidate env | `POSE_CROSS_FRAME_BATCH_SIZE=32`, `POSE_PROVIDER_CHUNK_PARALLELISM=2` |
| Rollback proof | `POSE_CROSS_FRAME_BATCH_SIZE=16`, `POSE_PROVIDER_CHUNK_PARALLELISM=1`, profiling flags `0` |

## Comparison

| Metric | Batch 16 accepted | Prior batch 32 | C3 parallel2 | Delta vs 16 | Delta vs prior 32 |
|---|---:|---:|---:|---:|---:|
| DB FPS | `5.680` | `5.625` | `5.465` | `-3.79 %` | `-2.84 %` |
| DB elapsed s | `799.428` | `807.326` | `830.947` | `+3.94 %` | `+2.93 %` |
| Step2 frame wall s | `462.188` | `459.750` | `460.775` | `-0.31 %` | `+0.22 %` |
| Step2 through pose s | `633.939` | `643.184` | `665.406` | `+4.96 %` | `+3.45 %` |
| GPU avg % | `12.168` | `14.288` | `12.092` | `-0.62 %` | `-15.37 %` |
| GPU peak % | `51.000` | `86.000` | `54.000` | `+5.88 %` | `-37.21 %` |
| Peak VRAM MiB | `15731.000` | `15725.000` | `15731.000` | `+0.00 %` | `+0.04 %` |
| Behavior RTT mean ms | `84.171` | `84.276` | `84.255` | `+0.10 %` | `-0.02 %` |
| Behavior RTT p95 ms | `130.802` | `130.372` | `130.928` | `+0.10 %` | `+0.43 %` |
| RTMPose RTT mean ms | `44.055` | `49.731` | `125.063` | `+183.88 %` | `+151.48 %` |
| RTMPose RTT p95 ms | `46.952` | `100.563` | `188.211` | `+300.86 %` | `+87.16 %` |
| Provider async wall ms | `108481.746` | `120029.376` | `139131.828` | `+28.25 %` | `+15.91 %` |
| Provider chunks | `1199` | `1199` | `1199` | `+0.00 %` | `+0.00 %` |
| Pose subjects | `19180` | `19180` | `19180` | `+0.00 %` | `+0.00 %` |
| Pose results | `19180` | `19180` | `19180` | `+0.00 %` | `+0.00 %` |
| Detection rows | `72744` | `72744` | `72744` | `+0.00 %` | `+0.00 %` |
| BBox rows | `72744` | `72744` | `72744` | `+0.00 %` | `+0.00 %` |
| Embedding rows | `72578` | `72578` | `72578` | `+0.00 %` | `+0.00 %` |
| Student tracks | `53` | `53` | `53` | `+0.00 %` | `+0.00 %` |

## Correctness

Both model-agreement comparisons reported `100.000 %` F1@IoU0.5 for:

| Model | Agreement vs batch 16 | Agreement vs prior batch 32 |
|---|---:|---:|
| `attention_tracking` | `100.000 %` | `100.000 %` |
| `hand_raising` | `100.000 %` | `100.000 %` |
| `person_detection` | `100.000 %` | `100.000 %` |
| `sitting_standing` | `100.000 %` | `100.000 %` |

This means the candidate is rejected for performance, not correctness.

## Decision Explanation

| Gate | Result | Explanation |
|---|---|---|
| Completion | PASS | Production job completed `4541/4541`. |
| Correctness | PASS | DB counters and model agreement were exact. |
| Batch-32 repair | FAIL | RTMPose p95 worsened from prior batch 32 by `+87.16 %`, and provider async wall worsened by `+15.91 %`. |
| Production value | FAIL | DB FPS worsened `-3.79 %` versus accepted batch 16 and `-2.84 %` versus prior batch 32. |
| GPU value | FAIL | GPU average fell below both accepted batch 16 and prior batch 32. |

The evidence indicates that concurrent internal RTMPose chunks created
contention rather than useful overlap. There is no accepted batch-32-specific
optimization from Cycle 14.C3.

## Production State

The wrapper restored the accepted production profile:

```text
POSE_TAIL_OPTIMIZATION_MODE=cross_frame_batch
POSE_CROSS_FRAME_BATCH_SIZE=16
POSE_PROVIDER_CHUNK_PARALLELISM=1
POSE_TAIL_PROFILING=0
EMBEDDING_STAGE_PROFILING=0
TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=1
EMBEDDING_PREFETCH_TRACK_LOOKUP=1
EMBEDDING_REDIS_SIDE_EFFECT_COALESCING=1
```

## Next Recommendation

Stop spending cycles on batch `32` until a new RTMPose runtime architecture
changes the engine or queueing constraint. The next sorted cycle is Cycle 14.D,
which must re-investigate server-side compact postprocessing/BLS/TRT-plugin
options against the now-accepted batch-16 profile before any implementation.
