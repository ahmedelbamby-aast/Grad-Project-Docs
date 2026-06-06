# Cycle 20 Streaming Persistence and Embedding Overlap Investigation

**Last updated:** 2026-06-06

**Status:** Phase E final-stable persistence overlap is production benchmarked
and **NOT ACCEPTED**. Replay
`cycle20e-final-stable-overlap-20260606T092512Z` repaired the Cycle 20.D root
causes: final-stable packets persisted once per frame, Step 3 reconciliation
fell to `0`, and rollback restored all Cycle 20 flags to defaults. It still
failed the optimization gate because DB-completed FPS regressed, total elapsed
regressed, Step 2 through-pose wall regressed, and embedding stayed serial. The
benchmark lock is released.

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
| Commit | `a6bf10a3` | Wrapper wait gate that keeps the timeline flag enabled until required post-stage markers are recorded. |
| Commit | `7bf66e97` | Backend-context wait fix used for the final Cycle 20.C terminal-marker production replay. |
| Commit | `4e294f52` | Packet-signature reconciliation deployed for the final Cycle 20.D r3 production replay. |
| Commit | `a9831c90` | Generated Triton offline profile binding fix and runtime commit used for the Cycle 20.E production replay. |
| Job | `58d53985-1c86-46fd-944c-771ea3afce1a` | Production Cycle 20 measurement job for replay `cycle20-post-stage-timeline-20260605T212526Z`. |
| Job | `7ff0dfd4-890e-4210-92c7-f0f3b069c65e` | Production Cycle 20.C terminal-marker replay job for `cycle20c-terminal-marker-r3-20260605T233053Z`. |
| Job | `24e9970f-b3bc-451d-ab50-b0bcbb1e8d8b` | Production Cycle 20.D r3 job for replay `cycle20d-streaming-persistence-r3-20260606T011056Z`. |
| Job | `211f96fa-7cfc-4d57-812e-9573906f41c5` | Production Cycle 20.E final-stable persistence replay job for `cycle20e-final-stable-overlap-20260606T092512Z`. |
| Figure manifest | `docs/figures/benchmark_artifacts/cycle20e-final-stable-overlap-20260606T092512Z/figure_manifest.json` | Generated figure manifest, input digests, unavailable-metric reasons, and status `NOT_ACCEPTED` for Cycle 20.E. |
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
completed Cycle 16.B Redis side-effect coalescing. Cycle 18.D is complete and
not accepted, and Cycle 20 has now produced measurement-only, terminal-marker,
Phase D streaming-writer, and Phase E final-stable production records. Phase E
repaired the Step 3 reconciliation root causes, but still did not improve total
throughput or embedding overlap, so future work must not assume that
final-stable persistence alone creates useful post-stage parallelism.

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
| Production wrapper | `prod_run_cycle20_post_stage_timeline_benchmark.sh` defaults to the measurement-only profile with `OFFLINE_STREAM_POST_STAGES=0`; Phase D passes `--stream-post-stages 1` to enable the default-off writer for one governed replay. The wrapper collects metrics/model agreement, generates figures, and restores both flags to `0`. |

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

The first two Phase C attempts are retained as failed wrapper evidence because
they exposed lifecycle collection gaps. The final r3 replay proves the terminal
marker repair while preserving the measurement-only contract. Section 49 of
`docs/production_inference_benchmark.md` is now the governing Phase C production
record.

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

