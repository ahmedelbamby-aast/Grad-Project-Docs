# Tasks: Runtime Stability Remediation Plan

**Input:** [docs/runtime_stability_remediation/plan.md](plan.md)

**Status:** COMPLETED 2026-05-29 — all tasks done, merged to master (SHA 375cc84)

**Branch target:** `release/prod-runtime-stabilization`

**Execution rule:** Local Windows/PostgreSQL validation proves code behavior and
contracts only. Lifecycle, GPU, and final acceptance evidence MUST come from the
native Linux RTX 5090 production server with `backend/.env`, PostgreSQL, Redis,
Celery, and Triton-only inference, tied to a committed Git SHA. SQLite is invalid
in every path.

**Convention:** Task IDs use the `RS` prefix (Runtime Stability). `[P]` marks
tasks that may run in parallel. Tasks that resolve a controlling-plan task name
that plan's ID in brackets, e.g. `(unblocks T038)`.

## Phase R0: Baseline lock (regression-proof the `441c28e` fixes)

- [x] RS001 [P] Add a test asserting `_coerce_embedding_dimension` returns exactly 768 values for under-size, exact, and over-size inputs in `backend/tests/unit/tracking/test_embeddings_compatibility.py`
- [x] RS002 [P] Add a test asserting `generate_embeddings` skips detections that already have an embedding (no duplicate write) in `backend/tests/unit/video_analysis/test_tasks_embeddings.py`
- [x] RS003 [P] Confirm existing `reconcile_runtime_workflows` tests cover stale `processing` and `embedding` finalization and fresh-job preservation in `backend/tests/unit/video_analysis/test_management_commands.py`

## Phase R1: Vector integrity at the write boundary (ISSUE-2, §17.2)

- [x] RS004 Enforce the dimension/value contract inside `persist_embedding` before commit (coerce-then-validate, or call `full_clean()`), so `FrameEmbedding.objects.create()` can never write a non-768 vector, in `backend/apps/tracking/embeddings.py`
- [x] RS005 Add a maximum payload-size guard (reject/coerce) before DB persistence and before the Redis cache write in `backend/apps/tracking/embeddings.py` and the embedding cache path
- [x] RS006 [P] Add a boundary regression test proving a 98,304-value vector and a wrong-dimension vector are rejected or deterministically coerced at `persist_embedding`, not silently written, in `backend/tests/unit/tracking/test_embedding_vector_contract.py`
- [x] RS007 [P] Add a test proving the Redis embedding cache write never stores a payload above the declared size guard in `backend/tests/unit/tracking/test_embedding_cache_payload.py`

## Phase R2: Bounded lifecycle + automatic reconciler (ISSUE-1, ISSUE-5, §17.1)

- [x] RS008 Add a declared per-stage wall-clock deadline (soft time limit) for the processing and embedding tasks and record it in task config in `backend/apps/video_analysis/tasks.py` and `backend/config/celery.py`
- [x] RS009 Register the stale-job reconciler in Celery `beat_schedule` so production finalizes stale `processing`/`embedding` jobs automatically, in `backend/config/celery.py` (wrap `reconcile_runtime_workflows` as a periodic task if a callable task wrapper is required)
- [x] RS010 Ensure a job that exceeds its stage deadline is forced to a terminal `FAILED`/`completed_partial` status with a recorded reason and reconciliation timestamp in `backend/apps/video_analysis/tasks.py`
- [x] RS011 [P] Add a test proving the reconciler is present in `beat_schedule` with a bounded interval in `backend/tests/unit/config/test_celery_beat_schedule.py`
- [x] RS012 [P] Add a test proving a job past its deadline is forced terminal (no indefinite non-terminal state) in `backend/tests/unit/video_analysis/test_job_lifecycle_deadline.py`

## Phase R3: Stage outcome accounting + fail-closed (ISSUE-3, §17.3)

- [x] RS013 Make `generate_embeddings` fail closed when the per-stage error ratio exceeds a declared threshold or when zero required outputs were produced; persist created/skipped/error counts as the authoritative stage outcome in `backend/apps/video_analysis/tasks.py`
- [x] RS014 [P] Add tests for all-fail, partial-fail (below/above threshold), and zero-output conditions in `backend/tests/unit/video_analysis/test_embedding_stage_failclosed.py`

## Phase R4: Idempotency generalization (ISSUE-4, §17.4)

