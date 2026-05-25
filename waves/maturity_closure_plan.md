# Maturity Closure Waves For Production Behavioral Intelligence Readiness

This plan is intentionally detailed for PR review. It contains full implementation-level work items for each wave, acceptance gates, test obligations, and evidence artifacts.

## Global Guardrails

1. Production mode is Triton-only.
2. Production mode keeps dual Triton endpoints configured:
   - one for live processing
   - one for offline video processing
3. Production host runs one mode at a time, selected via `.env`.
3. Linux production constraints are no Docker and no sudo.
4. Behavioral intelligence maturity is blocked until identity continuity and temporal truth are stable.
5. Mock-only validations do not count as production acceptance evidence.
6. Any `xfail(strict=True)` scaffold must be converted or explicitly deferred with rationale.

## Wave 1: Production Policy And Deployment Consistency

### Objectives

1. Remove endpoint authority conflicts across env/settings/scripts/docs.
2. Preserve dual endpoint configuration while enforcing single active mode at runtime.
3. Produce live production evidence, not script-text-only evidence.

### Scope Of Work

1. Define endpoint policy authority with dual configured endpoints and one active mode selector in `.env`.
2. Align `.env.example`, backend env examples, and production settings values.
3. Align quickstart and production docs to dual-endpoint configured, single-mode active workflow.
4. Align `tools/prod` scripts so start/status/preflight all enforce active endpoint policy.
5. Ensure no guidance implies Docker/sudo on production path.
6. Split dev fallback behavior from production behavior in docs/tests.
7. Replace endpoint scaffolds that conflict with "dual configured endpoints + one active mode at runtime."

### Deliverables

1. Canonical endpoint policy section in docs and runbooks.
2. Production config examples with one active endpoint model.
3. Updated script checks for active endpoint health and inactive endpoint unreachable state.
4. Updated tests for profile/endpoint resolution.

### Acceptance Gates

1. The `.env` active mode selector is validated (`live` or `offline`).
2. Active mode endpoint returns ready health.
3. Inactive mode endpoint is not used by production workers/app runtime.
3. Backend health and model-serving health pass.
4. Docs, envs, and scripts no longer conflict on ports/profiles/mode selector behavior.
5. Production path has no Docker/sudo dependency.

### Required Tests

1. Unit tests for endpoint resolution and production profile handling.
2. Script/unit test coverage for endpoint policy script behavior.
3. Production health snapshot validation command execution.
4. Boundary verifier execution and pass evidence.

### Required Evidence Artifacts

1. `ci_evidence/production/wave1/hash_parity.md`
2. `ci_evidence/production/wave1/endpoint_health.md`
3. `ci_evidence/production/wave1/ports_snapshot.txt`
4. `ci_evidence/production/wave1/backend_health.json`
5. `ci_evidence/production/wave1/model_serving_health.json`
6. `ci_evidence/production/wave1/boundary_verifier.txt`

## Wave 2: Ingestion, Queue Routing, Backpressure, And RTSP Recovery

### Objectives

1. Make queue routing authority unambiguous.
2. Close loop from queue pressure to ingestion behavior.
3. Implement explicit RTSP reconnect state machine.
4. Ensure source-accurate timestamp and drop accounting.

### Scope Of Work

1. Normalize Celery queue source-of-truth in one route map.
2. Explicitly map offline fanout behavior to offline queues or document intentional non-use.
3. Wire live task queue-wait telemetry (`queued_at_ms` to queue metrics start).
4. Add DLQ/dead-letter handling and starvation/collapse protections.
5. Integrate `DelayedLiveBuffer` into live path or retire/de-scope it explicitly.
6. Implement RTSP reconnect lifecycle states and bounded retry with jitter.
7. Eliminate blocking retry behavior where it stalls live loop.
8. Add source timestamp + ingest timestamp propagation across live payloads.
9. Add per-reason drop accounting:
   - decode failure
   - stale discard
   - buffer overflow
   - backpressure drop
   - timeout fallback
   - downstream failure
10. Reduce live per-frame synchronous persistence/broadcast bottleneck via batching/defer paths.

### Deliverables