```text
BENCHMARK_RELEASE
agent: Cycle 20.C terminal-marker repair agent
cycle: Cycle 20.C terminal-marker repair
replay_key: cycle20c-terminal-marker-r3-20260605T233053Z
job_id: 7ff0dfd4-890e-4210-92c7-f0f3b069c65e
status: BENCHMARK_RECORDED_NEEDS_DOC_DECISION
metrics_json: /home/bamby/grad_project/backend/logs/cycle20c-terminal-marker-r3-20260605T233053Z/post_stage_timeline_metrics.json
metrics_md: /home/bamby/grad_project/backend/logs/cycle20c-terminal-marker-r3-20260605T233053Z/post_stage_timeline_metrics.md
model_agreement_json: /home/bamby/grad_project/backend/logs/cycle20c-terminal-marker-r3-20260605T233053Z/model_agreement_baseline_vs_post_stage_timeline.json
model_agreement_md: /home/bamby/grad_project/backend/logs/cycle20c-terminal-marker-r3-20260605T233053Z/model_agreement_baseline_vs_post_stage_timeline.md
wait_snapshot_json: /home/bamby/grad_project/backend/logs/cycle20c-terminal-marker-r3-20260605T233053Z/post_stage_wait_snapshot.json
rollback_json: /home/bamby/grad_project/backend/logs/cycle20c-terminal-marker-r3-20260605T233053Z/rollback_status.json
figure_manifest: /home/bamby/grad_project/backend/logs/cycle20c-terminal-marker-r3-20260605T233053Z/figures/figure_manifest.json
released_at_utc: 2026-06-05T23:48:26Z
notes: Job completed 4541/4541 frames, wait snapshot status ready, missing_required empty, rollback restored OFFLINE_STREAM_POST_STAGES=0 and OFFLINE_STREAM_POST_STAGE_TIMELINE=0.
```

### Phase C production rerun result (2026-06-06)

Replay `cycle20c-terminal-marker-r3-20260605T233053Z` / job
`7ff0dfd4-890e-4210-92c7-f0f3b069c65e` completed with deployed commit
`7bf66e97`. The wrapper kept `OFFLINE_STREAM_POST_STAGES=0`, enabled only
`OFFLINE_STREAM_POST_STAGE_TIMELINE=1` during the replay, waited from the
`backend/` context until the required timeline markers existed, then rolled both
Cycle 20 flags back to `0` and restarted workers.

| Gate | Result |
|---|---|
| Wait snapshot | `status=ready`, `missing_required=[]` |
| Processed frames | `4541/4541` |
| DB-completed FPS | `5.274878` |
| Step 2 frame wall | `462.458481 s` |
| Persistence wall | `43.184512 s` |
| Embedding start lag after inference done | `69.173672 s` |
| Embedding wall | `99.369505 s` |
| Terminal lag after embedding | `0.244864 s` |
| Detection / bbox / embedding rows | `72744` / `72744` / `72578` |
| StudentTracks | `53` |
| Per-model agreement F1@IoU0.5 | `1.0` for attention, hand raising, person detection, and sitting/standing |
| GPU avg / peak util | `11.691 %` / `53.0 %` |
| Rollback | `rollback_verified=true`; both Cycle 20 flags restored to `0` |

The repaired terminal timestamp set is:

| Timestamp | Value |
|---|---|
| `inference_started_at` | `2026-06-05T23:33:44.365107+00:00` |
| `frame_inference_done_at` | `2026-06-05T23:45:15.390185+00:00` |
| `first_frame_persisted_at` | `2026-06-05T23:45:15.405796+00:00` |
| `all_frames_persisted_at` | `2026-06-05T23:45:58.590308+00:00` |
| `first_embedding_eligible_at` | `2026-06-05T23:45:58.592820+00:00` |
| `first_embedding_started_at` | `2026-06-05T23:46:24.563857+00:00` |
| `embedding_done_at` | `2026-06-05T23:48:03.933362+00:00` |
| `terminal_coordinator_done_at` | `2026-06-05T23:48:04.178226+00:00` |
| `first_persist_packet_ready_at` | unavailable: `serial_path_no_streaming_persistence_packet` |

This closes the terminal-marker measurement gap only. It does not accept Cycle
20 because persistence and embedding still start after frame inference, and no
streaming writer or embedding-overlap candidate has been benchmarked.

![Cycle 20.C decision delta](figures/benchmark_artifacts/cycle20c-terminal-marker-r3-20260605T233053Z/cycle20_post_stage_timeline__decision_delta.png)

![Cycle 20.C wall breakdown](figures/benchmark_artifacts/cycle20c-terminal-marker-r3-20260605T233053Z/cycle20_post_stage_timeline__wall_breakdown.png)

