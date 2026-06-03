# Cycle 15.B1 Two-Shard Runtime Investigation

**Last updated:** 2026-06-03

**Status:** PHASE A STARTED / RUNTIME BENCHMARK BLOCKED. No two-shard runtime
candidate has been implemented, production-benchmarked, accepted, rejected,
skipped, or closed.

**Streaming compatibility:** `offline-only`. The Cycle 15.B1 runtime candidate
requires whole-file access, frame windows, overlap pre-roll, and parent/shard
terminal coordination. It MUST remain disabled for RTSP, RTSPS, WHEP/WebRTC,
HLS fallback, and every live stream profile.

## Problem Statement

Cycle 15.B design proof showed that a two-shard plan can cover all `4541`
authoritative frames in `combined.mp4` with only `32` context-only frames
(`0.704691 %` overhead). The production pre-shard baseline benchmark then
recorded the accepted single-job runtime comparator:

| Baseline metric | Value |
|---|---:|
| Replay key | `cycle15b-pre-shard-baseline-20260603T193531Z` |
| Job id | `74561b05-105f-4ca8-aeaf-f510f4f802de` |
| DB FPS | `5.620` |
| Step 2 frame wall | `467.450 s` |
| Step 2 frame-loop FPS | `9.714` |
| Step 2 through-pose wall | `641.154 s` |
| Behavior ensemble mean RTT | `83.530 ms` |
| GPU average utilization | `11.846 %` |
| GPU peak utilization | `57.000 %` |
| Detection rows | `72744` |
| Bounding-box rows | `72744` |
| Embedding rows | `72578` |
| Student tracks | `53` |

The next question is not whether sharding can divide frame ranges. That is
already proven. The next question is whether a two-shard runtime can improve
throughput while preserving frame ownership, DB rows, model agreement,
tracking continuity, embeddings, terminal state, and rollback.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Parent sharding investigation | `docs/cycle_15b_video_sharding_design_proof_investigation.md` | Defines the sharding safety contract. |
| Two-shard design proof | `docs/cycle_15b1_two_shard_design_proof_investigation.md` | Defines the lowest-risk two-shard scenario. |
| Design-proof result | `docs/cycle_15b_shard_design_probe_results.md` | Records dry-run ranges and the pre-shard baseline benchmark. |
| Benchmark history | `docs/production_inference_benchmark.md` | Stores the authoritative production comparison table. |
| Shard planner | `tools/prod/prod_plan_video_shards.py` | Computes deterministic shard ownership and pre-roll frames. |
| Runtime readiness helper | `tools/prod/prod_check_cycle15b1_runtime_readiness.py` | Reproducibly reports blockers before a runtime benchmark. |
| Safe env-default helper | `tools/prod/prod_set_cycle15b1_sharding_defaults.sh` | Sets only disabled Cycle 15.B1 env defaults without rewriting accepted inference flags. |
| Offline task code | `backend/apps/video_analysis/tasks.py` | Owns frame inference, tracking assignment, Step 3 persistence, render, and embedding handoff. |
| Job and row models | `backend/apps/video_analysis/models.py` | Owns job, frame, detection, bounding-box, track, and embedding persistence constraints. |
| Ingest command | `backend/apps/video_analysis/management/commands/runtime_ingest_video.py` | Current benchmark entrypoint creates one lifecycle job per replay. |
| Streaming doctrine | `.specify/memory/constitution.md` | Section 8.6 forbids offline-only sharding on live sources. |

## Current Code Evidence

| Area | Evidence | Runtime implication |
|---|---|---|
| Single lifecycle job | `runtime_ingest_video.py` creates one `VideoAnalysisJob` for a replay. | A parent/shard orchestration layer does not exist yet. |
| Full-file frame loop | `tasks.py` opens the job video and returns one `frame_detections` map for the job. | No authoritative frame-window parameter exists yet. |
| Direct Step 3 persistence | `tasks.py` persists every `frame_detections` item into the same job. | Context-only frames would be persisted unless filtered before Step 3. |
| Frame idempotency exists | `Frame` has `uq_frame_job_frame_number`. | Frame upsert can protect parent rows. |
| Track idempotency exists | `StudentTrack` has `uq_student_track_job_tracking_id`. | Tracks are unique inside one job, but shard-local IDs still need canonicalization. |
| Detection/BBox merge key missing | `Detection` and `BoundingBox` have no shard provenance or natural merge key. | Parent merge cannot yet be idempotently retried without delete-and-replace or new provenance. |
| Embedding merge key missing | `FrameEmbedding` has indexes but no uniqueness/provenance. | Parent embedding merge cannot yet prove retry safety. |
| Fail-closed guard | `process_video_upload` rejects `OFFLINE_VIDEO_SHARDING_ENABLED=1` or shard metadata with `cycle15b1_sharding_runtime_not_implemented`. | Accidental enablement cannot corrupt persistence while the runtime candidate is still absent. |

