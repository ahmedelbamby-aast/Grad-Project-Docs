# RTMPose Tasks: ⚠️-Only Completion Plan

**Input**: `rtmpose_plan` (repo root) and `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-engineering-validation-report.md`  
**Scope Rule**: Implement **all and only** ⚠️ items; keep all ❌ items untouched/out-of-scope.  
**Architecture Rule**: Reuse current runtime architecture; no new major feature family and no new core DB tables.  
**Test Rule**: Test-in-loop (RED -> GREEN -> REFACTOR) with direct evidence artifacts per phase.

## Format: `[ID] [P?] [Area] Description`

- **[P]**: Parallelizable with non-overlapping files and no blocking dependency.
- **[Area]**: `SETUP`, `STREAM`, `DETECT`, `TRACK`, `POSE`, `TEMPORAL`, `STORAGE`, `OBS`, `SCI`, `API`, `UI`, `EVIDENCE`, `TEST-*`.

---

## Phase 0: Setup, Scope Guardrails, and Baseline

- [X] RT001 [SETUP] Create scope-lock note for ⚠️-only implementation in `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-scope-lock.md`.
- [X] RT002 [P] [SETUP] Create task evidence index in `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-task-evidence-index.md`.
- [X] RT003 [P] [SETUP] Capture baseline runtime payload samples before changes in `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-baseline-runtime-payloads.md`.
- [X] RT004 [P] [SETUP] Capture baseline backend test command outputs in `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-baseline-backend-tests.md`.
- [X] RT005 [P] [SETUP] Capture baseline frontend test command outputs in `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-baseline-frontend-tests.md`.

---

## Phase 1: Stream Runtime Hardening

### Tests First (RED)

- [X] RT006 [P] [TEST-UNIT] Add unit tests for timeout settings parsing/defaults in `backend/tests/unit/pipeline/test_runtime_stream_timeouts.py`.
- [X] RT007 [P] [TEST-UNIT] Add unit tests for bounded frame/event deque overflow policy (`drop_oldest_keep_latest`) in `backend/tests/unit/pipeline/test_runtime_stream_buffers.py`.
- [X] RT008 [P] [TEST-UNIT] Add unit tests for monotonic timestamp enforcement and frame delta calculation in `backend/tests/unit/pipeline/test_runtime_timestamps.py`.
- [X] RT009 [P] [TEST-INTEGRATION] Add integration test proving runtime event payload includes stream timing and overflow/backpressure counters in `backend/tests/integration/test_runtime_event_stream_fields.py`.

### Implementation

- [X] RT010 [STREAM] Add explicit stream control settings (`*_TIMEOUT_MS`, `*_BUFFER_MAX_FRAMES`, `*_QUEUE_MAX_EVENTS`, `*_MAX_EMPTY_READS`) in `backend/apps/pipeline/config.py`.
- [X] RT011 [STREAM] Enforce monotonic timestamp sync and emit `source_timestamp_ms`, `ingest_timestamp_ms`, `frame_delta_ms`, `effective_fps` in `backend/apps/pipeline/runtime_ingestion.py`.
- [X] RT012 [STREAM] Add bounded frame/event buffers and overflow counters in `backend/apps/pipeline/runtime_ingestion.py`.
- [X] RT013 [STREAM] Emit overflow/backpressure counters into lifecycle/summary payloads in `backend/apps/pipeline/services.py`.
- [X] RT014 [P] [DOC] Document new stream settings and payload fields in `docs/backend/apps/pipeline/config.md` and `docs/backend/apps/pipeline/services.md`.

---

## Phase 2: Detection-Layer Partial Completion

### Tests First (RED)

- [X] RT015 [P] [TEST-UNIT] Add unit tests for intra-model duplicate suppression IoU behavior in `backend/tests/unit/pipeline/test_detector_duplicate_suppression.py`.
- [X] RT016 [P] [TEST-UNIT] Add unit tests for cross-model person-box duplicate guard in `backend/tests/unit/pipeline/test_detector_cross_model_guard.py`.
- [X] RT017 [P] [TEST-UNIT] Add unit tests for occlusion hold/expiration behavior in `backend/tests/unit/pipeline/test_detector_occlusion_carry_forward.py`.
- [X] RT018 [P] [TEST-SYSTEM] Add benchmark test for detector `fps` and `latency_ms` field emission in `backend/tests/system/test_detector_metrics_benchmark.py`.