![Cycle 20.C correctness gate](figures/benchmark_artifacts/cycle20c-terminal-marker-r3-20260605T233053Z/cycle20_post_stage_timeline__correctness_gate.png)

![Cycle 20.C evidence completeness](figures/benchmark_artifacts/cycle20c-terminal-marker-r3-20260605T233053Z/cycle20_post_stage_timeline__evidence_completeness.png)

### Phase D streaming persistence writer implementation (2026-06-06)

Phase D opens the first default-off behavior candidate after the Phase C
terminal-marker replay. The implementation still does not add embedding window
workers or new Celery queues. It moves only the authoritative
`Frame`/`Detection`/`BoundingBox`/`StudentTrack` persistence packet into the
per-frame inference callback when `OFFLINE_STREAM_POST_STAGES=1` and the job is
not live-source metadata.

| Surface | Phase D repo-side change |
|---|---|
| Upload task | `backend/apps/video_analysis/tasks.py` adds `_cycle20_stream_post_stages_enabled`, a live-source exclusion guard, `_persist_offline_detection_frame_packet`, and a Step 2 callback that writes one completed frame packet before Step 3. |
| Idempotency policy | Existing pre-embedding frame packets are replaced before retry; packets with existing `FrameEmbedding` rows are skipped so embedding evidence is never orphaned or deleted. |
| Step 3 | Step 3 becomes reconciliation when the flag is enabled: it skips frame packets whose detection count already matches, repairs missing or mismatched packets before embeddings start, and records reconciliation counters. |
| Timeline metadata | The candidate records `first_persist_packet_ready_at`, `first_frame_persisted_at`, stream packet counters, Step 3 reconciliation counters, and `embedding_watermark=streaming_persistence_all_frames`. |
| Metrics collector | `tools/prod/prod_collect_benchmark_metrics.py` exposes stream packet counters and derived `first_packet_ready_before_inference_done` evidence. |
| Production wrapper | `tools/prod/prod_run_cycle20_post_stage_timeline_benchmark.sh --stream-post-stages 1` enables the writer for a single replay and requires `first_persist_packet_ready_at` before rollback. |
| CI gate | `.github/workflows/inference-parallelization.yml` now syntax-checks the Cycle 20 wrapper and runs `backend/tests/unit/video_analysis/test_cycle20_post_stage_timeline.py`. |
| Focused tests | `backend/tests/unit/video_analysis/test_cycle20_post_stage_timeline.py` verifies streaming profile metadata, live exclusion, duplicate-free packet replacement, and existing-embedding protection; `backend/tests/unit/pipeline/test_prod_collect_benchmark_metrics.py` verifies Phase D metric extraction. |

Figure Planner: Cycle 20.D streaming persistence writer agent. Required plots
consume the accepted baseline metrics JSON, Phase D candidate metrics JSON,
model-agreement JSON, rollback JSON, runtime probe, post-stage wait snapshot,
and the figure input manifest. `first_persist_packet_ready_at` is required for
Phase D. `persist_queue_wait_ms` remains unavailable because this candidate
uses an inline callback writer rather than a separate persistence queue.
`embedding_queue_wait_ms` remains unavailable because embedding overlap is not
implemented in this phase. These unavailable metrics must render with reasons,
not zeros.

Figure Implementer: Cycle 20.D streaming persistence writer agent. Role
separation is unavailable in this single-agent implementation turn; the planner
policy above and the generator/wrapper implementation are still kept as separate
evidence: wrapper option, figure input manifest, generated figures, and final
Markdown embeds after the production replay.

Local validation before the production lock:

| Gate | Result |
|---|---|
| Python compile | `python -m py_compile backend/apps/video_analysis/tasks.py tools/prod/prod_collect_benchmark_metrics.py` passed. |
| Wrapper syntax | `bash -n tools/prod/prod_run_cycle20_post_stage_timeline_benchmark.sh` passed. |
| Focused pytest | `..\\.venv\\Scripts\\python.exe -m pytest tests/unit/video_analysis/test_cycle20_post_stage_timeline.py tests/unit/pipeline/test_prod_collect_benchmark_metrics.py -q` passed with `16 passed`. |
| Dry-run wrapper | `bash tools/prod/prod_run_cycle20_post_stage_timeline_benchmark.sh --stream-post-stages 1 --tag cycle20d-dry-run --figure-cycle-slug cycle20d_streaming_persistence --figure-cycle-label 'Cycle 20.D streaming persistence writer' --dry-run` passed and showed rollback to both flags `0`. |
| Git whitespace | `git diff --check` passed with line-ending warnings only. |