1. Unified queue routing contract doc.
2. Live queue telemetry instrumentation in runtime events.
3. Reconnect state machine implementation and transition logs.
4. Drop-accounting schema and counters.
5. Backpressure control rules tied to queue depth and latency thresholds.

### Acceptance Gates

1. Queue names in metrics match queue names in actual routing.
2. Live queue wait is recorded and queryable.
3. RTSP disconnect/reconnect scenarios pass state transition assertions.
4. Backpressure actions are triggered under overload, not only observed.
5. Preferred/fallback consumer absence is detected in preflight.
6. Source timestamp truth and drop reasons are visible in persisted telemetry.

### Required Tests

1. Unit tests for queue route map and alias behavior.
2. Unit tests for live queue-wait telemetry.
3. Integration tests for RTSP disconnect/reconnect/fail-stop matrix.
4. Resilience tests for Triton timeout + reuse fallback.
5. Load tests for burst ingestion and queue collapse prevention.

### Required Evidence Artifacts

1. `ci_evidence/production/wave2/queue_routing_matrix.md`
2. `ci_evidence/production/wave2/active_queues.txt`
3. `ci_evidence/production/wave2/live_queue_wait_events.json`
4. `ci_evidence/production/wave2/rtsp_fault_matrix.md`
5. `ci_evidence/production/wave2/drop_accounting_report.json`
6. `ci_evidence/production/wave2/backpressure_slo_report.md`

## Wave 3: Identity Continuity, Tracking Lifecycle, ReID, And Association Correctness

### Objectives

1. Eliminate identity collisions and cross-talk in live multi-camera sessions.
2. Promote ReID output from metadata note to canonical identity decision path.
3. Replace index-based interpolation matching with identity-aware association.
4. Unify lifecycle state behavior in primary persistence path.

### Scope Of Work

1. Define identity namespace contract with required keys:
   - `session_id`
   - `camera_id`
   - `canonical_track_id`
   - `local_track_id`
2. Update Redis keys and runtime identifiers to include camera scope.
3. Replace session-only live identity keys to avoid camera overwrite.
4. Define explicit ReID decision policy for this wave:
   - in-camera alias/merge supported
   - preserve provenance of original local track id
   - store match score and threshold decision metadata
5. Move `reid_best_match` out of failure metadata and into typed identity mapping record.
6. Wire lifecycle states into persisted detection timeline:
   - active
   - occluded
   - reidentified
   - lost
   - ended
7. Replace interpolation-by-index with association matching and gating:
   - cost matrix (IoU/center distance/size consistency)
   - deterministic matching (Hungarian or equivalent)
   - reject low-confidence associations
8. Add identity quality metrics:
   - ID switch count
   - occlusion recovery rate
   - re-entry precision/recall
   - fragmentation
9. Convert stale scaffold tests that claim pending reuse semantics.

### Deliverables

1. Identity scope contract doc.
2. Canonical ReID mapping persistence path and audit events.
3. Association-based interpolation module and tests.
4. Tracking lifecycle persistence integration.
5. Identity quality metric exports.

### Acceptance Gates

1. Two cameras in same session cannot collide identity keys.
2. ReID merges/aliases are persisted in canonical mapping path.
3. Lifecycle states appear in timeline and artifacts.
4. Interpolation does not rely on raw list index when identity/association exists.
5. High-density crossing tests show reduced invalid identity bridge behavior.
6. Strict xfail scaffolds for implemented behaviors are removed or corrected.

### Required Tests

1. Unit tests for key partitioning and scoped Redis identities.
2. Unit tests for ReID decision and mapping persistence.
3. Unit tests for association matching and gating failures.
4. Integration tests for multi-camera same-session identity isolation.
5. Integration tests for occlusion and re-entry outcomes.
6. System tests for crowded classroom sequence identity stability.

### Required Evidence Artifacts

1. `ci_evidence/production/wave3/identity_scope_contract.md`
2. `ci_evidence/production/wave3/reid_mapping_examples.json`
3. `ci_evidence/production/wave3/id_switch_metrics.json`
4. `ci_evidence/production/wave3/occlusion_reentry_report.md`
5. `ci_evidence/production/wave3/interpolation_association_report.md`

## Wave 4: Pose Runtime Integrity, RTMPose Correctness, And Temporal Truth

### Objectives

