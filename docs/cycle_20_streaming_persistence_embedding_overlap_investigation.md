# Cycle 20 Streaming Persistence and Embedding Overlap Investigation

**Last updated:** 2026-06-06

**Status:** Phase C terminal-coordinator timestamp repair started after the
Phase B measurement-only production benchmark recorded the current serial
post-stage gaps. The only valid benchmark decision state remains
`NO_DECISION_PENDING_REVIEW`: replay
`cycle20-post-stage-timeline-20260605T212526Z` completed on production, but
this work still does **not** implement streaming persistence, embedding overlap,
new queues, or terminal-coordinator behavior.

**Streaming compatibility:** `offline-only` for the first implementation
profile. The candidate relies on offline job lifecycle coordination and must not
run on RTSP, RTSPS, WHEP/WebRTC, or HLS live profiles unless a separate live
design proves bounded per-camera queues, append-only corrections, and
per-frame latency budgets.

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
| File | `backend/config/settings/base.py` | Defines the default-off Cycle 20 flags `OFFLINE_STREAM_POST_STAGES` and `OFFLINE_STREAM_POST_STAGE_TIMELINE`. |
| File | `backend/apps/video_analysis/models.py` | Defines `Frame`, `Detection`, `BoundingBox`, `StudentTrack`, and `FrameEmbedding` constraints. |
| File | `backend/apps/tracking/embeddings.py` | Defines `persist_embedding` and `persist_embeddings_bulk` write-boundary behavior. |
| File | `tools/prod/prod_enable_parallel_flow.sh` | Current offline profile switchboard and rollback-flag authority. |
| File | `tools/prod/prod_run_cycle20_post_stage_timeline_benchmark.sh` | Measurement-only production wrapper for the first Cycle 20 benchmark. |
| File | `tools/prod/prod_collect_benchmark_metrics.py` | Collects the Cycle 20 timeline metadata into raw metrics JSON/Markdown. |
| File | `tools/prod/prod_watch_benchmark_metrics.sh` | Exposes the Cycle 20 timeline while a benchmark is running. |
| File | `tools/prod/prod_triton_endpoint_policy.sh` | Keeps `OFFLINE_STREAM_POST_STAGES=0` on the live profile. |
| Test | `backend/tests/unit/video_analysis/test_cycle20_post_stage_timeline.py` | Verifies default-off and timestamp/unavailable metadata behavior. |
| Test | `backend/tests/unit/pipeline/test_prod_collect_benchmark_metrics.py` | Verifies derived Cycle 20 timeline metrics. |
| Commit | `ba4e2882` | Production runtime commit used for the Cycle 20 measurement replay. |
| Commit | `70aebb99` | Wrapper evidence-manifest fix deployed before strict figure regeneration. |
| Job | `58d53985-1c86-46fd-944c-771ea3afce1a` | Production Cycle 20 measurement job for replay `cycle20-post-stage-timeline-20260605T212526Z`. |
| Doc | `docs/inference_parallelization_plan.md` | Current stage-concurrency table and accepted Cycle 12/13 roadmap. |
| Doc | `docs/production_inference_benchmark.md` | Production benchmark authority and post-stage timing history. |
| Doc | `docs/cycle_13_persistence_render_investigation.md` | First Cycle 13 decomposition of persistence/render/embedding tail. |
| Doc | `docs/cycle_13_embedding_profile_results.md` | Measurement-only embedding sub-stage wall breakdown. |
| Doc | `docs/cycle_13_embedding_track_lookup_results.md` | Latest accepted post-stage optimization baseline. |
| Doc | `docs/cycle_13c_redis_db_side_effect_measurement_investigation.md` | Active Redis/DB side-effect measurement that must complete before this cycle is prioritized. |
| Doc | `docs/redis_broader_optimization_opportunities.md` | Redis roadmap that this cycle depends on for optional boundary-state and progress-stream support. |
| Doc | `.specify/memory/constitution.md` | Production benchmark decision authority and lifecycle terminal-state gates. |
| Turn ledger | `docs/agent_20_remaining_lanes_turn.md` | Agent 20 readiness ownership and non-overlap boundaries. |

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

## Why It Must Not Jump Directly To Streaming

This is a lifecycle architecture change, not a small optimization. It affects
authoritative PostgreSQL rows, idempotency, job terminal-state semantics,
render inputs, embedding inputs, and benchmark evidence collection.