Planned production command after commit/push and production hash parity:

```bash
bash tools/prod/prod_run_cycle20_post_stage_timeline_benchmark.sh \
  --stream-post-stages 1 \
  --tag cycle20d-streaming-persistence-<UTC> \
  --figure-cycle-slug cycle20d_streaming_persistence \
  --figure-cycle-label "Cycle 20.D streaming persistence writer" \
  --figure-planner "Cycle 20.D streaming persistence writer agent" \
  --figure-implementer "Cycle 20.D streaming persistence writer agent"
```

The Phase D decision is now recorded in
`docs/production_inference_benchmark.md` §50 as **NOT ACCEPTED**. The r3 replay
completed with rollback proof, raw metrics, model-agreement evidence, and
generated figures, but the candidate failed the throughput, inference
protection, GPU-utilization, and Step 3 elimination gates.

```text
BENCHMARK_LOCK
agent: Cycle 20.D streaming persistence writer agent
cycle: Cycle 20.D streaming persistence writer
replay_key: cycle20d-streaming-persistence-20260606T002816Z
baseline_metrics: cycle15b-pre-shard-baseline-20260603T193531Z
candidate_env_delta: OFFLINE_STREAM_POST_STAGES=1; OFFLINE_STREAM_POST_STAGE_TIMELINE=1 during wrapper only; no embedding-overlap queue; live profile remains disabled
started_at_utc: 2026-06-06T00:28:16Z
expected_cleanup: restore OFFLINE_STREAM_POST_STAGES=0 and OFFLINE_STREAM_POST_STAGE_TIMELINE=0; restart Celery workers; record rollback_status.json, post_stage_wait_snapshot.json, metrics, model agreement, runtime probe, and figures
```

```text
BENCHMARK_RELEASE
agent: Cycle 20.D streaming persistence writer agent
cycle: Cycle 20.D streaming persistence writer
replay_key: cycle20d-streaming-persistence-20260606T002816Z
job_id: 40b9ee20-1179-4363-990f-621ea9d78a8d
status: NEEDS_ITERATION_NO_DECISION
metrics_json: /home/bamby/grad_project/backend/logs/cycle20d-streaming-persistence-20260606T002816Z/post_stage_timeline_metrics.json
model_agreement_json: /home/bamby/grad_project/backend/logs/cycle20d-streaming-persistence-20260606T002816Z/model_agreement_baseline_vs_post_stage_timeline.json
wait_snapshot_json: /home/bamby/grad_project/backend/logs/cycle20d-streaming-persistence-20260606T002816Z/post_stage_wait_snapshot.json
rollback_json: /home/bamby/grad_project/backend/logs/cycle20d-streaming-persistence-20260606T002816Z/rollback_status.json
figure_manifest: /home/bamby/grad_project/backend/logs/cycle20d-streaming-persistence-20260606T002816Z/figures/figure_manifest.json
released_at_utc: 2026-06-06T00:50:32Z
notes: Replay completed and rollback restored both Cycle 20 flags to 0, but the candidate is not decision-valid: metadata still carried a stale unavailable reason for first_persist_packet_ready_at, and Step 3 reconciled 3633/4541 packets because interpolation mutated prior frame packets after the first callback.
```

