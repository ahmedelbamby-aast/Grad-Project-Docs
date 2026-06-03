# Cycle 15.B1 Two-Shard Runtime Investigation

**Last updated:** 2026-06-03

**Status:** RUNTIME CANDIDATE IMPLEMENTED LOCALLY / PRODUCTION BENCHMARK
PENDING. No two-shard runtime candidate has been accepted, rejected, skipped,
or closed; only a full production Linux RTX 5090 `combined.mp4` benchmark can
make that decision.

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
| Single lifecycle job | `runtime_ingest_video.py` creates one `VideoAnalysisJob` for a replay. | The new Cycle 15.B1 command creates a parent job plus two child shard jobs; normal ingest remains single-job. |
| Frame-window runtime | `_run_triton_frame_level_inference(..., decode_frame_window=...)` can decode a 0-based half-open window. | Only valid offline child shard metadata uses the window; normal full-video path remains unchanged. |
| Direct Step 3 persistence | `tasks.py` filters `frame_detections` through `filter_authoritative_frame_detections(...)` before Step 3 for child shards. | Context-only frames feed tracking/boundary evidence but are not authoritative persisted rows. |
| Frame idempotency exists | `Frame` has `uq_frame_job_frame_number`. | Frame upsert can protect parent rows. |
| Track idempotency exists | `StudentTrack` has `uq_student_track_job_tracking_id`. | Tracks are unique inside one job, but shard-local IDs still need canonicalization. |
| Detection/BBox provenance | `Detection.shard_provenance` and `BoundingBox.shard_provenance` exist in migration `0014_cycle15b1_shard_provenance.py`. | Parent merge can replace authoritative parent rows idempotently and preserve source lineage. |
| Embedding provenance | `FrameEmbedding.embedding_provenance` exists in migration `0014_cycle15b1_shard_provenance.py`. | Child embeddings can be copied with provenance; current runtime generates parent embeddings after merge. |
| Fail-closed guard | `process_video_upload` rejects malformed or disabled shard metadata with `cycle15b1_sharding_runtime_invalid_request`. | Accidental or partial enablement still fails closed before inference. |

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

Production deployment and readiness audit:

| Field | Value |
|---|---|
| Deployed SHA | `74631e6` |
| Evidence directory | `/home/bamby/grad_project/backend/logs/cycle15b1-runtime-readiness-safe-default-20260603T202308Z` |
| JSON | `/home/bamby/grad_project/backend/logs/cycle15b1-runtime-readiness-safe-default-20260603T202308Z/readiness.json` |
| Markdown | `/home/bamby/grad_project/backend/logs/cycle15b1-runtime-readiness-safe-default-20260603T202308Z/readiness.md` |
| Overall status | `blocked_no_runtime_candidate` |
| Ready for runtime benchmark | `False` |
| Critical blockers | `5` |
| Warnings | `0` |
| Production unit validation | `5 passed` for `test_cycle15b1_sharding_guard.py` plus `test_prod_plan_video_shards.py`. |
| Production env proof | `OFFLINE_VIDEO_SHARDING_ENABLED=0`, `OFFLINE_VIDEO_SHARD_COUNT=1`, `OFFLINE_VIDEO_SHARD_CONTEXT_FRAMES=32`. |
| Accepted profile preserved | `MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble_gaze_slice_topk`, `TRITON_BEHAVIOR_TOP_K_ENABLED=1`, `TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=1`. |
| Worker state | Celery workers and beat restarted; active-task inspect returned empty queues. |

Production Mermaid validation note: the date/reading-order verifier passed on
production, but Mermaid rendering could not run because `mmdc` is not installed
on the production host. The same changed-doc Mermaid set passed locally with
`5 / 5` rendered blocks before deployment.

Remaining blockers:

| Blocker | Why it still blocks benchmark |
|---|---|
| `runtime_candidate_active` | The fail-closed guard is still active; no sharded runtime candidate exists. |
| `runtime_benchmark_wrapper` | No reproducible two-shard production benchmark wrapper exists. |
| `parent_merge_helper` | No authoritative parent merge/finalization helper exists. |
| `bbox_detection_merge_idempotency` | Detection and bounding-box rows still lack shard provenance or a natural retry key. |
| `embedding_merge_idempotency` | FrameEmbedding rows still lack shard provenance or a natural retry key. |

## Implementation Slice 2: Runtime Candidate and Merge Path

This slice implements the minimum benchmarkable Cycle 15.B1 runtime candidate.
It does not accept, reject, skip, or close the cycle because production
benchmark evidence is still pending.