### Implementation

- [X] RT019 [DETECT] Implement deterministic post-detection suppression pass in `backend/apps/pipeline/detector.py`.
- [X] RT020 [DETECT] Implement cross-model duplicate guard for person boxes in `backend/apps/pipeline/multi_model.py`.
- [X] RT021 [DETECT] Add occlusion-aware carry-forward policy with expiration in `backend/apps/pipeline/tracker.py`.
- [X] RT022 [DETECT] Emit detector benchmark metrics (`fps`, `latency_ms`) in runtime payloads via `backend/apps/pipeline/services.py`.
- [X] RT023 [P] [DOC] Document additive/configurable detection passes in `docs/backend/apps/pipeline/services.md`.

---

## Phase 3: Tracking Continuity and Lifecycle

### Tests First (RED)

- [X] RT024 [P] [TEST-UNIT] Add unit tests for lifecycle transitions (`active`, `occluded`, `reidentified`, `lost`, `ended`) in `backend/tests/unit/tracking/test_track_lifecycle_states.py`.
- [X] RT025 [P] [TEST-UNIT] Add unit tests for lost-track timeout (frame-based + ms-based) in `backend/tests/unit/tracking/test_track_timeout_config.py`.
- [X] RT026 [P] [TEST-UNIT] Add unit tests for continuity score computation in `backend/tests/unit/tracking/test_track_continuity_score.py`.
- [X] RT027 [P] [TEST-INTEGRATION] Add integration test for tracker desync recovery events in `backend/tests/integration/test_tracker_desync_recovery.py`.

### Implementation

- [X] RT028 [TRACK] Add lifecycle state machine fields to track metadata/events in `backend/apps/tracking/services/tracking_service.py`.
- [X] RT029 [TRACK] Add configurable lost-track timeout settings in `backend/apps/pipeline/config.py` and consume in `backend/apps/tracking/services/tracking_service.py`.
- [X] RT030 [TRACK] Add per-track continuity scoring and emit to runtime/student timeline payloads in `backend/apps/pipeline/services.py` and `backend/apps/video_analysis/views.py`.
- [X] RT031 [TRACK] Implement desync recovery trigger + recovery event logging in `backend/apps/tracking/services/tracking_service.py`.
- [X] RT032 [P] [DOC] Document lifecycle semantics and continuity fields in `docs/backend/apps/tracking/services/tracking_service.md`.

---

## Phase 4: RTMPose Runtime Partial Completion

### Tests First (RED)

- [X] RT033 [P] [TEST-UNIT] Add unit tests for provider selection and fallback reason mapping in `backend/tests/unit/pipeline/test_rtmpose_provider_fallback_reasons.py`.
- [X] RT034 [P] [TEST-UNIT] Add unit tests for model identity validation and startup event in `backend/tests/unit/pipeline/test_rtmpose_model_identity.py`.
- [X] RT035 [P] [TEST-INTEGRATION] Add integration test proving provider/device/GPU status fields are present in runtime payloads in `backend/tests/integration/test_rtmpose_runtime_provider_fields.py`.
- [X] RT036 [P] [TEST-SYSTEM] Add system benchmark for pose FPS and e2e latency output in `backend/tests/system/test_rtmpose_fps_e2e_latency_benchmark.py`.

### Implementation

- [X] RT037 [POSE] Replace mock-only path with provider-backed inference path (ONNXRuntime primary + existing fallback policy) in `backend/apps/pipeline/services/pose_runtime.py`.
- [X] RT038 [POSE] Preserve current ROI crop/map contract while switching runtime execution path in `backend/apps/pipeline/services/rtmpose_pipeline.py`.
- [X] RT039 [POSE] Validate RTMPose model identity at startup and emit `model_identity` event in `backend/apps/pipeline/model_lifecycle/rtmpose_manifest.py` and `backend/apps/pipeline/model_bootstrap.py`.
- [X] RT040 [POSE] Emit provider/device/GPU execution fields in runtime payloads via `backend/apps/pipeline/services.py`.
- [X] RT041 [POSE] Emit explicit fallback reason codes (`provider_error`, `timeout`, `model_unavailable`, ...) in `backend/apps/pipeline/services/pose_runtime.py`.
- [X] RT042 [P] [DOC] Update runtime provider/fallback docs in `docs/backend/apps/pipeline/services/pose_runtime.md`.