```text
BENCHMARK_LOCK
agent: Cycle 20.D streaming persistence writer agent
cycle: Cycle 20.D streaming persistence writer r2
replay_key: cycle20d-streaming-persistence-r2-20260606T005032Z
baseline_metrics: cycle15b-pre-shard-baseline-20260603T193531Z
candidate_env_delta: OFFLINE_STREAM_POST_STAGES=1; OFFLINE_STREAM_POST_STAGE_TIMELINE=1 during wrapper only; revision callback re-persists interpolation-mutated packets; no embedding-overlap queue; live profile remains disabled
started_at_utc: 2026-06-06T00:50:32Z
expected_cleanup: restore OFFLINE_STREAM_POST_STAGES=0 and OFFLINE_STREAM_POST_STAGE_TIMELINE=0; restart Celery workers; record rollback_status.json, post_stage_wait_snapshot.json, metrics, model agreement, runtime probe, and figures
```

```text
BENCHMARK_RELEASE
agent: Cycle 20.D streaming persistence writer agent
cycle: Cycle 20.D streaming persistence writer r2
replay_key: cycle20d-streaming-persistence-r2-20260606T005032Z
job_id: b44945e1-b6a1-48b1-99cd-3af74e7f817d
status: NEEDS_ITERATION_NO_DECISION
metrics_json: /home/bamby/grad_project/backend/logs/cycle20d-streaming-persistence-r2-20260606T005032Z/post_stage_timeline_metrics.json
model_agreement_json: /home/bamby/grad_project/backend/logs/cycle20d-streaming-persistence-r2-20260606T005032Z/model_agreement_baseline_vs_post_stage_timeline.json
wait_snapshot_json: /home/bamby/grad_project/backend/logs/cycle20d-streaming-persistence-r2-20260606T005032Z/post_stage_wait_snapshot.json
rollback_json: /home/bamby/grad_project/backend/logs/cycle20d-streaming-persistence-r2-20260606T005032Z/rollback_status.json
figure_manifest: /home/bamby/grad_project/backend/logs/cycle20d-streaming-persistence-r2-20260606T005032Z/figures/figure_manifest.json
released_at_utc: 2026-06-06T01:10:56Z
notes: Replay completed and rollback restored both Cycle 20 flags to 0. r2 fixed stale unavailable metadata and reached step3_reconciled_packet_count=0, but it is not decision-valid because person_detection model-agreement F1 fell to 4.383676% after stream-time track IDs diverged from the final global tracking assignment.
```

```text
BENCHMARK_LOCK
agent: Cycle 20.D streaming persistence writer agent
cycle: Cycle 20.D streaming persistence writer r3
replay_key: cycle20d-streaming-persistence-r3-20260606T011056Z
baseline_metrics: cycle15b-pre-shard-baseline-20260603T193531Z
candidate_env_delta: OFFLINE_STREAM_POST_STAGES=1; OFFLINE_STREAM_POST_STAGE_TIMELINE=1 during wrapper only; Step 3 compares packet signatures and rewrites pre-embedding packets whose final tracking IDs/boxes differ; no embedding-overlap queue; live profile remains disabled
started_at_utc: 2026-06-06T01:10:56Z
expected_cleanup: restore OFFLINE_STREAM_POST_STAGES=0 and OFFLINE_STREAM_POST_STAGE_TIMELINE=0; restart Celery workers; record rollback_status.json, post_stage_wait_snapshot.json, metrics, model agreement, runtime probe, and figures
```