- [x] RS015 Document the idempotency key for each durable-write stage and confirm existence-guarded writes for embeddings (and any sibling stage that writes per-detection rows) in `backend/apps/video_analysis/tasks.py`
- [x] RS016 [P] Add a re-run test proving a second `generate_embeddings` invocation for a completed job creates zero duplicate rows in `backend/tests/unit/video_analysis/test_embedding_idempotency.py`

## Phase R5: Local verification

- [x] RS017 Run the focused backend suite (vector contract, lifecycle deadline, fail-closed, idempotency, reconciler) on PostgreSQL and save output to `ci_evidence/production/runtime_maturity/phase3_runtime_stability_local_tests.md`
- [x] RS018 Run the existing env/strategy/runtime-mode/replay suites to confirm no regression and append results to the same evidence file

## Phase R6: Production lifecycle re-validation (unblocks controlling plan)

- [x] RS019 Commit the remediation on `release/prod-runtime-stabilization`, verify local/origin/production hash parity via `tools/prod/prod-hash-parity.ps1`
- [x] RS020 Sync the fix SHA to production and run `reconcile_runtime_workflows` to finalize the stuck `c375ea84` lineage; save evidence to `ci_evidence/production/runtime_maturity/phase3_stuck_job_reconciliation.md`
- [x] RS021 Run `tools/prod/prod-runtime-preflight.sh` and confirm it passes for the active endpoint; refresh `ci_evidence/production/runtime_maturity/phase2_prod_preflight.md`
- [x] RS022 Run a fresh real-video lifecycle job to terminal `completed` (not a timeout) and overwrite `ci_evidence/production/runtime_maturity/phase3_fresh_lifecycle_job.md` with the accepted job ID and terminal status **(unblocks T038)**

## Phase R7: Maturity closure handoff (controlling plan)

- [x] RS023 Capture production GPU telemetry during the accepted job under `ci_evidence/production/runtime_maturity/phase4_gpu_trace/` **(unblocks T046)**
- [x] RS024 Export causality for the accepted job under `ci_evidence/production/runtime_maturity/phase4_causality/` **(unblocks T047)**
- [x] RS025 Generate the evidence manifest for the accepted job at `ci_evidence/production/runtime_maturity/phase4_evidence_manifest.json` **(unblocks T048)**
- [x] RS026 Resume controlling-plan closure tasks T051–T055 and T060 with the accepted job

## Phase R8: Governance propagation (§17 closure)

- [x] RS027 [P] Propagate the Section 17 gates into Spec Kit templates (`.specify/templates/plan-template.md`, `spec-template.md`, `tasks-template.md`)
- [x] RS028 [P] Reference this remediation plan and Section 17 from `AGENTS.md` and the controlling maturity plan
- [x] RS029 Record a debt/deferral entry for any Section 17 gate not yet enforced in production, with owner and expiry, in the controlling-plan evidence

## Dependencies
- R0 baseline tests can run immediately and in parallel.
- R1–R4 are independent code-invariant phases; their `[P]` tests can run in parallel once each invariant lands.
- R5 (local verification) requires R1–R4 code complete.
- R6 (production) requires R5 green and committed.
- R7 requires R6's accepted terminal job.
- R8 can run in parallel with R6/R7 once Section 17 is ratified.

## Validation commands
- Vector + lifecycle + fail-closed + idempotency focused suite:
  - `.\.venv\Scripts\python.exe -m pytest backend/tests/unit/tracking/test_embedding_vector_contract.py backend/tests/unit/video_analysis/test_job_lifecycle_deadline.py backend/tests/unit/video_analysis/test_embedding_stage_failclosed.py backend/tests/unit/video_analysis/test_embedding_idempotency.py backend/tests/unit/config/test_celery_beat_schedule.py -q --tb=short`
- Existing runtime authority suites:
  - `.\.venv\Scripts\python.exe -m pytest backend/tests/unit/pipeline/test_runtime_env_authority.py backend/tests/unit/pipeline/test_production_inference_strategy.py backend/tests/contract/test_runtime_mode_contract.py -q --tb=short`
- Production lifecycle:
  - `.\tools\prod\prod-runtime-ingest-video.ps1`
- Stuck-job reconciliation (production):
  - `.\tools\prod\prod-ssh.ps1 -Cmd "cd /home/bamby/grad_project && backend/.venv/bin/python backend/manage.py reconcile_runtime_workflows --stale-minutes 30"`