1. Fix RTMPose Triton config correctness risks.
2. Make pose batch behavior robust under partial failure.
3. Define canonical persisted pose streams for downstream behavior features.
4. Validate temporal pose quality on representative real data.

### Scope Of Work

1. Validate and correct RTMPose Triton IO naming and warmup config mismatch.
2. Clarify production runtime scope:
   - Triton runtime path is production authority
   - local ONNX/OpenVINO/TensorRT claims are non-production unless implemented
3. Normalize pose input size semantics (`height/width`) and map-back consistency.
4. Add batch partial-success handling:
   - preserve successful crop results
   - record per-crop failures
   - avoid frame-level discard for one failed crop
5. Measure and reduce serial fallback risk from batch failure path.
6. Define canonical pose streams:
   - `raw_keypoints`
   - `smoothed_keypoints`
   - `display_keypoints`
7. Persist stream provenance and confidence semantics.
8. Add temporal quality validations:
   - jitter
   - missing keypoint windows
   - confidence stability
9. Measure disk frame IO overhead and decide in-memory optimization path.
10. Add behavior-ready pose feature primitives for Wave 5.

### Deliverables

1. RTMPose config validation checks and tests.
2. Batch partial-success handling implementation.
3. Pose stream contract doc with semantic rules.
4. Real-data jitter/quality benchmark artifacts.
5. Pose fallback telemetry counters.

### Acceptance Gates

1. RTMPose config and warmup pass deterministic validation checks.
2. Batch partial failures do not erase successful detections.
3. Pose stream semantics are explicit and versioned.
4. Real-data jitter and missing-window thresholds pass agreed limits.
5. Fallback frequency and latency are measured and bounded.

### Required Tests

1. Unit tests for Triton IO config validation.
2. Unit tests for shape/name mismatch handling.
3. Unit tests for partial-success batch behavior.
4. System tests for jitter reduction on representative videos.
5. Performance tests for high-person-count pose load.

### Required Evidence Artifacts

1. `ci_evidence/production/wave4/rtmpose_config_validation.txt`
2. `ci_evidence/production/wave4/pose_batch_partial_success.json`
3. `ci_evidence/production/wave4/pose_fallback_metrics.json`
4. `ci_evidence/production/wave4/pose_stream_contract.md`
5. `ci_evidence/production/wave4/pose_jitter_real_data_report.md`

## Wave 5: Typed Temporal Sequence Store And Behavior Feature Layer

### Objectives

1. Move from frame-centric processing to temporal-first behavior foundation.
2. Build typed sequence records suitable for ST-GCN/transformer/contrastive pipelines.
3. Implement first behavior feature set with clear ontology and versioning.

### Scope Of Work

1. Add typed temporal sequence storage with idempotency:
   - identity scope fields
   - frame/timestamp fields
   - lifecycle fields
   - pose stream fields
   - feature vector fields
   - event identity fields
2. Add per-student rolling memory buffers with explicit retention policy.
3. Define behavior ontology v1:
   - feature names
   - units/ranges
   - window semantics
   - missing-data semantics
4. Implement v1 feature extraction set:
   - head direction proxy
   - head angular velocity
   - glance duration
   - repeated glance count
   - wrist velocity
   - wrist disappearance duration
   - motion entropy
   - posture instability
   - torso orientation variation
   - neighbor directional overlap
   - synchronized movement indicator
5. Add temporal anomaly primitives:
   - change-point
   - drift
   - repeated-pattern
6. Add sequence export pipeline for research readiness.
7. Add interaction graph foundation schema for future graph reasoning.

### Deliverables

1. Typed sequence schema and migration artifacts.
2. Behavior ontology v1 specification.
3. Feature extraction service with versioned outputs.
4. Temporal anomaly primitive outputs with confidence and evidence windows.
5. Export manifests for model training pipelines.

### Acceptance Gates

1. Sequence records are typed, deterministic, and idempotent.
2. Feature extraction uses canonical identity and canonical timestamps.
3. Missing data produces explicit missing-state outputs, not fake behavior values.
4. Export manifests are reproducible and include schema/ontology versions.
5. Temporal anomaly primitives operate on sequence windows, not isolated frames.

### Required Tests

