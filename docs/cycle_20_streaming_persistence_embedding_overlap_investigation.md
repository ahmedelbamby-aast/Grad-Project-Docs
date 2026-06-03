# Cycle 20 Streaming Persistence and Embedding Overlap Investigation

**Last updated:** 2026-06-03

**Status:** Phase A investigation staged. No code is implemented by this
document. The only valid decision state is
`NO_DECISION_PRODUCTION_BENCHMARK_REQUIRED` until a completed production Linux
RTX 5090 benchmark on `combined.mp4` compares this candidate against the latest
accepted baseline.

## Question Answered

Yes, in the current offline `crop_frame` workflow the authoritative
`Frame`/`Detection`/`BoundingBox` database rows are persisted after the frame
inference loop has produced the full in-memory `frame_detections` map.
Embedding generation starts even later: after Step 3 persistence, render/audit
finalization, and the follow-up task handoff.

One nuance matters for operators: `processed_frames=total_frames` can appear
before all inference-side work has drained. The current frame counter is not a
proof that Step 3 or embedding has started.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| File | `backend/apps/video_analysis/tasks.py` | Owns `process_video_upload`, Step 3 persistence, render/finalization, `run_detection_and_tracking`, and `generate_embeddings`. |
| Doc | `docs/inference_parallelization_plan.md` | Current stage-concurrency table and accepted Cycle 12/13 roadmap. |
| Doc | `docs/production_inference_benchmark.md` | Production benchmark authority and post-stage timing history. |
| Doc | `docs/cycle_13_persistence_render_investigation.md` | First Cycle 13 decomposition of persistence/render/embedding tail. |
| Doc | `docs/cycle_13_embedding_profile_results.md` | Measurement-only embedding sub-stage wall breakdown. |
| Doc | `docs/cycle_13_embedding_track_lookup_results.md` | Latest accepted post-stage optimization baseline. |
| Doc | `docs/cycle_13c_redis_db_side_effect_measurement_investigation.md` | Active Redis/DB side-effect measurement that must complete before this cycle is prioritized. |
| Doc | `docs/redis_broader_optimization_opportunities.md` | Redis roadmap that this cycle depends on for optional boundary-state and progress-stream support. |
| Doc | `.specify/memory/constitution.md` | Production benchmark decision authority and lifecycle terminal-state gates. |

## Current Flow Evidence

The current code path is ordered as follows:

| Order | Current stage | Evidence |
|---:|---|---|
| 1 | Frame decode, person/behavior/pose inference, tracking aggregation | `process_video_upload` builds `frame_detections` in memory before Step 3. |
| 2 | Step 3 DB persistence | Step 3 iterates `sorted(frame_detections.items())` and writes `Frame`, `Detection`, `BoundingBox`, and `StudentTrack` rows. |
| 3 | Render and audit/finalization | `process_video_upload` writes render/audit artifacts before the follow-up handoff. |
| 4 | Follow-up tracking confirmation | `run_detection_and_tracking` confirms track metadata and queues or calls embedding generation. |
| 5 | Embedding generation | `generate_embeddings` reads persisted rows and writes `FrameEmbedding` rows plus Redis side effects. |

This means DB persistence and embedding are not currently overlapped with the
same job's frame inference. `OFFLINE_OFFLOAD_POST_STAGES=1` only moves the
follow-up stages out of the `process_video_upload` call path after finalization;
it does not stream rows while inference is still running.

## Why This Cycle Is Plausible

Cycle 12.C reduced Step 2 wall, and Cycle 13.B reduced embedding track lookup.
The accepted Cycle 13.B profile still leaves material post-stage work:

| Bucket | Latest accepted evidence |
|---|---:|
| Embedding profile wall | `121.681 s` |
| Redis flush | `59.874 s` |
| DB flush | `38.773 s` |
| Track lookup | `0.447 s` |

If the remaining persistence/embedding work can run behind inference without
starving Triton, total wall time may fall even when individual stage wall times
stay similar. The intended benefit is pipeline overlap, not faster model
execution.