---

## Phase 5: Temporal Buffer and Timestamp Alignment

### Tests First (RED)

- [X] RT043 [P] [TEST-UNIT] Add unit tests for per-track rolling temporal buffer windows (seconds + max frames) in `backend/tests/unit/pipeline/test_temporal_track_buffer.py`.
- [X] RT044 [P] [TEST-UNIT] Add unit tests for continuity checks (no backward time, bounded gaps) in `backend/tests/unit/pipeline/test_temporal_continuity_checks.py`.
- [X] RT045 [P] [TEST-INTEGRATION] Add integration test for runtime timeline freshness/buffer health fields in `backend/tests/integration/test_runtime_timeline_buffer_health.py`.

### Implementation

- [X] RT046 [TEMPORAL] Implement per-track rolling temporal buffer keyed by tracking ID in `backend/apps/pipeline/services/rtmpose_pipeline.py`.
- [X] RT047 [TEMPORAL] Store aligned timestamps with buffered points and enforce continuity checks in `backend/apps/pipeline/services.py`.
- [X] RT048 [TEMPORAL] Expose buffer health and freshness in runtime timeline APIs in `backend/apps/video_analysis/views.py` and `backend/apps/anomalies/views.py`.
- [X] RT049 [P] [DOC] Document temporal buffer health/freshness fields in `docs/backend/apps/video_analysis/services.md`.

---

## Phase 6: Behavior Feature + Temporal Embedding Storage Completion

### Tests First (RED)

- [X] RT050 [P] [TEST-UNIT] Add unit tests for behavior proxy extraction payload blocks in `backend/tests/unit/pipeline/test_behavior_proxy_storage_blocks.py`.
- [X] RT051 [P] [TEST-UNIT] Add unit tests for lightweight temporal embedding generation from keypoint deltas/windows in `backend/tests/unit/pipeline/test_temporal_embedding_windows.py`.
- [X] RT052 [P] [TEST-INTEGRATION] Add integration test for persistence/readback of behavior + embedding metadata blocks in `backend/tests/integration/test_behavior_embedding_metadata_persistence.py`.

### Implementation

- [X] RT053 [STORAGE] Persist behavior proxies and continuity/jitter summaries into structured metadata in `backend/apps/pipeline/models.py` and `backend/apps/video_analysis/models.py`.
- [X] RT054 [STORAGE] Persist lightweight temporal embeddings in runtime/offline metadata payloads in `backend/apps/pipeline/services.py`.
- [X] RT055 [STORAGE] Add compact numeric encoding and capped history strategy in `backend/apps/pipeline/exporter.py`.
- [X] RT056 [P] [DOC] Document storage contract and limits in `docs/backend/apps/pipeline/models.md`.

---

## Phase 7: Runtime Observability + Temporal Replay Visualization

### Tests First (RED)

- [X] RT057 [P] [TEST-CONTRACT] Add contract tests for extended runtime event schema fields in `backend/tests/contract/test_runtime_event_schema_contract.py`.
- [X] RT058 [P] [TEST-INTEGRATION] Add integration test for `runtime/timeline`, `runtime/student-timeline`, `runtime/summary` extended fields in `backend/tests/integration/test_runtime_api_extended_fields.py`.
- [X] RT059 [P] [TEST-UNIT] Add frontend runtime type tests for new API fields in `frontend/tests/unit/api/runtime-contracts.test.ts`.
- [X] RT060 [P] [TEST-INTEGRATION] Add frontend integration test for replay controls and observability panels in `frontend/tests/integration/runtime-replay-observability.test.tsx`.

### Implementation

- [X] RT061 [OBS] Extend runtime API payloads with GPU/memory/queue/throughput metrics in `backend/apps/anomalies/views.py` and `backend/apps/video_analysis/views.py`.
- [X] RT062 [OBS] Add temporal replay ordered-point surfaces with lifecycle state/timestamps in `backend/apps/video_analysis/views.py`.
- [X] RT063 [UI] Extend runtime frontend types in `frontend/src/types/runtime.ts`.
- [X] RT064 [UI] Add/adjust runtime API client mappings in `frontend/src/api/runtime.ts`.
- [X] RT065 [UI] Render replay controls and GPU/queue/backpressure metrics in `frontend/src/pages/RuntimePage.tsx`.
- [X] RT066 [P] [DOC] Update runtime API and UI docs in `docs/frontend/src/pages/RuntimePage.md` and `docs/backend/apps/video_analysis/services.md`.

