# Cycle 14.C3 Batch-32 Parallel Provider Chunks Investigation

**Last updated:** 2026-06-03

**Status:** STAGED. No optimization decision exists until a completed
production Linux RTX 5090 benchmark on `combined.mp4` records before/after
metrics, correctness parity, and rollback proof.

## Problem Statement

Cycle 14.C2 proved that `POSE_CROSS_FRAME_BATCH_SIZE=32` is not acceptable in
its current form. It improved GPU average utilization, but regressed DB FPS,
Step 2 through-pose wall, and RTMPose p95 latency compared with the accepted
batch-16 production profile.

The measured shape suggests a narrow batch-32-specific candidate: keep the
outer cross-frame batch at `32`, but execute the two internal `16`-crop
RTMPose provider chunks concurrently instead of sequentially.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Results doc | `docs/cycle_14c_pose_batch_size_matrix_results.md` | Proves batch 32 improved GPU average utilization but regressed RTMPose p95 and Step 2 through-pose wall. |
| Plan doc | `docs/inference_parallelization_plan.md` | Records Cycle 14.C ordering and the production benchmark authority rule. |
| Execution log | `docs/crop_frame_optimization_execution.md` | Current sorted cycle map and rollback profile. |
| Runtime code | `backend/apps/pipeline/services/pose_runtime.py` | Owns RTMPose provider chunking and currently submits chunks sequentially. |
| Task code | `backend/apps/video_analysis/tasks.py` | Owns `POSE_CROSS_FRAME_BATCH_SIZE` outer flush behavior. |
| Settings code | `backend/config/settings/base.py` | Owns guarded runtime knobs. |
| Matrix wrapper | `tools/prod/prod_run_cycle14c_pose_batch_size_matrix.sh` | Reproducible Cycle 14.C benchmark precedent. |

## Evidence From Cycle 14.C

| Metric | Batch 16 baseline | Batch 32 | Delta |
|---|---:|---:|---:|
| DB FPS | `5.680314` | `5.624745` | `-0.98 %` |
| Step 2 through pose | `633.939 s` | `643.184 s` | `+1.46 %` |
| GPU avg util | `12.168 %` | `14.288 %` | `+17.42 %` |
| RTMPose RTT p95 | `46.952 ms` | `100.563 ms` | `+114.18 %` |
| Cross-frame flushes | `1199` | `600` | `-49.96 %` |
| Provider chunks | `1199` | `1199` | `0.00 %` |
| Provider async wall | `108.482 s` | `120.029 s` | `+10.64 %` |
| Model agreement F1 | baseline | `100.000 %` | preserved |

## Root Cause Hypothesis

`POSE_CROSS_FRAME_BATCH_SIZE=32` reduces outer flushes but `PoseRuntime` still
chunks to the deployed RTMPose engine cap of `16`. The current implementation
submits those chunks sequentially. For a full 32-crop flush, the outer loop
waits for chunk A and then chunk B, which can explain why GPU utilization rises
while p95 latency and provider async wall regress.

The candidate is to add a guarded `POSE_PROVIDER_CHUNK_PARALLELISM` setting.
The default `1` preserves existing behavior. The batch-32 candidate sets it to
`2` so the two 16-crop chunks can be in flight together.

## Expected Gain

This is not expected to beat batch 16 automatically. It is expected to answer
whether the batch-32 penalty was sequential internal chunk waiting rather than
the batch size itself.

Expected measurable effects:

| Metric | Expected direction |
|---|---|
| RTMPose p95 vs prior batch 32 | improve |
| Provider async wall vs prior batch 32 | improve |
| Step 2 through-pose vs prior batch 32 | improve |
| DB FPS vs prior batch 32 | improve |
| GPU avg vs batch 16 | remain higher or similar |

## Risk Assessment

| Risk | Level | Mitigation |
|---|---|---|
| Triton contention from two RTMPose chunks in flight | Medium | Guarded by `POSE_PROVIDER_CHUNK_PARALLELISM`; benchmark must prove RTT/p95 does not regress. |
| Ordering regression | Low | Responses are reassembled by chunk index before decode. |
| Correctness regression | Low | Tensor inputs are unchanged; production model-agreement gate is still mandatory. |
| Worker RSS increase | Low-medium | Benchmark collector captures memory; rollback restores batch 16 and parallelism 1. |

## Rollback Strategy

Restore the accepted Cycle 14.B2 profile and restart Celery workers:

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

## Acceptance Criteria

Cycle 14.C3 can be accepted only if a real production benchmark on
`combined.mp4` completes and proves all of the following:

| Gate | Required result |
|---|---|
| Completion | Job status `completed`, `4541/4541` frames. |
| Correctness | DB rows and model agreement preserve the accepted batch-16 baseline. |
| Batch-32 repair | RTMPose p95 and provider async wall improve versus prior Cycle 14.C2 batch 32. |
| Production value | DB FPS and Step 2 through-pose are no worse than accepted batch 16, or the decision table explicitly justifies a `NEEDS FURTHER ITERATION` status rather than acceptance. |
| Rollback | Wrapper restores `POSE_CROSS_FRAME_BATCH_SIZE=16` and `POSE_PROVIDER_CHUNK_PARALLELISM=1` unless the result is explicitly accepted. |

If these gates do not pass, batch 32 remains not accepted and the next sorted
cycle starts.
