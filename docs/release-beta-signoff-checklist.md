# Repo Hygiene Classifier (Agent C1)

Date: 2026-05-21  
Branch: `master`  
Scope: dirty working tree classification + non-destructive cleanup plan

## Summary

- Total dirty entries: `113`
- Classification buckets used:
  - `pre-existing user work`
  - `agent code`
  - `safe generated artifacts`
  - `unknown-risk`

## 1) Pre-existing User Work

Tracked edits and new source/test/spec artifacts that appear intentional and should be preserved for user review.

### Tracked modifications (product code, tests, docs/specs)

- `.env.example`
- `README.md`
- `backend/.env.example`
- `backend/apps/pipeline/multi_model.py`
- `backend/apps/pipeline/services/inference_client_factory.py`
- `backend/apps/pipeline/services/model_route_service.py`
- `backend/apps/pipeline/services/triton_client.py`
- `backend/apps/video_analysis/services/inference_orchestrator.py`
- `backend/apps/video_analysis/tasks.py`
- `backend/apps/video_analysis/urls.py`
- `backend/apps/video_analysis/views.py`
- `backend/config/celery.py`
- `backend/config/settings/base.py`
- `backend/config/settings/test.py`
- `backend/core/observability.py`
- `backend/pyproject.toml`
- `backend/pytest.ini`
- `backend/requirements.txt`
- `backend/tests/conftest.py`
- `backend/tests/contract/test_status.py`
- `backend/tests/contract/test_upload.py`
- `backend/tests/contract/test_visibility.py`
- `backend/tests/integration/test_full_inference_flow.py`
- `backend/tests/system/test_toggle_response_time.py`
- `backend/tests/unit/pipeline/test_dependency_injection_boundaries.py`
- `backend/tests/unit/pipeline/test_triton_client_validation.py`
- `backend/tests/unit/video_analysis/test_edge_cases.py`
- `backend/tests/unit/video_analysis/test_external_factor_events.py`
- `docker-compose.dev.yml`
- `frontend/package.json`
- `frontend/src/api/runtime.ts`
- `frontend/src/api/videoAnalysis.ts`
- `frontend/src/components/ModelVisibilityToggles/ModelVisibilityToggles.tsx`
- `frontend/src/components/VideoPlayer/VideoPlayer.tsx`
- `frontend/src/components/VideoUploadTab/JobsList.module.css`
- `frontend/src/components/VideoUploadTab/JobsList.tsx`
- `frontend/src/components/VideoUploadTab/VideoUploadTab.tsx`
- `frontend/src/components/camera/BoundingBoxCanvas.tsx`
- `frontend/src/pages/CameraFeedPage.tsx`
- `frontend/src/pages/DashboardPage.tsx`
- `frontend/src/pages/LoginPage.tsx`
- `frontend/src/pages/SettingsPage.tsx`
- `frontend/src/pages/VideoAnalysisPage.module.css`
- `frontend/src/pages/VideoAnalysisPage.tsx`
- `frontend/src/stores/authStore.ts`
- `frontend/src/stores/cameraStore.ts`
- `frontend/src/stores/detectionStore.ts`
- `frontend/src/stores/visibilityStore.ts`
- `frontend/src/types/videoAnalysis.ts`
- `frontend/tests/e2e/full-baseline.spec.ts`
- `frontend/tests/e2e/login.spec.ts`
- `frontend/tests/e2e/recording-playback.spec.ts`
- `frontend/tests/unit/api/dashboard.test.ts`
- `frontend/tests/unit/components/DashboardPage.test.tsx`
- `frontend/tests/unit/components/LoginPage.test.tsx`
- `frontend/tests/unit/components/anomaly/AlertHistory.test.tsx`
- `frontend/tests/unit/components/camera/BoundingBoxCanvas.test.tsx`
- `frontend/tests/unit/components/camera/BoundingBoxFiltered.test.tsx`
- `frontend/tests/unit/components/recording/RecordingPlayer.test.tsx`
- `frontend/tests/unit/stores/cameraStore.test.ts`
- `frontend/tests/unit/stores/detectionStore.test.ts`
- `frontend/tests/unit/stores/visibilityStore.test.ts`
- `rtmpose_tasks.md`
- `scripts/sync-triton-tensorrt-repository.ps1`
- `specs/001-exam-monitor-dashboard/quickstart.md`
- `specs/004-video-upload-inference-tab/quickstart.md`
- `specs/007-pose-behavior-pipeline/evidence/final/frontend-test-results.md`
- `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-benchmark-and-load.md`
- `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-engineering-validation-report.md`
- `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-quality-gates.md`
- `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-release-readiness.md`
- `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-test-results.md`
- `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-traceability-matrix.md`