1. Unit tests for sequence schema constraints and dedup keys.
2. Unit tests for each behavior feature.
3. Unit tests for missing-data handling.
4. Integration tests from pose records to sequence records to feature outputs.
5. Integration tests for export pipeline manifests.
6. Scenario tests with labeled temporal behavior windows.

### Required Evidence Artifacts

1. `ci_evidence/production/wave5/sequence_schema_contract.md`
2. `ci_evidence/production/wave5/behavior_ontology_v1.md`
3. `ci_evidence/production/wave5/feature_output_samples.json`
4. `ci_evidence/production/wave5/temporal_anomaly_primitives_report.md`
5. `ci_evidence/production/wave5/export_manifest.json`

## Wave 6: Observability Trust, Benchmark Integrity, And Scientific Rigor

### Objectives

1. Remove synthetic trust gaps in telemetry and benchmark logic.
2. Make backend metrics authoritative and faithfully represented in frontend.
3. Add inferential rigor and repeatability to performance validation.

### Scope Of Work

1. Replace hardcoded telemetry source availability with real probes.
2. Define telemetry retention and persistence policy with operational metrics.
3. Add event identity and dedup invariants (`event_id + session scope`).
4. Remove frontend KPI zero-overwrite/masking behavior.
5. Remove benchmark self-baselining pass path.
6. Require explicit baseline run and candidate run in comparisons.
7. Add statistical reporting:
   - repeated runs
   - confidence intervals
   - significance tests or equivalent inference method
   - variance and repeatability
8. Separate mock/simulated benchmarks from real-system acceptance reports.
9. Improve machine-structured logs for runtime/queue/model events.

### Deliverables

1. Probe-backed telemetry readiness outputs.
2. Event dedup contracts and enforcement.
3. Updated benchmark validation logic that fails without external baseline.
4. Statistical summary fields in benchmark artifacts.
5. Frontend KPI truth-aligned behavior.

### Acceptance Gates

1. Telemetry readiness does not report synthetic availability.
2. Benchmark cannot pass without explicit baseline candidate comparison.
3. Statistical outputs are present for acceptance runs.
4. Frontend displays backend truth (null/unknown/zero are distinct).
5. Duplicate event replay does not corrupt metrics.

### Required Tests

1. Unit tests for readiness probe logic.
2. Unit tests for event dedup/idempotency.
3. Unit tests for benchmark baseline requirement enforcement.
4. Unit tests for statistical summary generation.
5. Frontend tests for KPI rendering truth.
6. System tests for telemetry integrity under live load.

### Required Evidence Artifacts

1. `ci_evidence/production/wave6/telemetry_probe_report.json`
2. `ci_evidence/production/wave6/event_dedup_report.md`
3. `ci_evidence/production/wave6/benchmark_baseline_required_report.md`
4. `ci_evidence/production/wave6/statistical_repeatability_report.md`
5. `ci_evidence/production/wave6/frontend_kpi_truth_report.md`

## Wave 7: API/WS Contract Governance And Forensic Behavior Debug UX

### Objectives

1. Stop REST/WS contract drift.
2. Remove broad serializer exposure in public APIs.
3. Consolidate websocket client behavior.
4. Provide unified forensic trace view for behavior debugging.

### Scope Of Work

1. Build one machine-readable registry for REST and WS payload contracts.
2. Enforce schema versions across message families.
3. Replace `fields='__all__'` in public serializers with explicit fields.
4. Consolidate frontend websocket creation path and reconnection behavior.
5. Define artifact source authority and fallback policy (DB/filesystem/cache).
6. Implement forensic trace UX linking:
   - event
   - track identity
   - lifecycle
   - pose stream
   - feature output
   - anomaly primitive
   - artifact
   - benchmark/profile context
7. Measure API/frontend bottlenecks and document thresholds.

### Deliverables

1. Unified API+WS schema registry and generated/validated TS types.
2. Serializer hardening changes with contract tests.
3. Consolidated websocket hook/client behavior.
4. Artifact source authority policy doc.
5. Forensic trace UI flow and integration tests.

### Acceptance Gates

1. REST and WS contracts are governed by one source of truth.
2. Public serializers are explicit-field and contract-tested.
3. Websocket schema version drift is detectable and test-covered.
4. Forensic UI trace provides end-to-end debugging path.
5. API/frontend performance bottlenecks are measured and reported.