```text
BENCHMARK_RELEASE
agent: Cycle 20.D streaming persistence writer agent
cycle: Cycle 20.D streaming persistence writer r3
replay_key: cycle20d-streaming-persistence-r3-20260606T011056Z
job_id: 24e9970f-b3bc-451d-ab50-b0bcbb1e8d8b
status: NOT_ACCEPTED
metrics_json: /home/bamby/grad_project/backend/logs/cycle20d-streaming-persistence-r3-20260606T011056Z/post_stage_timeline_metrics.json
metrics_md: /home/bamby/grad_project/backend/logs/cycle20d-streaming-persistence-r3-20260606T011056Z/post_stage_timeline_metrics.md
model_agreement_json: /home/bamby/grad_project/backend/logs/cycle20d-streaming-persistence-r3-20260606T011056Z/model_agreement_baseline_vs_post_stage_timeline.json
model_agreement_md: /home/bamby/grad_project/backend/logs/cycle20d-streaming-persistence-r3-20260606T011056Z/model_agreement_baseline_vs_post_stage_timeline.md
wait_snapshot_json: /home/bamby/grad_project/backend/logs/cycle20d-streaming-persistence-r3-20260606T011056Z/post_stage_wait_snapshot.json
rollback_json: /home/bamby/grad_project/backend/logs/cycle20d-streaming-persistence-r3-20260606T011056Z/rollback_status.json
figure_manifest: /home/bamby/grad_project/backend/logs/cycle20d-streaming-persistence-r3-20260606T011056Z/figures/figure_manifest.json
released_at_utc: 2026-06-06T01:29:31Z
notes: Replay completed and rollback restored both Cycle 20 flags to 0. r3 restored row parity and all four model-agreement F1 values to 100.000%, but DB-completed FPS regressed 14.30%, Step 2 frame wall regressed 16.18%, GPU average utilization regressed 14.93%, embedding still did not overlap inference, and Step 3 reconciled 4449/4541 packets before embeddings.
```

### Phase D production result (2026-06-06)

Cycle 20.D is **NOT ACCEPTED**. The r3 candidate proved that per-frame
persistence packets can be written before frame inference drains, but those
writes did not become useful overlap: final tracking assignment and revised
packet signatures forced Step 3 to rewrite nearly every frame packet before
embedding generation.

| Metric | Baseline | Candidate r3 | Result |
|---|---:|---:|---|
| DB-completed FPS | `5.619787` | `4.816369` | `-14.30 %`; fails throughput gate |
| DB completed elapsed | `808.038 s` | `942.826 s` | `+16.68 %`; fails total-wall gate |
| Step 2 frame wall | `467.449833 s` | `543.095716 s` | `+16.18 %`; fails inference-protection gate |
| Step 2 through-pose wall | `641.154064 s` | `776.076979 s` | `+21.04 %`; fails inference-protection gate |
| Behavior RTT mean | `83.530 ms` | `87.057 ms` | `+4.22 %`; regressed |
| GPU avg util | `11.846 %` | `10.077 %` | `-14.93 %`; fails utilization gate |
| GPU peak util | `57.000 %` | `53.000 %` | `-7.02 %` |
| Detection/BBox rows | `72744` / `72744` | `72744` / `72744` | row parity restored |
| Embedding rows | `72578` | `72578` | row parity restored |
| StudentTracks | `53` | `53` | parity restored |
| Per-model F1@IoU0.5 | `100.000 %` | `100.000 %` | correctness proxy passed |
| First packet before inference done | `unavailable` | `true` | overlap marker recorded |
| Embedding starts before inference done | `unavailable` | `false` | embedding stayed serial |
| Streaming persisted packets | `unavailable` | `8174` | initial plus revised packets |
| Step 3 reconciled packets | `unavailable` | `4449` | fails Step 3 elimination gate |
| Rollback verified | required | `true` | both Cycle 20 flags restored to `0` |

Figure manifest:
`docs/figures/benchmark_artifacts/cycle20d-streaming-persistence-r3-20260606T011056Z/figure_manifest.json`.

![Cycle 20.D decision delta](figures/benchmark_artifacts/cycle20d-streaming-persistence-r3-20260606T011056Z/cycle20d_streaming_persistence__decision_delta.png)

![Cycle 20.D packet readiness](figures/benchmark_artifacts/cycle20d-streaming-persistence-r3-20260606T011056Z/cycle20d_streaming_persistence__packet_readiness.png)

![Cycle 20.D packet budget](figures/benchmark_artifacts/cycle20d-streaming-persistence-r3-20260606T011056Z/cycle20d_streaming_persistence__packet_budget.png)

![Cycle 20.D correctness gate](figures/benchmark_artifacts/cycle20d-streaming-persistence-r3-20260606T011056Z/cycle20d_streaming_persistence__correctness_gate.png)

The decision consequence is to keep `OFFLINE_STREAM_POST_STAGES=0` and stop
this writer profile. Cycle 21 worker-count scaling should not start from this
candidate because the run did not create beneficial independent post-stage
work. A future Cycle 20 follow-up needs a different design that persists only
final-tracking-stable packets, or a separate measured bottleneck that justifies
a fresh benchmark lock.

