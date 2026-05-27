# Tasks: Heterogeneous Production Runtime Maturity Plan

**Input:** [docs/heterogeneous_production_runtime_maturity_plan.md](heterogeneous_production_runtime_maturity_plan.md)

**Status:** Active, not completed

**Branch target:** `release/prod-runtime-stabilization`

**Execution rule:** Local Windows/Docker validation may prove code behavior and contracts only. Production acceptance must be validated on the native Linux RTX 5090 server using `backend/.env`, PostgreSQL, Redis, Celery, Triton-only inference, and evidence tied to a committed Git SHA.

## User Stories

**US1 - Branch and deployment governance:** As a production operator, I need production code, origin, and local worktrees tied to the same reviewed branch and SHA so evidence cannot be detached from source.

**US2 - Runtime contract lock:** As a production operator, I need startup to fail before processing if backend env authority, Triton-only inference, endpoint isolation, PostgreSQL, Redis, Celery, or toolchain pinning are invalid.

**US3 - Replay and lifecycle integrity:** As a benchmark operator, I need explicit replay policies so failed lineage is never accepted as production evidence and fresh jobs can be linked to prior attempts.

**US4 - GPU and causality certification:** As an evaluator, I need production evidence for the exact job ID proving RTX 5090 execution, Triton model calls, endpoint isolation, queue stability, batching behavior, and causality export.

**US5 - Closure and reproducibility:** As a maintainer, I need packaged evidence, merged release changes, and a clean production tree so the runtime remains reproducible after sign-off.

## Phase 1: Setup

- [x] T001 Create `release/prod-runtime-stabilization` locally from the current active branch and record the starting SHA in `ci_evidence/production/runtime_maturity/startup_branch_state.md`
- [x] T002 [P] Add a production evidence directory convention for this plan in `ci_evidence/production/runtime_maturity/README.md`
- [x] T003 [P] Add a short operator checklist for this plan in `docs/production/runtime_maturity_operator_checklist.md`
- [x] T004 Update production helper documentation to name `release/prod-runtime-stabilization` as the target stabilization branch in `docs/heterogeneous_production_runtime_maturity_plan.md`
- [x] T005 [P] Add the local contract workflow for this plan in `.github/workflows/prod-runtime-maturity.yml`
- [x] T006 [P] Add a production evidence orchestrator in `tools/prod/prod-runtime-maturity-evidence.ps1`
- [x] T007 [P] Add evidence templates in `ci_evidence/production/runtime_maturity/acceptance_matrix.md` and `ci_evidence/production/runtime_maturity/manifest.template.json`

## Phase 2: Foundational

- [x] T008 Audit current runtime mode, inference strategy, env loading, replay, Celery routing, and production helper scripts; record findings in `ci_evidence/production/runtime_maturity/local_code_audit.md`
- [x] T009 Map existing tests that cover env contracts, runtime mode authority, endpoint isolation, replay, queue routing, and causality export in `ci_evidence/production/runtime_maturity/test_coverage_map.md`
- [x] T010 [P] Add missing environment contract expectations for `backend/.env` authority and repo-root `.env` exclusion in `backend/tests/unit/pipeline/test_runtime_env_authority.py`
- [x] T011 [P] Add missing production inference fail-closed expectations for `INFERENCE_STRATEGY=triton_only` in `backend/tests/unit/pipeline/test_production_inference_strategy.py`
- [x] T012 [P] Add missing endpoint isolation expectations for live/offline Triton profiles in `backend/tests/contract/test_runtime_mode_contract.py`
- [x] T013 [P] Add missing queue route ownership expectations for production inference queues in `backend/tests/system/test_wave1_runtime_policy.py`

## Phase 3: User Story 1 - Branch and Deployment Governance

**Goal:** Production, origin, and local state are verifiably aligned to the temporary stabilization branch before production validation starts.

**Independent test:** Run local and production hash parity checks and confirm the evidence file records local HEAD, origin HEAD, production HEAD, branch names, tracked dirty state, and reviewed stash state.

- [x] T014 [US1] Update `tools/prod/prod-hash-parity.ps1` to default to `release/prod-runtime-stabilization` and capture local, origin, and production HEAD values
- [x] T015 [P] [US1] Update `tools/prod/prod-stash-hygiene.ps1` to write reviewed stash summaries without exposing secrets
- [x] T016 [US1] Add branch parity output fields to `tools/prod/write-evidence-manifest.py`
- [x] T017 [P] [US1] Add local tests for branch parity manifest formatting in `backend/tests/unit/scripts/test_production_branch_parity_manifest.py`
- [ ] T018 [US1] Run `tools/prod/prod-hash-parity.ps1` and save output to `ci_evidence/production/runtime_maturity/phase1_hash_parity.md`
- [ ] T019 [US1] Run `tools/prod/prod-stash-hygiene.ps1` and save reviewed output to `ci_evidence/production/runtime_maturity/phase1_stash_hygiene.md`

## Phase 4: User Story 2 - Runtime Contract Lock