## Why It Must Not Be Implemented Immediately

This is a lifecycle architecture change, not a small optimization. It affects
authoritative PostgreSQL rows, idempotency, job terminal-state semantics,
render inputs, embedding inputs, and benchmark evidence collection.

The Cycle 13.C / 16.A Redis command-cost benchmark finished and promoted Cycle
16.B Redis side-effect coalescing as the next Redis task. Cycle 20 should not
jump ahead of Cycle 16.B or Cycle 14-19 unless a completed production decision
table proves that post-stage overlap is now the highest-return lever.

## Proposed Cycle 20 Scope

Cycle 20 should investigate and then benchmark a streaming job contract:

| Component | Proposed behavior |
|---|---|
| Inference producer | After each frame batch is postprocessed, publish a bounded persistence packet instead of waiting for all frames. |
| Persistence worker | Bulk upsert `Frame`, `Detection`, `BoundingBox`, and `StudentTrack` rows per committed packet. |
| Embedding worker | Start on persisted frame windows once a safe row watermark is available. |
| Backpressure | Stop streaming or fall back to current Step 3 if the persistence queue exceeds a bounded threshold. |
| Terminal coordinator | Mark the job complete only after inference, streaming persistence, render, embedding, and ReID reach terminal states. |
| Evidence collector | Record overlap ratio, queue waits, row watermarks, DB parity, embedding parity, and rollback state. |

## New Measurements Required Before Code

| Metric | Purpose |
|---|---|
| `inference_done_at` | Establish when frame inference actually finishes. |
| `first_frame_persisted_at` | Prove whether streaming starts before inference ends. |
| `all_frames_persisted_at` | Measure DB persistence wall and overlap ratio. |
| `first_embedding_started_at` | Prove whether embedding starts before inference ends or only after persistence drains. |
| `embedding_done_at` | Measure total embedding overlap and tail. |
| `persist_queue_wait_ms` | Detect DB writer starvation or queue backlog. |
| `persist_batch_wall_ms` | Bound PostgreSQL write cost per packet. |
| `embedding_queue_wait_ms` | Detect Celery/Redis scheduling delay. |
| `triton_rtt_during_streaming` | Ensure DB/embedding overlap does not regress inference RTT. |
| `gpu_util_during_streaming` | Ensure side-stage CPU/DB work does not starve the GPU path. |

## Acceptance Criteria

Cycle 20 can be accepted only if a real production benchmark proves all gates:

| Gate | Required evidence |
|---|---|
| Production authority | Completed `combined.mp4` run on the Linux RTX 5090 with deployed SHA, replay key, job ID, env fingerprint, GPU CSV, and evidence bundle. |
| Total throughput | Total wall and DB-completed FPS improve against the latest accepted baseline. |
| Inference protection | Step 2 wall, behavior RTT mean/p95, and GPU utilization do not materially regress. |
| Overlap proof | Persistence or embedding starts before inference finishes, with measured overlap ratio. |
| Correctness | DB row parity, StudentTrack count, FrameEmbedding count, and model-agreement F1 remain within the active gate. |
| Stability | No stale-state failure, duplicate rows, orphan rows, non-terminal jobs, or Redis/Celery queue leak. |
| Decision table | `docs/production_inference_benchmark.md` contains the required before/after explanation table. |

## Rollback Strategy

The implementation must be guarded by a disabled-by-default flag, for example:

```text
OFFLINE_STREAM_POST_STAGES=0
```

Rollback must restore the current behavior: build all `frame_detections` in
memory, run Step 3 persistence after inference, then queue follow-up embedding.
The rollback must require only an env flag reset and worker restart unless a
future implementation explicitly documents a schema migration.

## Ordering Decision

Stage this as **Cycle 20** after Cycle 16.B and the planned Cycle 14-19
roadmap. It is last by default because it is a broader lifecycle contract
change. It may be moved earlier only after a completed production benchmark
decision table proves that post-stage tail, not inference wall or Redis
side-effect coalescing, is the next dominant limiter.