### 2026-06-06 Cycle 20.E Root-Cause Repair Benchmark

Status: `PRODUCTION_BENCHMARK_COMPLETE / BENCHMARK_LOCK_RELEASED /
NOT_ACCEPTED`.

Cycle 20.E addressed the two Cycle 20.D r3 root causes without rerunning the
rejected callback writer profile. `backend/apps/video_analysis/tasks.py`
supports `OFFLINE_STREAM_POST_STAGE_MODE=final_stable_overlap` behind the
existing default-off `OFFLINE_STREAM_POST_STAGES=1` gate. In that mode, Step 2
callbacks record packet readiness only; they do not issue authoritative
PostgreSQL writes. After final `_assign_tracking_ids(...)` and any shard
authoritative-frame filter complete, a bounded one-thread background writer
persists cloned final-stable packets while the pose tail can continue. Step 3
then verifies packet signatures against those final-stable rows.

The production drift blocking the fresh benchmark was fixed before replay:
generated base behavior model configs were rebuilt for the accepted
`TRITON_CROP_BEHAVIOR_INPUT_SIZE=320` Top-K route, and
`backend/scripts/build_tensorrt_engines.py` now writes matching
`configs/offline.pbtxt` files so `prod_triton_endpoint_policy.sh --profile
offline` validates the same 320 configs that Triton serves.

```text
BENCHMARK_LOCK
agent: Cycle 20.E final-stable persistence agent
cycle: Cycle 20.E final-stable persistence overlap
replay_key: cycle20e-final-stable-overlap-20260606T092512Z
baseline_metrics: cycle15b-pre-shard-baseline-20260603T193531Z
candidate_env_delta: OFFLINE_STREAM_POST_STAGES=1; OFFLINE_STREAM_POST_STAGE_TIMELINE=1; OFFLINE_STREAM_POST_STAGE_MODE=final_stable_overlap during wrapper only; live profile remains disabled
started_at_utc: 2026-06-06T09:25:12Z
expected_cleanup: restore OFFLINE_STREAM_POST_STAGES=0, OFFLINE_STREAM_POST_STAGE_TIMELINE=0, and OFFLINE_STREAM_POST_STAGE_MODE=inline_db; restart Celery workers; record rollback_status.json, post_stage_wait_snapshot.json, metrics, model agreement, runtime probe, and figures
```

```text
BENCHMARK_RELEASE
agent: Cycle 20.E final-stable persistence agent
cycle: Cycle 20.E final-stable persistence overlap
replay_key: cycle20e-final-stable-overlap-20260606T092512Z
job_id: 211f96fa-7cfc-4d57-812e-9573906f41c5
status: NOT_ACCEPTED
metrics_json: /home/bamby/grad_project/backend/logs/cycle20e-final-stable-overlap-20260606T092512Z/post_stage_timeline_metrics.json
metrics_md: /home/bamby/grad_project/backend/logs/cycle20e-final-stable-overlap-20260606T092512Z/post_stage_timeline_metrics.md
model_agreement_json: /home/bamby/grad_project/backend/logs/cycle20e-final-stable-overlap-20260606T092512Z/model_agreement_baseline_vs_post_stage_timeline.json
model_agreement_md: /home/bamby/grad_project/backend/logs/cycle20e-final-stable-overlap-20260606T092512Z/model_agreement_baseline_vs_post_stage_timeline.md
wait_snapshot_json: /home/bamby/grad_project/backend/logs/cycle20e-final-stable-overlap-20260606T092512Z/post_stage_wait_snapshot.json
rollback_json: /home/bamby/grad_project/backend/logs/cycle20e-final-stable-overlap-20260606T092512Z/rollback_status.json
figure_manifest: /home/bamby/grad_project/backend/logs/cycle20e-final-stable-overlap-20260606T092512Z/figures/figure_manifest.json
released_at_utc: 2026-06-06T09:42:41Z
notes: Replay completed and rollback restored all Cycle 20 flags to defaults. Final-stable packet persistence and Step 3 reconciliation root causes were repaired, but DB-completed FPS, total elapsed, Step 2 through-pose wall, behavior call rate, and embedding overlap still failed the optimization gate.
```