The Cycle 13.C / 16.A Redis command-cost benchmark already promoted and then
completed Cycle 16.B Redis side-effect coalescing. Cycle 18.D is now complete
and not accepted, and the sorted queue promotes Cycle 20 as the next
non-sharding latency lane. That promotion authorizes only the first
measurement-only implementation slice: record lifecycle timestamps and prove
the current serial gaps before any writer, queue, worker split, or terminal
coordinator changes production behavior.

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

## Agent 20 Readiness Tasks

Agent 20 owns readiness documentation only. These tasks do not authorize runtime
implementation:

| Task | State | Required output |
|---|---|---|
| `A20-C20-01` | `COMPLETED` | Measurement-only timestamp contract for inference/persistence/embedding overlap. |
| `A20-C20-02` | `COMPLETED` | Idempotency checklist for `Frame`, `Detection`, `BoundingBox`, `StudentTrack`, and `FrameEmbedding` writes. |
| `A20-C20-03` | `COMPLETED` | Terminal coordinator entry gate that prevents completed status before required side stages finish. |
| `A20-C20-04` | `COMPLETED` | Rollback and live-profile exclusion notes for any future `OFFLINE_STREAM_POST_STAGES` flag. |

Runtime implementation remains blocked until these contracts are reviewed and a
separate implementation turn is explicitly opened.

## Readiness Contract V0

This contract is documentation-only. It defines what a later implementation
must prove before streaming persistence or embedding overlap can be benchmarked.

## Phase B Measurement-Only Kickoff (2026-06-05)

This kickoff starts Cycle 20 without enabling the future streaming behavior.
The code adds:

| Surface | Change |
|---|---|
| Settings | `OFFLINE_STREAM_POST_STAGES=0` remains the future rollback flag; `OFFLINE_STREAM_POST_STAGE_TIMELINE=0` defaults the measurement probe off. |
| Upload task | When the timeline flag is enabled, `process_video_upload` records `inference_started_at`, `frame_inference_done_at`, `first_frame_persisted_at`, `all_frames_persisted_at`, and `first_embedding_eligible_at` in `VideoAnalysisJob.metadata.cycle20_post_stage_timeline`. |
| Embedding task | `generate_embeddings` records `first_embedding_started_at`, `embedding_done_at`, created/skipped/error counters, and the embedding stage outcome in the same metadata block. |
| Terminal stage | Phase B attempted `terminal_coordinator_done_at`; Phase C moves the measurement marker before the ReID task reports terminal `COMPLETED` status so the benchmark collector cannot observe completion before the marker is present. |
| Missing packet fields | `first_persist_packet_ready_at` is explicitly marked unavailable with reason `serial_path_no_streaming_persistence_packet`; it is not faked as a zero timestamp. |
| Metrics collector | `prod_collect_benchmark_metrics.py` emits `post_stage_timeline` with timestamps, unavailable reasons, missing required fields, and derived serial gaps. |
| Production wrapper | `prod_run_cycle20_post_stage_timeline_benchmark.sh` enables only `OFFLINE_STREAM_POST_STAGE_TIMELINE=1`, forces `OFFLINE_STREAM_POST_STAGES=0`, collects metrics/model agreement, generates figures, and restores both flags to `0`. |

Figure Planner: Cycle 20 kickoff agent. Required plots consume the baseline
metrics JSON, candidate metrics JSON, model-agreement JSON, rollback JSON, and
figure input manifest. Missing streaming-only fields render as unavailable,
not zero.

Figure Implementer: Cycle 20 kickoff agent. Role separation is unavailable in
this single-agent kickoff; the wrapper still keeps planner intent, generator
inputs, unavailable-metric policy, manifest, and produced images as separate
artifacts before any benchmark decision is claimed.

The Phase B benchmark remains `NO_DECISION_PENDING_REVIEW`: production evidence
is captured, but the run proves the current serial gaps only and does not
enable the future streaming writer, embedding overlap, or terminal coordinator.

### Phase B production measurement (2026-06-05 UTC / 2026-06-06 Cairo)

Production replay `cycle20-post-stage-timeline-20260605T212526Z` ran on
`combined.mp4` as job `58d53985-1c86-46fd-944c-771ea3afce1a`. The benchmark
completed `4541/4541` frames with `OFFLINE_STREAM_POST_STAGES=0` and
`OFFLINE_STREAM_POST_STAGE_TIMELINE=1` during the run. Rollback then restored
both Cycle 20 flags to `0`.