---

## Phase 8: Scientific/Readiness Validation Completion

### Tests and Validation Scripts First (RED)

- [X] RT067 [P] [TEST-SYSTEM] Add deterministic temporal stability score test in `backend/tests/system/test_temporal_stability_score.py`.
- [X] RT068 [P] [TEST-SYSTEM] Add deterministic noise-reduction score test in `backend/tests/system/test_temporal_noise_reduction_score.py`.
- [X] RT069 [P] [TEST-SYSTEM] Add deterministic temporal continuity score test in `backend/tests/system/test_temporal_continuity_score.py`.
- [X] RT070 [P] [TEST-SYSTEM] Add multi-person long-session continuity robustness scenario in `backend/tests/system/test_multi_person_long_session_continuity.py`.
- [X] RT071 [P] [TEST-SYSTEM] Add contrastive-learning-ready tensor export sanity test in `backend/tests/system/test_contrastive_tensor_export_sanity.py`.

### Implementation

- [X] RT072 [SCI] Add deterministic runtime validation helpers in `backend/apps/pipeline/utils.py`.
- [X] RT073 [SCI] Add reusable scoring/aggregation utilities in `backend/apps/pipeline/results.py`.
- [X] RT074 [P] [DOC] Add methodology + thresholds doc in `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-scientific-validation-method.md`.

---

## Phase 9: API and Contract Surface Completion

- [X] RT075 [API] Extend runtime event payload schema for timing, buffer, lifecycle, provider/fallback, and resource metrics in `backend/apps/pipeline/contracts.py`.
- [X] RT076 [API] Extend `runtime/timeline`, `runtime/student-timeline`, `runtime/summary` serializers/responses in `backend/apps/video_analysis/serializers.py` and `backend/apps/anomalies/serializers.py`.
- [X] RT077 [P] [TEST-CONTRACT] Add/update API contract tests for all new fields in `backend/tests/contract/test_runtime_timeline_api_contract.py` and `backend/tests/contract/test_runtime_summary_api_contract.py`.
- [X] RT078 [P] [TEST-CONTRACT] Add websocket contract test for runtime lifecycle fields in `backend/tests/contract/test_runtime_websocket_contract.py`.

---

## Phase 10: Full Test Matrix (All Test Types)

### Unit

- [X] RT079 [TEST-UNIT] Run backend unit suite for modified pipeline/tracking modules: `\.venv\Scripts\python.exe -m pytest backend/tests/unit/pipeline backend/tests/unit/tracking -n auto --dist=loadscope -q --tb=short`.
- [X] RT080 [TEST-UNIT] Run frontend unit suite: `npm run test:unit:parallel`.

### Integration

- [X] RT081 [TEST-INTEGRATION] Run backend integration suite: `\.venv\Scripts\python.exe -m pytest backend/tests/integration -n auto --dist=loadscope -q --tb=short`.
- [X] RT082 [TEST-INTEGRATION] Run frontend integration tests: `npm run test:all:parallel` (collect integration assertions from run output).

### Contract

- [X] RT083 [TEST-CONTRACT] Run backend contract suite: `\.venv\Scripts\python.exe -m pytest backend/tests/contract -n auto --dist=loadscope -q --tb=short`.
- [X] RT084 [TEST-CONTRACT] Run frontend API/runtime contract tests in Vitest using `npm run test:unit:parallel`.

### System and E2E

- [X] RT085 [TEST-SYSTEM] Run targeted backend system tests for RTMPose runtime + timeline scenarios: `\.venv\Scripts\python.exe -m pytest backend/tests/system -n auto --dist=loadscope -q --tb=short`.
- [X] RT086 [TEST-E2E] Run frontend E2E suite: `npm run test:e2e:parallel`.

### Benchmark and Performance Regression