## Runtime Contract Before Code

| Contract item | Required rule |
|---|---|
| Env gate | `OFFLINE_VIDEO_SHARDING_ENABLED=0` by default; live profile explicitly sets it to `0`. |
| Parent job | Owns replay key, benchmark evidence, terminal status, and aggregate metrics. |
| Shard jobs | Carry `parent_shard_job_id`, `shard_index`, decode range, authoritative range, and context-frame count in metadata. |
| Context frames | Decode and feed tracking only; they must not write authoritative `Frame`, `Detection`, `BoundingBox`, or embedding rows. |
| Original frame numbers | Every persisted row keeps original video frame numbering, not shard-local numbering. |
| Track stitching | Boundary tracks merge only with explicit overlap evidence; unresolved tracks fail the acceptance gate instead of silently merging. |
| Persistence idempotency | Parent merge must be safely retryable for frames, detections, bounding boxes, tracks, embeddings, and artifacts. |
| Terminal state | Parent cannot complete until every shard is terminal and merge succeeds; any failed shard fails the parent. |
| Rollback | One command restores accepted single-job profile and leaves sharding flags disabled. |
| Benchmark evidence | Parent metrics, every shard job id, GPU CSV, model RTT, DB parity, embedding parity, and rollback proof are recorded. |

## Candidate Split

| Sub-candidate | Scope | Decision rule |
|---|---|---|
| 15.B1.R0 readiness audit | Read-only checker for blockers and safe defaults. | Can run immediately; no runtime acceptance. |
| 15.B1.R1 same-file frame-window runtime | Child jobs read the original file with decode/authoritative frame windows. | First implementable candidate because original frame numbers can be preserved. |
| 15.B1.R2 physical clip shards | Child jobs run on clipped shard files. | Benchmark only if R1 fails; frame-number/timestamp offset risk must be solved first. |

## Phase A Hypothesis

Two-shard runtime may improve Step 2 throughput by running independent frame
ranges in parallel, but the pre-shard baseline already shows that the frame loop
is not the whole job. The future candidate must also account for `173.704 s`
of pose-tail over-frame-loop wall and `98.578 s` of embedding-created span.
If those post-frame stages remain serialized after shard merge, DB FPS may
improve less than Step 2 frame wall.

## Risk Assessment

| Risk | Severity | Control |
|---|---|---|
| Duplicate authoritative rows | High | Context frames are filtered before persistence; parent merge is idempotent. |
| Track identity split at boundary | High | Boundary overlap evidence must drive canonical IDs; unresolved merges fail acceptance. |
| Parent completes early | High | Parent terminal state waits for every shard plus merge proof. |
| Live-stream collapse | High | Sharding flags remain offline-only and explicitly disabled in live profile. |
| False performance win | Medium | Compare against pre-shard baseline with DB/model/embedding parity gates. |
| GPU contention | Medium | Benchmark captures GPU avg/peak, VRAM, model RTT, and worker RSS. |

## Rollback Strategy

Runtime rollback will be valid only when a future implementation provides a
script that:

| Rollback item | Required behavior |
|---|---|
| Env | Sets `OFFLINE_VIDEO_SHARDING_ENABLED=0` and restores accepted Cycle 14.B2/15 baseline flags. |
| Services | Restarts workers without starting shard jobs. |
| DB | Leaves failed candidate jobs terminal and does not mutate accepted baseline evidence. |
| Validation | Runs a single-job `combined.mp4` smoke or benchmark selector and verifies current profile. |

## Acceptance Criteria

No 15.B1 runtime candidate can be accepted until all gates pass:

| Gate | Requirement |
|---|---|
| Production benchmark | Full Linux RTX 5090 `combined.mp4` benchmark with sharding enabled. |
| Performance | Improves DB FPS, Step 2 frame wall, Step 2 through-pose wall, total wall, and GPU utilization versus `cycle15b-pre-shard-baseline-20260603T193531Z`. |
| Correctness | DB row parity, model agreement, embedding parity, and StudentTrack continuity pass. |
| Lifecycle | Parent and every shard job reach terminal status; stuck shard is a failure. |
| Evidence | Metrics JSON/Markdown, parent job id, shard job ids, replay key, GPU CSV, and rollback proof are recorded. |
| Streaming safety | Live profile remains disabled and no RTSP/RTSPS path can enable sharding. |

## Phase A Readiness Result

Production read-only readiness audit:

| Field | Value |
|---|---|
| Evidence directory | `/home/bamby/grad_project/backend/logs/cycle15b1-runtime-readiness-20260603T200611Z` |
| JSON | `/home/bamby/grad_project/backend/logs/cycle15b1-runtime-readiness-20260603T200611Z/readiness.json` |
| Markdown | `/home/bamby/grad_project/backend/logs/cycle15b1-runtime-readiness-20260603T200611Z/readiness.md` |
| Overall status | `blocked_no_runtime_candidate` |
| Ready for runtime benchmark | `False` |
| Critical blockers | `8` |
| Warnings | `0` |