| Evidence item | Value |
|---|---|
| Runtime commit | `ba4e2882` |
| Figure/evidence fix commit | `70aebb99` |
| Replay key | `cycle20-post-stage-timeline-20260605T212526Z` |
| Job ID | `58d53985-1c86-46fd-944c-771ea3afce1a` |
| Metrics JSON/MD | `/home/bamby/grad_project/backend/logs/cycle20-post-stage-timeline-20260605T212526Z/post_stage_timeline_metrics.json` / `.md` |
| Model agreement JSON/MD | `/home/bamby/grad_project/backend/logs/cycle20-post-stage-timeline-20260605T212526Z/model_agreement_baseline_vs_post_stage_timeline.json` / `.md` |
| Rollback JSON/MD | `/home/bamby/grad_project/backend/logs/cycle20-post-stage-timeline-20260605T212526Z/rollback_status.json` / `.md` |
| Figure manifest | `docs/figures/benchmark_artifacts/cycle20-post-stage-timeline-20260605T212526Z/figure_manifest.json` |
| Figure roles | Planner `Agent Cycle 20 kickoff`; implementer `Agent Cycle 20 kickoff` |

Measured serial lifecycle gaps:

| Metric | Value |
|---|---:|
| DB-completed FPS | `5.216317` |
| Step 2 frame wall | `462.124911 s` |
| Step 2 through-pose wall | `691.776287 s` |
| Persistence starts before inference done | `false` |
| Persistence wall | `52.703285 s` |
| Embedding starts before inference done | `false` |
| Embedding start lag after inference done | `78.581286 s` |
| Embedding wall | `98.739642 s` |
| Detection / bbox rows | `72744 / 72744` |
| Embedding rows | `72578` |
| Student tracks | `53` |
| Model agreement F1@IoU0.5 | `100.000 %` for all four behavior models |
| Rollback verified | `true` |

The measurement proves the current path is serial for the Cycle 20 boundaries:
first frame persistence happened after frame inference finished, and embedding
started after frame inference finished. It also exposes one instrumentation gap:
`terminal_coordinator_done_at` remained missing, so later Cycle 20 work must
either record that boundary reliably or keep it explicitly unavailable with a
reason before any decision claim.

### Phase C terminal-marker repair start (2026-06-06)

The next Cycle 20 step has started repo-side as a default-off measurement
repair. `run_reid_pipeline` now writes
`VideoAnalysisJob.metadata.cycle20_post_stage_timeline.terminal_coordinator_done_at`
before it calls `_set_job_status(...COMPLETED...)`. This keeps the Phase B
measurement flag contract but closes the race where benchmark collection could
observe terminal status before the terminal marker was persisted.

The focused unit test
`backend/tests/unit/video_analysis/test_cycle20_post_stage_timeline.py`
verifies that the marker is already present in PostgreSQL when the ReID task
attempts the terminal status update. The repair remains gated by
`OFFLINE_STREAM_POST_STAGE_TIMELINE=1`; live-source metadata is still excluded by
the existing `_cycle20_post_stage_timeline_enabled` guard.

Figure Planner for the next production rerun: Cycle 20.C terminal-marker repair
agent. Required plots and manifest inputs are unchanged from Phase B because the
rerun still measures the same serial lifecycle fields and unavailable streaming
packet fields.

Figure Implementer for the next production rerun: Cycle 20.C terminal-marker
repair agent. Role separation is unavailable in this single-agent repair; the
next wrapper run must still keep the planner intent, figure input manifest,
unavailable-metric policy, generated plots, and rollback proof separate.

No production rerun has been recorded for Phase C yet. Until a replay proves the
marker and regenerates the evidence bundle, Section 48 of
`docs/production_inference_benchmark.md` remains the governing production result.

```text
BENCHMARK_LOCK
agent: Cycle 20.C terminal-marker repair agent
cycle: Cycle 20.C terminal-marker repair
replay_key: cycle20c-terminal-marker-20260605T225327Z
baseline_metrics: cycle15b-pre-shard-baseline-20260603T193531Z
candidate_env_delta: OFFLINE_STREAM_POST_STAGES=0; OFFLINE_STREAM_POST_STAGE_TIMELINE=1 during wrapper only
started_at_utc: 2026-06-05T22:53:27Z
expected_cleanup: restore OFFLINE_STREAM_POST_STAGES=0 and OFFLINE_STREAM_POST_STAGE_TIMELINE=0; restart Celery workers; record rollback_status.json and figures
```