| Change | File | Evidence |
|---|---|---|
| Offline shard service | `backend/apps/video_analysis/services/offline_sharding.py` | Defines `OfflineVideoShardRequest`, `ShardRange`, pre-roll-only `build_even_shards`, authoritative filtering, boundary summaries, and idempotent parent merge. |
| Decode window | `backend/apps/video_analysis/tasks.py` | `_run_triton_frame_level_inference(..., decode_frame_window=...)` decodes only a 0-based half-open window for valid child shard jobs. |
| Triton-only guard | `backend/apps/video_analysis/tasks.py` + `tools/prod/prod_run_cycle15b1_two_shard_runtime_benchmark.sh` | Valid child shards fail unless the offline runtime is the Triton-only path; the wrapper refuses non-`triton_only` settings before submit. |
| Valid shard request path | `backend/apps/video_analysis/tasks.py` | Valid metadata enters `cycle15b1_sharding_runtime_active`; malformed/disabled metadata fails with `cycle15b1_sharding_runtime_invalid_request`. |
| Child authoritative persistence | `backend/apps/video_analysis/tasks.py` | Child jobs filter context frames before Step 3, persist only authoritative rows, write an inference audit, and stop before render/embedding. |
| Parent orchestrator | `backend/apps/video_analysis/management/commands/cycle15b1_sharded_ingest.py` | Creates a parent job, creates two child jobs, queues child Celery tasks, waits for terminal children, merges rows, then runs parent embeddings inline. |
| Parent merge helper | `tools/prod/prod_merge_cycle15b1_shards.py` | Re-runs the same idempotent merge independently and can optionally run parent embeddings. |
| Benchmark wrapper | `tools/prod/prod_run_cycle15b1_two_shard_runtime_benchmark.sh` | Enables sharding only for the benchmark window, restarts workers, captures GPU CSV, runs the orchestrator, collects metrics, and restores safe defaults on exit. |
| DB provenance | `backend/apps/video_analysis/models.py` + `backend/apps/video_analysis/migrations/0014_cycle15b1_shard_provenance.py` | Adds `Detection.shard_provenance`, `BoundingBox.shard_provenance`, and `FrameEmbedding.embedding_provenance`. |
| CI coverage | `.github/workflows/inference-parallelization.yml` | Adds shell/compile checks for new helpers and runs the Cycle 15.B1 guard/merge tests. |

Local validation:

| Check | Result |
|---|---|
| Python compile | `python -m py_compile backend/apps/video_analysis/services/offline_sharding.py backend/apps/video_analysis/tasks.py backend/apps/video_analysis/management/commands/cycle15b1_sharded_ingest.py tools/prod/prod_merge_cycle15b1_shards.py tools/prod/prod_check_cycle15b1_runtime_readiness.py` passed. |
| Shell syntax | `bash -n tools/prod/prod_run_cycle15b1_two_shard_runtime_benchmark.sh` passed. |
| Django check | `.venv\Scripts\python.exe manage.py check` passed. |
| Migration drift | `.venv\Scripts\python.exe manage.py makemigrations --check --dry-run` passed with `No changes detected`. |
| Focused unit tests | `.venv\Scripts\python.exe -m pytest tests/unit/video_analysis/test_cycle15b1_sharding_guard.py tests/unit/video_analysis/test_cycle15b1_shard_merge.py -q` passed: `4 passed`. |
| Local readiness audit | `backend/logs/cycle15b1_readiness_local.json` reported `ready_for_runtime_benchmark=True`, `critical_blocker_count=0`, `warning_count=0`. |

Readiness blocker resolution:

| Prior blocker | Current status | Resolution evidence |
|---|---|---|
| `runtime_candidate_active` | `ok` | Runtime markers `cycle15b1_sharding_runtime_active`, `decode_frame_window`, and `filter_authoritative_frame_detections` exist. |
| `runtime_benchmark_wrapper` | `ok` | `tools/prod/prod_run_cycle15b1_two_shard_runtime_benchmark.sh` exists and passes shell syntax. |
| `parent_merge_helper` | `ok` | `tools/prod/prod_merge_cycle15b1_shards.py` exists and compiles. |
| `bbox_detection_merge_idempotency` | `ok` | `shard_provenance` fields exist and `test_cycle15b1_parent_merge_is_idempotent` passes. |
| `embedding_merge_idempotency` | `ok` | `embedding_provenance` exists; parent embeddings are generated after merge for the runtime candidate. |

Production benchmark command:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_run_cycle15b1_two_shard_runtime_benchmark.sh \
  --video "/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4" \
  --baseline-metrics "/home/bamby/grad_project/backend/logs/cycle15b-pre-shard-baseline-20260603T193531Z/metrics.json"
```

Decision state: `NOT DECIDED`. The runtime candidate is only ready to deploy
and benchmark. Acceptance remains impossible until the production wrapper
completes, metrics are collected, DB/model/embedding parity is measured, safe
defaults are restored, and this document plus the benchmark history are updated
with the production replay key and parent/shard job ids.