| Check | Status | Evidence |
|---|---|---|
| `env_flag_declared` | `blocked` | `OFFLINE_VIDEO_SHARDING_ENABLED` missing from `backend/config/settings/base.py`. |
| `profile_flag_managed` | `blocked` | `OFFLINE_VIDEO_SHARDING_ENABLED` missing from `tools/prod/prod_enable_parallel_flow.sh`. |
| `task_authoritative_window` | `blocked` | `authoritative_start_frame` marker missing from `backend/apps/video_analysis/tasks.py`. |
| `task_parent_shard_link` | `blocked` | `parent_shard_job_id` marker missing from `backend/apps/video_analysis/tasks.py`. |
| `runtime_benchmark_wrapper` | `blocked` | `tools/prod/prod_run_cycle15b1_two_shard_runtime_benchmark.sh` missing. |
| `parent_merge_helper` | `blocked` | `tools/prod/prod_merge_cycle15b1_shards.py` missing. |
| `frame_idempotency` | `ok` | `Frame` already has `uq_frame_job_frame_number`. |
| `track_idempotency` | `ok` | `StudentTrack` already has `uq_student_track_job_tracking_id`. |
| `bbox_detection_merge_idempotency` | `blocked` | `shard_provenance` marker missing from `backend/apps/video_analysis/models.py`. |
| `embedding_merge_idempotency` | `blocked` | `embedding_provenance` marker missing from `backend/apps/video_analysis/models.py`. |
| `baseline_metrics_available` | `ok` | Pre-shard baseline metrics JSON exists. |
| `prod_env_safe_default` | `ok` | `OFFLINE_VIDEO_SHARDING_ENABLED` is absent, so no sharding runtime can be enabled accidentally. |

This result is not a rejection, skip, or acceptance. It is the implementation
checklist for the next code slice. A full sharded runtime benchmark remains
blocked until these checks are resolved.

## Implementation Slice 1: Safe Defaults and Fail-Closed Guard

This slice does not implement sharded inference and does not authorize a
benchmark. It only makes accidental enablement safer and makes the remaining
blockers reproducible.

| Change | File | Evidence |
|---|---|---|
| Disabled settings added | `backend/config/settings/base.py` | `OFFLINE_VIDEO_SHARDING_ENABLED=0`, `OFFLINE_VIDEO_SHARD_COUNT=1`, and `OFFLINE_VIDEO_SHARD_CONTEXT_FRAMES=32` exist with bounded parsing. |
| Production profile awareness | `tools/prod/prod_enable_parallel_flow.sh` | The profile script knows the new keys and keeps sharding disabled. |
| Targeted env helper | `tools/prod/prod_set_cycle15b1_sharding_defaults.sh` | Sets only Cycle 15.B1 sharding defaults so accepted Top-K/overlap flags are not rewritten. |
| Fail-closed runtime guard | `backend/apps/video_analysis/tasks.py` | Sharding flag or shard metadata fails with `cycle15b1_sharding_runtime_not_implemented` before video validation. |
| Unit coverage | `backend/tests/unit/video_analysis/test_cycle15b1_sharding_guard.py` | Verifies both env-flag and metadata-triggered guard paths. |

Local validation:

| Check | Result |
|---|---|
| Python compile | `python -m py_compile backend/apps/video_analysis/tasks.py tools/prod/prod_check_cycle15b1_runtime_readiness.py` passed. |
| Shell syntax | `bash -n tools/prod/prod_set_cycle15b1_sharding_defaults.sh` passed. |
| Focused unit tests | `.venv\Scripts\python.exe -m pytest tests/unit/video_analysis/test_cycle15b1_sharding_guard.py -q --tb=short -o addopts=` passed: `2 passed`. |
| Documentation date/reading-order gate | `python scripts/ci/verify_doc_dates_and_reading_order.py` passed: `241` priority docs. |
| Local readiness audit | `blocked_no_runtime_candidate`, `5` critical blockers, `0` warnings. |

Remaining blockers:

| Blocker | Why it still blocks benchmark |
|---|---|
| `runtime_candidate_active` | The fail-closed guard is still active; no sharded runtime candidate exists. |
| `runtime_benchmark_wrapper` | No reproducible two-shard production benchmark wrapper exists. |
| `parent_merge_helper` | No authoritative parent merge/finalization helper exists. |
| `bbox_detection_merge_idempotency` | Detection and bounding-box rows still lack shard provenance or a natural retry key. |
| `embedding_merge_idempotency` | FrameEmbedding rows still lack shard provenance or a natural retry key. |