```text
BENCHMARK_RELEASE
agent: Cycle 20.C terminal-marker repair agent
cycle: Cycle 20.C terminal-marker repair
replay_key: cycle20c-terminal-marker-20260605T225327Z
job_id: dd807237-24b5-49c7-a392-db41f9b7e128
status: POST_STAGE_WAIT_FAILED_NEEDS_REVIEW
metrics_json: /home/bamby/grad_project/backend/logs/cycle20c-terminal-marker-20260605T225327Z/post_stage_timeline_metrics.json
metrics_md: /home/bamby/grad_project/backend/logs/cycle20c-terminal-marker-20260605T225327Z/post_stage_timeline_metrics.md
model_agreement_json: /home/bamby/grad_project/backend/logs/cycle20c-terminal-marker-20260605T225327Z/model_agreement_baseline_vs_post_stage_timeline.json
model_agreement_md: /home/bamby/grad_project/backend/logs/cycle20c-terminal-marker-20260605T225327Z/model_agreement_baseline_vs_post_stage_timeline.md
rollback_json: /home/bamby/grad_project/backend/logs/cycle20c-terminal-marker-20260605T225327Z/rollback_status.json
figure_manifest: /home/bamby/grad_project/backend/logs/cycle20c-terminal-marker-20260605T225327Z/figures/figure_manifest.json
released_at_utc: 2026-06-05T23:12:22Z
notes: Lower-level benchmark returned while the job was still in embedding; the wrapper disabled OFFLINE_STREAM_POST_STAGE_TIMELINE before embedding_done_at and terminal_coordinator_done_at could be recorded.
```

```text
BENCHMARK_LOCK
agent: Cycle 20.C terminal-marker repair agent
cycle: Cycle 20.C terminal-marker repair
replay_key: cycle20c-terminal-marker-r2-20260605T231222Z
baseline_metrics: cycle15b-pre-shard-baseline-20260603T193531Z
candidate_env_delta: OFFLINE_STREAM_POST_STAGES=0; OFFLINE_STREAM_POST_STAGE_TIMELINE=1 during wrapper only; wrapper waits for terminal timeline markers before rollback
started_at_utc: 2026-06-05T23:12:22Z
expected_cleanup: restore OFFLINE_STREAM_POST_STAGES=0 and OFFLINE_STREAM_POST_STAGE_TIMELINE=0; restart Celery workers; record rollback_status.json, post_stage_wait_snapshot.json, and figures
```

```text
BENCHMARK_RELEASE
agent: Cycle 20.C terminal-marker repair agent
cycle: Cycle 20.C terminal-marker repair
replay_key: cycle20c-terminal-marker-r2-20260605T231222Z
job_id: edf7b205-3e96-47b6-91fc-fadbe3f977ec
status: POST_STAGE_WAIT_FAILED_NEEDS_REVIEW
metrics_json: /home/bamby/grad_project/backend/logs/cycle20c-terminal-marker-r2-20260605T231222Z/post_stage_timeline_metrics.json
metrics_md: /home/bamby/grad_project/backend/logs/cycle20c-terminal-marker-r2-20260605T231222Z/post_stage_timeline_metrics.md
model_agreement_json: /home/bamby/grad_project/backend/logs/cycle20c-terminal-marker-r2-20260605T231222Z/model_agreement_baseline_vs_post_stage_timeline.json
model_agreement_md: /home/bamby/grad_project/backend/logs/cycle20c-terminal-marker-r2-20260605T231222Z/model_agreement_baseline_vs_post_stage_timeline.md
rollback_json: /home/bamby/grad_project/backend/logs/cycle20c-terminal-marker-r2-20260605T231222Z/rollback_status.json
figure_manifest: /home/bamby/grad_project/backend/logs/cycle20c-terminal-marker-r2-20260605T231222Z/figures/figure_manifest.json
released_at_utc: 2026-06-05T23:30:53Z
notes: Job reached terminal completed and recorded embedding_done_at plus terminal_coordinator_done_at, but the new wrapper wait subprocess ran from the repo root and failed to import Django's config module, so the wrapper status stayed failed.
```

```text
BENCHMARK_LOCK
agent: Cycle 20.C terminal-marker repair agent
cycle: Cycle 20.C terminal-marker repair
replay_key: cycle20c-terminal-marker-r3-20260605T233053Z
baseline_metrics: cycle15b-pre-shard-baseline-20260603T193531Z
candidate_env_delta: OFFLINE_STREAM_POST_STAGES=0; OFFLINE_STREAM_POST_STAGE_TIMELINE=1 during wrapper only; wrapper waits for terminal timeline markers from backend/ before rollback
started_at_utc: 2026-06-05T23:30:53Z
expected_cleanup: restore OFFLINE_STREAM_POST_STAGES=0 and OFFLINE_STREAM_POST_STAGE_TIMELINE=0; restart Celery workers; record rollback_status.json, post_stage_wait_snapshot.json, and figures
```