### Untracked files likely intentional user work

- `backend/tests/integration/conftest.py`
- `backend/tests/integration/test_partial_results.py`
- `backend/tests/integration/test_runtime_timeline_authorization.py`
- `backend/tests/performance/test_rtmpose_runtime_performance.py`
- `backend/tests/performance/test_spec008_staged_load_harness.py`
- `backend/tests/resilience/test_rtmpose_runtime_chaos.py`
- `backend/tests/unit/pipeline/test_spec008_observability.py`
- `backend/tests/unit/video_analysis/test_triton_offline_batch_queue.py`
- `ci_evidence/spec008/load-validation-runbook.md`
- `frontend/tests/unit/components/ModelVisibilityToggles.test.tsx`
- `frontend/tests/unit/stores/authTheme.test.ts`
- `load-tests/baseline.js`
- `load-tests/locustfile.py`
- `load-tests/scenarios/stress.js`
- `load-tests/staged_validation.py`
- `specs/008-triton-upload-pipeline-optimization/evidence/final/implementation-evidence.md`
- `specs/008-triton-upload-pipeline-optimization/evidence/final/staged-load-validation.md`
- `specs/008-triton-upload-pipeline-optimization/plan.md`
- `specs/008-triton-upload-pipeline-optimization/tasks.md`

## 2) Agent Code

Artifacts that appear agent-authored or agent-owned in this cleanup pass.

- `docs/release-beta-signoff-checklist.md` (this classifier report)

## 3) Safe Generated Artifacts

Ephemeral runtime/test outputs; safe to ignore for release hygiene and move out of commit scope.

- `backend/tests/system/artifacts/real_video_benchmark_metrics.json`
- `backend/tests/system/artifacts/real_video_benchmark_recommendations.json`
- `backend/tests/system/artifacts/real_video_benchmark_results.json`
- `backend/data/sessions/live_sessions/*/lifecycle_events.jsonl` (14 files)
- `backend/test_db.sqlite3`
- `backend/test_db_*.sqlite3*` (including `*_gw*` and `*-journal` variants)

## 4) Unknown-Risk

Items that are plausibly generated/tooling-related but require explicit owner decision before keep/ignore policy is finalized.

- `pip-audit.py` (could be ad-hoc utility; provenance unclear)
- `tools/k6/k6-v2.0.0-windows-amd64/k6.exe` (binary in repo tree)
- `tools/k6/k6.zip` (archived binary payload)

## Non-Destructive Cleanup Plan (No Deletions)

1. Preserve all `pre-existing user work` in a dedicated feature commit set; do not mix with artifact hygiene.
2. Keep `agent code` in a separate docs-only commit for auditability.
3. Quarantine `safe generated artifacts` from VCS scope without deleting:
   - Add ignore rules for:
     - `backend/test_db*.sqlite3*`
     - `backend/data/sessions/live_sessions/**/lifecycle_events.jsonl`
     - `backend/tests/system/artifacts/real_video_benchmark_*.json`
   - For already-tracked generated JSON artifacts, decide one of:
     - keep tracked as evidence and stop regenerating in normal test runs, or
     - untrack in a dedicated hygiene PR (non-destructive to local files).
4. Resolve `unknown-risk` explicitly:
   - If required for reproducible load/security workflows, move to approved tool/docs location and document checksum/version.
   - Otherwise mark as local-only and ignore via repo policy.
5. Before beta cut, run a hygiene gate:
   - `git status --porcelain=v1 -uall`
   - Expect only intentional source/docs/test files; no DB/session/journal/runtime artifacts.

## Suggested Commit Partition (for clean history)

1. `feat/spec008-runtime-and-tests` -> all intentional code/tests/spec updates.
2. `docs/release-beta-signoff-checklist-c1` -> this file only.
3. `chore/gitignore-generated-artifacts` -> ignore-policy updates only.
4. `chore/tooling-provenance-k6-pipaudit` -> resolve unknown-risk files only.