- [X] RT087 [TEST-BENCH] Run detector/pose/throughput benchmark tests and archive metrics JSON in `backend/tests/system/artifacts/`.
- [X] RT088 [TEST-PERF] Run backend performance regression suite: `\.venv\Scripts\python.exe -m pytest backend/tests/performance -n auto --dist=loadscope -q --tb=short`.

### Load

- [X] RT089 [TEST-LOAD] Run baseline load scenario: `k6 run --vus 50 --duration 5m load-tests/baseline.js`.
- [X] RT090 [TEST-LOAD] Run stress load scenario: `k6 run load-tests/scenarios/stress.js`.
- [X] RT091 [TEST-LOAD] Run Locust concurrent load scenario: `locust -f load-tests/locustfile.py --headless -u 200 -r 20 -t 10m --processes -1`.

### Resilience and Chaos

- [X] RT092 [TEST-RESILIENCE] Run backend resilience suite: `\.venv\Scripts\python.exe -m pytest backend/tests/resilience -n auto --dist=loadscope -q --tb=short`.
- [X] RT093 [TEST-CHAOS] Add/run provider timeout, tracker desync, and queue saturation chaos scenarios in `backend/tests/resilience/test_rtmpose_runtime_chaos.py`.

### Security

- [X] RT094 [TEST-SECURITY] Run frontend dependency audit: `npm audit --audit-level=high`.
- [X] RT095 [TEST-SECURITY] Run backend dependency audit: `\.venv\Scripts\python.exe -m pip-audit`.
- [X] RT096 [TEST-SECURITY] Add API authz regression checks for runtime timeline endpoints in `backend/tests/integration/test_runtime_timeline_authorization.py`.

### Accessibility and Visual Regression

- [X] RT097 [TEST-A11Y] Run accessibility suite: `npx playwright test --grep @a11y --workers=50%`.
- [X] RT098 [TEST-VISUAL] Run visual regression suite: `npx playwright test --grep @visual --workers=50%`.

### Smoke and Canary

- [X] RT099 [TEST-SMOKE] Run frontend smoke tests: `npx playwright test --grep @smoke --workers=50%`.
- [X] RT100 [TEST-CANARY] Run backend canary tests: `\.venv\Scripts\python.exe -m pytest -m canary -n auto --dist=loadscope -q --tb=short`.
- [X] RT101 [TEST-SMOKE] Run backend smoke/canary gate combo: `\.venv\Scripts\python.exe -m pytest -m "smoke or canary" -n auto --dist=loadscope -q --tb=short`.

---

## Phase 11: Evidence, Report Flip, and Final Gate

- [X] RT102 [EVIDENCE] Record unit/integration/contract/system outputs in `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-test-results.md`.
- [X] RT103 [EVIDENCE] Record benchmark/performance/load/resilience outputs in `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-benchmark-and-load.md`.
- [X] RT104 [EVIDENCE] Record security/a11y/visual/smoke/canary outputs in `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-quality-gates.md`.
- [X] RT105 [EVIDENCE] Update `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-engineering-validation-report.md`: flip only addressed ⚠️ items to ✅.
- [X] RT106 [EVIDENCE] Verify all ❌ items remain unchanged and explicitly out-of-scope in `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-engineering-validation-report.md`.
- [X] RT107 [EVIDENCE] Add traceability matrix (⚠️ item -> code path -> test -> evidence) in `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-traceability-matrix.md`.
- [X] RT108 [EVIDENCE] Create release-readiness signoff note in `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-release-readiness.md`.

---

## Dependency Order

- [X] RT109 Complete `RT001-RT005` before any implementation phase.
- [X] RT110 Complete test-first tasks (`RT006-RT009`, `RT015-RT018`, `RT024-RT027`, `RT033-RT036`, `RT043-RT045`, `RT050-RT052`, `RT057-RT060`, `RT067-RT071`) before their corresponding implementation tasks.
- [X] RT111 Complete API/UI contract updates (`RT075-RT078`) before full matrix execution (`RT079-RT101`).
- [X] RT112 Complete evidence/report tasks (`RT102-RT108`) only after all mandatory test gates pass.

## Parallel Execution Notes

- Use framework-native parallelism only.
- Do not run multiple backend `pytest -n ...` invocations concurrently unless DB isolation per invocation is guaranteed.
- Prefer one backend suite at a time, with `-n auto --dist=loadscope`.
- Frontend can run unit/E2E parallel workers via existing npm scripts.