### Measurement-only timestamp contract

Every future implementation must emit these fields into the benchmark evidence
bundle before it changes production behavior:

| Field | Producer boundary | Required proof |
|---|---|---|
| `inference_started_at` | `process_video_upload` task start | Establish the total overlap denominator. |
| `frame_inference_done_at` | after frame inference builds `frame_detections` | Distinguish inference drain from progress counters. |
| `first_persist_packet_ready_at` | first completed postprocessed frame batch | Prove packets exist before Step 3 drains. |
| `first_frame_persisted_at` | first committed `Frame` row | Prove persistence starts before inference ends. |
| `all_frames_persisted_at` | final committed persistence packet | Bound DB persistence wall and lag. |
| `first_embedding_eligible_at` | first safe persisted-row watermark | Prove embedding work has valid inputs. |
| `first_embedding_started_at` | first embedding worker loop entry | Prove embedding overlap or prove it stayed serial. |
| `embedding_done_at` | final embedding flush | Bound post-stage tail. |
| `terminal_coordinator_done_at` | final job terminal write | Prove completion waited for all required stages. |

Missing fields must be recorded as `unavailable` with a reason, not converted
to zero. The production decision table remains invalid until the timestamp set
shows whether overlap actually occurred.

### Persistence idempotency checklist

| Durable record | Current evidence | Future Cycle 20 requirement |
|---|---|---|
| `Frame` | `Frame` has unique `(job, frame_number)` and Step 3 uses `update_or_create`. | Streaming writer keeps `(job, frame_number)` as the semantic key. |
| `StudentTrack` | `StudentTrack` has unique `(job, tracking_id)` and Step 3 uses `get_or_create`. | Track updates are merge-safe and never create duplicate tracks on retry. |
| `Detection` | Current Step 3 can `bulk_create` detections for each frame. | Streaming packets need a semantic detection key or packet replace policy before retry is allowed. |
| `BoundingBox` | Current Step 3 creates boxes alongside detections. | Box writes must be tied to the detection key and converge on retry. |
| `FrameEmbedding` | `generate_embeddings` skips detections that already have embeddings. | Embedding windows must preserve detection-level skip behavior and never orphan rows. |
| Job metadata | Current task writes progress and stage metadata in the upload task. | Stage outcome counters must be append-only until the terminal coordinator writes final state. |

The first implementation slice must be measurement-only unless the detection
and bounding-box idempotency key is explicit.

### Terminal coordinator entry gate

The job may be marked terminal only after these states are all available:

| Stage | Required terminal marker | Failure behavior |
|---|---|---|
| Inference producer | `frame_inference_done_at` and frame count | Fail closed if required detections are zero without an allowed partial policy. |
| Persistence stream | persisted frame/detection/box counts plus last packet ID | Fail closed on row mismatch, duplicate packet, or non-idempotent retry. |
| Render/audit | render status and audit artifact path or explicit skip reason | Do not report completed if render/audit is still pending. |
| Embedding worker | embedding count, skipped-existing count, and final flush time | Fail closed on orphan embeddings or missing required vectors. |
| Redis/side effects | Redis counters or `unavailable` reason | Do not treat Redis loss as PostgreSQL success evidence. |

This gate is the reason Cycle 20 remains a lifecycle cycle rather than a simple
queue optimization.

### Rollback and live-profile exclusion

Any future implementation must keep the rollback shape:

```text
OFFLINE_STREAM_POST_STAGES=0
```

Rollback must restore the current serial path: build `frame_detections`, run
Step 3 persistence, render/audit, then dispatch the follow-up embedding chain.
Because this first profile is `offline-only`, any shipped flag must also be
explicitly disabled in the live profile block of
`tools/prod/prod_enable_parallel_flow.sh`.

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
roadmap. It is late by default because it is a broader lifecycle contract
change. It may be moved earlier only after a completed production benchmark
decision table proves that post-stage tail, not inference wall or Redis
side-effect coalescing, is the next dominant limiter. If Cycle 20 creates
independent streaming persistence or embedding tasks, run the Cycle 21
concurrency matrix afterward rather than raising worker counts blindly.