**Goal:** Production startup fails closed before processing any video when the runtime contract is invalid.

**Independent test:** Run the production runtime preflight on the RTX 5090 server and confirm it validates Linux/no-Docker/no-sudo assumptions, backend venv Python, pinned Triton binary, active endpoint readiness, inactive endpoint unreachable, PostgreSQL authority, Redis reachability, Celery queues, model artifact hashes, and `backend/.env` fingerprint.

- [x] T020 [US2] Implement or harden production runtime preflight checks in `tools/prod/prod-runtime-preflight.sh`
- [x] T021 [US2] Add backend env authority checks that reject repo-root `.env` as backend startup authority in `backend/config/runtime_env.py`
- [x] T022 [US2] Enforce production `INFERENCE_STRATEGY=triton_only` fail-closed behavior in `backend/apps/pipeline/multi_model.py`
- [x] T023 [US2] Enforce no local model fallback for production inference in `backend/apps/video_analysis/services/inference_orchestrator.py`
- [x] T024 [US2] Ensure Celery production queue routes are explicit for runtime inference in `backend/config/celery.py`
- [x] T025 [P] [US2] Add preflight tests for `backend/.env` authority in `backend/tests/unit/pipeline/test_runtime_env_authority.py`
- [x] T026 [P] [US2] Add preflight tests for Triton-only production enforcement in `backend/tests/unit/pipeline/test_production_inference_strategy.py`
- [x] T027 [P] [US2] Add endpoint active/inactive contract tests in `backend/tests/contract/test_runtime_mode_contract.py`
- [x] T028 [US2] Run focused local tests for env, strategy, runtime mode, and queue policy and save output to `ci_evidence/production/runtime_maturity/phase2_local_tests.md`
- [ ] T029 [US2] Run `tools/prod/prod-runtime-preflight.sh` on production and save output to `ci_evidence/production/runtime_maturity/phase2_prod_preflight.md`

## Phase 5: User Story 3 - Replay and Lifecycle Integrity

**Goal:** `runtime_ingest_video` uses explicit replay policy and cannot treat failed replay lineage as valid production acceptance evidence.

**Independent test:** Submit commands for `reuse-success`, `fail-on-existing`, and `new-attempt`; confirm only successful completed replay can be reused, existing keys can be rejected, and new attempts link lineage without replacing fresh validation.

- [x] T030 [US3] Add replay policy CLI options to `backend/apps/video_analysis/management/commands/runtime_ingest_video.py`
- [x] T031 [US3] Persist replay policy, replay key, source hash, original job ID, and lineage status in `backend/apps/video_analysis/models.py`
- [x] T032 [US3] Create the required replay lineage migration in `backend/apps/video_analysis/migrations/`
- [x] T033 [US3] Reject failed replay lineage as production acceptance evidence in `backend/apps/video_analysis/management/commands/runtime_ingest_video.py`
- [x] T034 [P] [US3] Add replay policy unit tests in `backend/tests/unit/video_analysis/test_runtime_ingest_replay_policy.py`
- [x] T035 [P] [US3] Add replay lifecycle contract tests in `backend/tests/contract/test_runtime_ingest_replay_contract.py`
- [x] T036 [US3] Update production ingest wrapper policy flags in `tools/prod/prod-runtime-ingest-video.ps1`
- [x] T037 [US3] Run local replay policy tests and save output to `ci_evidence/production/runtime_maturity/phase3_replay_tests.md`
- [ ] T038 [US3] Run a fresh production real-video lifecycle job and save command output to `ci_evidence/production/runtime_maturity/phase3_fresh_lifecycle_job.md`

## Phase 6: User Story 4 - GPU and Causality Certification

**Goal:** Production evidence proves the exact accepted job ran through Triton on the RTX 5090 with correct endpoint isolation, queue behavior, batching, telemetry, and causality export.

**Independent test:** For one accepted production job ID, validate GPU trace, Triton readiness/model status, inactive endpoint unreachable, queue telemetry, causality export, and manifest fields all refer to the same job ID, Git SHA, env fingerprint, runtime profile, and model artifact hashes.

- [x] T039 [US4] Add job-scoped GPU telemetry capture to `tools/prod/prod-health-snapshot.ps1`
- [x] T040 [US4] Add Triton active/inactive endpoint evidence capture to `tools/prod/prod_triton_endpoint_policy.sh`
- [x] T041 [US4] Add model artifact hash collection to `tools/prod/write-evidence-manifest.py`
- [x] T042 [US4] Add queue topology and Celery worker state collection to `tools/prod/prod-workers.ps1`
- [x] T043 [US4] Ensure causality export accepts and records the exact job ID in `tools/benchmarks/export_benchmark_causality.py`
- [x] T044 [P] [US4] Add manifest schema tests for GPU trace, queue topology, model hashes, and replay fields in `backend/tests/unit/scripts/test_production_evidence_manifest.py`
- [x] T045 [P] [US4] Add causality job-ID integrity tests in `backend/tests/unit/scripts/test_benchmark_causality_job_integrity.py`
- [ ] T046 [US4] Capture production GPU telemetry during the accepted job and save output under `ci_evidence/production/runtime_maturity/phase4_gpu_trace/`
- [ ] T047 [US4] Export causality for the accepted job and save output under `ci_evidence/production/runtime_maturity/phase4_causality/`
- [ ] T048 [US4] Generate a production evidence manifest for the accepted job at `ci_evidence/production/runtime_maturity/phase4_evidence_manifest.json`