### Required Tests

1. Contract tests for registry schema conformance.
2. Serializer exposure tests.
3. WS contract/version tests.
4. Frontend unit tests for websocket manager.
5. Frontend e2e tests for forensic trace workflow.
6. API performance tests for timeline/artifact endpoints.

### Required Evidence Artifacts

1. `ci_evidence/production/wave7/api_ws_registry.json`
2. `ci_evidence/production/wave7/serializer_hardening_report.md`
3. `ci_evidence/production/wave7/ws_version_compat_report.md`
4. `ci_evidence/production/wave7/artifact_source_policy.md`
5. `ci_evidence/production/wave7/forensic_trace_e2e_report.md`
6. `ci_evidence/production/wave7/api_frontend_perf_report.md`

## Wave 8: Final Acceptance, Sign-Off, And Paper Closure

### Objectives

1. Close outstanding M6/M7/M8 acceptance work with real evidence.
2. Produce final profile matrix and ranking.
3. Update paper maturity claims only after implementation evidence exists.

### Scope Of Work

1. Complete dynamic-batch TensorRT/Triton sign-off.
2. Execute full profile matrix:
   - baseline
   - throughput_guardrails
   - live_latency_first
3. Run representative offline validation on production-like datasets.
4. Run representative live soak and fault-injected validation.
5. Execute final automated suites (backend/frontend/system/performance/resilience/contract).
6. Close or explicitly defer remaining strict xfail scaffolds.
7. Complete hash parity and stash/sync hygiene.
8. Update paper coverage and traceability artifacts with real closure evidence.
9. Add final benchmark and RTMPose figures from accepted artifacts.

### Deliverables

1. Final profile matrix outputs and ranking report.
2. Dynamic-batch model/config sign-off report.
3. Offline/live acceptance gate reports.
4. Full test gate pass/fail artifact set.
5. Paper coverage matrix and traceability updates tied to evidence files.

### Acceptance Gates

1. Partial/missing maturity rows are closed or explicitly deferred with rationale.
2. Production evidence is reproducible and attached.
3. Real-system validations are clearly separated from mock/simulated tests.
4. Final maturity claims match validated implementation state.

### Required Tests

1. Full backend suites by category.
2. Frontend unit + e2e smoke.
3. Real Triton/GPU required validation subset.
4. Benchmark/profile matrix runners.
5. Boundary verifier and health snapshots.

### Required Evidence Artifacts

1. `ci_evidence/production/wave8/final_profile_matrix_results.json`
2. `ci_evidence/production/wave8/profile_ranking_report.md`
3. `ci_evidence/production/wave8/dynamic_batch_signoff.md`
4. `ci_evidence/production/wave8/offline_validation_report.md`
5. `ci_evidence/production/wave8/live_soak_report.md`
6. `ci_evidence/production/wave8/final_test_gate_results.md`
7. `ci_evidence/production/wave8/xfail_closure_report.md`
8. `ci_evidence/production/wave8/paper_traceability_update_report.md`
9. `ci_evidence/production/wave8/hash_sync_report.md`

## Cross-Wave Hard Blocking Dependencies

1. Wave 3 blocks Wave 5 behavior semantics maturity.
2. Wave 4 blocks Wave 5 feature fidelity and Wave 6 benchmark trust.
3. Wave 6 blocks final scientific rigor claims.
4. Wave 7 blocks contract maturity and forensic reviewability claims.
5. Wave 8 cannot sign off before Waves 1 through 7 gates are either passed or formally deferred.

## Definition Of “Mature Enough” For PR Closure

A PR may claim maturity closure only when all conditions are met:

1. Identity-temporal integrity is validated on representative live and offline scenarios.
2. Source timestamp and drop accounting are trustworthy.
3. RTMPose runtime/config paths are validated and stable.
4. Typed temporal sequence and behavior feature layer is running with versioned schema.
5. Observability and benchmark outputs are trustworthy and statistically grounded.
6. API/WS contracts are governed and drift-tested.
7. Final acceptance evidence is attached and reproducible.
8. Paper claims are updated to match validated implementation state.