Cycle 20.E is **NOT ACCEPTED**. It proves the root-cause repair, not a
production optimization:

| Metric | Baseline | Candidate | Result |
|---|---:|---:|---|
| DB-completed FPS | `5.619787` | `5.290508` | `-5.86 %`; fails throughput gate |
| DB completed elapsed | `808.038 s` | `858.330 s` | `+6.22 %`; fails total-wall gate |
| Step 2 frame wall | `467.449833 s` | `460.132193 s` | `-1.57 %`; small improvement |
| Step 2 through-pose wall | `641.154064 s` | `719.937697 s` | `+12.29 %`; fails lifecycle wall gate |
| Behavior RTT mean | `83.530 ms` | `84.019 ms` | `+0.59 %`; slight regression |
| Behavior call rate | `4.451525 calls/s` | `4.190697 calls/s` | `-5.86 %`; regressed |
| GPU avg util | `11.846 %` | `12.224 %` | `+3.19 %`; improved but not decisive |
| GPU peak util | `57.000 %` | `70.000 %` | `+22.81 %` |
| Peak VRAM | `15725 MiB` | `15725 MiB` | unchanged |
| Detection/BBox rows | `72744` / `72744` | `72750` / `72750` | within row-parity tolerance |
| Embedding rows | `72578` | `72584` | within row-parity tolerance |
| StudentTracks | `53` | `53` | parity restored |
| Minimum F1@IoU0.5 | required proxy | `99.766 %` | model-agreement proxy passed |
| Persistence starts before inference done | `unavailable` | `true` | final-stable writer overlapped inference |
| Embedding starts before inference done | `unavailable` | `false` | embedding stayed serial |
| Embedding start lag after inference done | `unavailable` | `37.279 s` | fails embedding-overlap intent |
| Final-stable persisted packets | `unavailable` | `4541/4541` | root-cause repair passed |
| Final-stable failed packets | `unavailable` | `0` | root-cause repair passed |
| Step 3 reconciled packets | `4449/4541` in 20.D r3 | `0/4541` | root-cause repair passed |
| Rollback verified | required | `true` | all Cycle 20 flags restored to defaults |

Figure manifest:
`docs/figures/benchmark_artifacts/cycle20e-final-stable-overlap-20260606T092512Z/figure_manifest.json`.

![Cycle 20.E decision delta](figures/benchmark_artifacts/cycle20e-final-stable-overlap-20260606T092512Z/cycle20e_final_stable_overlap__decision_delta.png)

![Cycle 20.E packet readiness](figures/benchmark_artifacts/cycle20e-final-stable-overlap-20260606T092512Z/cycle20e_final_stable_overlap__packet_readiness.png)

![Cycle 20.E packet budget](figures/benchmark_artifacts/cycle20e-final-stable-overlap-20260606T092512Z/cycle20e_final_stable_overlap__packet_budget.png)

![Cycle 20.E correctness gate](figures/benchmark_artifacts/cycle20e-final-stable-overlap-20260606T092512Z/cycle20e_final_stable_overlap__correctness_gate.png)

The production consequence is to keep `OFFLINE_STREAM_POST_STAGES=0`,
`OFFLINE_STREAM_POST_STAGE_TIMELINE=0`, and
`OFFLINE_STREAM_POST_STAGE_MODE=inline_db`. Do not promote Cycle 21 from this
result; the benchmark still did not create useful independent embedding work
for extra workers to consume.

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

Cycle 20.E is now closed as **NOT ACCEPTED**. It fixed the Cycle 20.D packet
signature and Step 3 reconciliation root causes, but it did not create
beneficial independent embedding work and did not improve total throughput.
Cycle 21 concurrency scaling must not be promoted from this result. Any future
post-stage overlap work needs a fresh benchmark lock and a different mechanism
that starts embedding earlier or exposes a separately measured queue bottleneck.