## Phase 7: User Story 5 - Closure and Reproducibility

**Goal:** The stabilization branch is merged back after sign-off, production is clean, and evidence can be replayed and audited from committed source.

**Independent test:** Verify local HEAD equals origin HEAD, production HEAD equals origin HEAD, production tracked tree is clean, reviewed stash items are archived or dropped only when understood, and evidence package validates against the accepted job manifest.

- [x] T049 [US5] Add final evidence package validation to `tools/prod/write-evidence-manifest.py`
- [x] T050 [P] [US5] Document final acceptance gates in `docs/production/runtime_maturity_operator_checklist.md`
- [ ] T051 [US5] Package production evidence under `ci_evidence/production/runtime_maturity/final/`
- [ ] T052 [US5] Run successful replay validation for the accepted job and save output to `ci_evidence/production/runtime_maturity/final/successful_replay_validation.md`
- [ ] T053 [US5] Verify final local/origin/production hash parity and save output to `ci_evidence/production/runtime_maturity/final/hash_parity.md`
- [ ] T054 [US5] Verify final production stash and tracked-tree hygiene and save output to `ci_evidence/production/runtime_maturity/final/prod_hygiene.md`
- [ ] T055 [US5] Merge `release/prod-runtime-stabilization` back into the active development branch and record merge SHA in `ci_evidence/production/runtime_maturity/final/merge_record.md`

## Final Phase: Polish and Cross-Cutting

- [ ] T056 [P] Update `AGENTS.md` with completed phase status and final evidence paths after sign-off
- [ ] T057 [P] Update `docs/heterogeneous_production_runtime_maturity_plan.md` with completed phase status and accepted job ID after sign-off
- [ ] T058 [P] Update `docs/linux_production_optimization_execution_phases.md` only where older instructions conflict with the accepted maturity plan
- [x] T059 Run the local focused backend suite for runtime authority, replay, queue policy, and causality and save output to `ci_evidence/production/runtime_maturity/final/local_test_summary.md`
- [ ] T060 Run production preflight, endpoint policy, lifecycle, GPU telemetry, causality export, and evidence manifest validation and save output to `ci_evidence/production/runtime_maturity/final/production_acceptance_summary.md`

## Dependencies

- Phase 1 must complete before production deployment changes.
- Phase 2 blocks all user stories because it establishes test and audit coverage.
- US1 must complete before US2 production validation so evidence is tied to a branch and SHA.
- US2 must complete before US3 and US4 production jobs so invalid runtime shape fails before processing.
- US3 must complete before US4 acceptance evidence so replay lineage cannot mask failures.
- US4 must complete before US5 closure.
- US5 must complete before the release branch is considered disposable.

## Parallel Execution Examples

- Setup parallel: T002 and T003 can run while T001 records branch state.
- Foundational parallel: T010, T011, T012, and T013 target different tests and can run after T008 identifies gaps.
- US1 parallel: T015 and T017 can run while T014 is implemented.
- US2 parallel: T025, T026, and T027 can run after the expected contract shape is known.
- US3 parallel: T034 and T035 can run while T036 updates the wrapper.
- US4 parallel: T044 and T045 can run while production helper evidence capture is updated.
- Final parallel: T056, T057, and T058 can run after final evidence paths are known.

## MVP Scope

The MVP is US1 plus US2: branch/SHA governance and runtime contract lock. Do not run fresh production lifecycle acceptance until these two stories are complete and their evidence is saved.

## Validation Commands

- Local focused tests:
  - `.\.venv\Scripts\python.exe -m pytest backend/tests/unit/pipeline/test_runtime_env_authority.py backend/tests/unit/pipeline/test_production_inference_strategy.py backend/tests/contract/test_runtime_mode_contract.py backend/tests/system/test_wave1_runtime_policy.py -q --tb=short`
- Replay tests:
  - `.\.venv\Scripts\python.exe -m pytest backend/tests/unit/video_analysis/test_runtime_ingest_replay_policy.py backend/tests/contract/test_runtime_ingest_replay_contract.py -q --tb=short`
- Production preflight:
  - `.\tools\prod\prod-ssh.ps1 -Cmd "cd /home/bamby/grad_project && bash tools/prod/prod-runtime-preflight.sh"`
- Production hash parity:
  - `.\tools\prod\prod-hash-parity.ps1`
- Production lifecycle:
  - `.\tools\prod\prod-runtime-ingest-video.ps1`

