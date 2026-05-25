# Feature Specification: Production Behavioral Intelligence Maturity Closure

**Feature Branch**: `010-behavioral-maturity-closure`  
**Created**: 2026-05-25  
**Status**: Draft  
**Input**: User description: "Production Behavioral Intelligence Maturity Closure Specification"

## Clarifications

### Session 2026-05-25

- Q: What policy should ReID use to turn candidate matches into canonical identity decisions? -> A: Conservative auto-alias: ReID creates a canonical alias only when score, camera scope, and lifecycle continuity all pass; otherwise it records an unresolved candidate.
- Q: What acceptance threshold model should backpressure use? -> A: Operational SLO gates: require numeric thresholds for queue depth, p95 queue wait, timeout rate, and drop rate per mode.
- Q: What temporal sequence retention policy should production use? -> A: Long raw retention: keep all raw temporal sequence records indefinitely unless manually soft-purged or archived.
- Q: Who can access forensic traces and long-retained raw temporal sequences? -> A: Basic authenticated access: any authenticated production user with dashboard access can view traces and raw sequences.
- Q: What benchmark rigor policy should acceptance use? -> A: Hybrid confidence/statistical policy: production acceptance uses confidence-gated repeated real runs, while paper/research benchmark claims require effect sizes, p-values or nonparametric tests, and power notes.
- Q: What initial per-mode backpressure SLO thresholds should the spec use for acceptance tests? -> A: Balanced thresholds: live queue depth <=120 frames/camera, live p95 queue wait <=1000ms, live timeout rate <=2%, live drop rate <=5%; offline queue depth <=2000 frames/job, offline p95 queue wait <=30s, offline timeout rate <=2%, and offline decoded-frame drop rate must be 0%.
- Q: What minimum representative validation dataset should the spec require for maturity acceptance? -> A: Balanced dataset: at least 3 offline classroom videos and 2 live/RTSP streams covering normal operation, crowded crossings, occlusion/re-entry, pose partial failures, and RTSP disconnect/reconnect.
- Q: Who should be allowed to manually purge or archive raw temporal sequence records? -> A: Dashboard users: any authenticated production dashboard user can purge or archive records they can view, and every action must be audit logged.
- Q: What should "purge" mean for raw temporal sequence records during maturity closure? -> A: No delete: purge means soft-delete/archive only; physical deletion is not supported during maturity closure.
- Q: What minimum repeated-run count should production benchmark acceptance require? -> A: Balanced repetition: at least 5 baseline runs and 5 candidate runs per profile/input.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Govern Production Runtime Policy (Priority: P1)

As a production operator, I need one authoritative production runtime policy so that live and offline processing run with predictable inference routing, clear mode selection, and no conflicting deployment assumptions.

**Why this priority**: All later maturity work depends on consistent production mode selection, endpoint routing, and health validation.

**Independent Test**: Can be fully tested by selecting live mode and offline mode separately in the production environment configuration and verifying that only the selected mode is treated as active while the other mode receives no production inference traffic.

**Acceptance Scenarios**:

1. **Given** production is configured for live mode, **When** startup validation runs, **Then** the live endpoint is validated as active and the offline endpoint is reported as inactive for production traffic.
2. **Given** production is configured for offline mode, **When** startup validation runs, **Then** the offline endpoint is validated as active and the live endpoint is reported as inactive for production traffic.
3. **Given** production is configured with an unsupported mode, **When** startup validation runs, **Then** startup is rejected with a machine-readable diagnostic.
4. **Given** production documentation, environment examples, and scripts are reviewed, **When** reviewers compare endpoint and runtime mode semantics, **Then** all sources agree on dual configured endpoints and single active runtime mode.

---

### User Story 2 - Stabilize Ingestion And Queue Control (Priority: P1)

As an operator and reviewer, I need ingestion, queue routing, backpressure, and RTSP recovery to be observable and deterministic so that live and offline video processing remain trustworthy under unstable streams and load spikes.

**Why this priority**: Behavioral analysis depends on source timestamp truth, recoverable live streams, and queue behavior that does not hide overload or silently drop context.

**Independent Test**: Can be tested by running live and offline processing under controlled disconnects, timeouts, and queue pressure across the representative validation dataset while verifying timestamp propagation, queue wait metrics, reconnect states, and drop accounting.

**Acceptance Scenarios**:

1. **Given** live processing is queued, **When** a worker starts the task, **Then** queue wait duration, queue entry time, and pickup delay are recorded.
2. **Given** a live stream disconnects, **When** reconnect policy is applied, **Then** state transitions progress through explicit recovery or failure states.
3. **Given** ingestion load exceeds configured balanced per-mode SLO limits, **When** live queue depth exceeds 120 frames/camera, live p95 queue wait exceeds 1000ms, live timeout rate exceeds 2%, live drop rate exceeds 5%, offline queue depth exceeds 2000 frames/job, offline p95 queue wait exceeds 30s, offline timeout rate exceeds 2%, or offline decoded-frame drop rate exceeds 0%, **Then** the system takes an observable degradation action and records the reason.
4. **Given** a frame or event is dropped, **When** telemetry is inspected, **Then** the drop reason, timestamp, stream context, and queue context are present.

---

### User Story 3 - Stabilize Identity And Temporal Continuity (Priority: P1)

As a behavioral analysis reviewer, I need stable student identity and lifecycle continuity so that temporal behavior, anomaly windows, and sequence exports are not corrupted by camera cross-talk, ID switches, or incorrect interpolation.

**Why this priority**: Identity-temporal integrity is the largest blocker to reliable behavioral intelligence and sequence learning readiness.

**Independent Test**: Can be tested with representative crowded crossing scenes, occlusion/re-entry clips, and long sessions that measure ID switches, re-entry recovery, lifecycle persistence, and interpolation correctness.

**Acceptance Scenarios**:

1. **Given** two cameras run in the same session, **When** they generate the same local track identifier, **Then** their identities remain isolated by session and camera scope.
2. **Given** a student leaves and re-enters view, **When** re-identification succeeds, **Then** the canonical identity decision is persisted with score, threshold, provenance, and auditability.
3. **Given** a student becomes temporarily occluded, **When** tracking resumes, **Then** lifecycle state changes are persisted and visible in timelines and artifacts.
4. **Given** two students cross paths, **When** interpolation is required, **Then** association uses identity-aware matching and rejects low-confidence bridges.

---

### User Story 4 - Make Pose Runtime Behavior-Grade (Priority: P1)

As a machine-learning and paper reviewer, I need pose inference, pose stream semantics, and pose temporal quality to be validated so that pose becomes a reliable substrate for behavior features instead of only an overlay artifact.

**Why this priority**: Behavioral feature quality depends on correct pose model IO, reliable batch behavior, canonical pose streams, and measured temporal quality.

**Independent Test**: Can be tested by validating model bindings, running high-person-count pose workloads, injecting partial crop failures, and measuring jitter and missing-joint windows on the representative validation dataset.

**Acceptance Scenarios**:

1. **Given** pose runtime starts, **When** model configuration validation runs, **Then** input/output names, dimensions, warmup configuration, and model readiness are verified.
2. **Given** one crop in a pose batch fails, **When** batch processing completes, **Then** successful student pose outputs are preserved and the failed crop is recorded separately.
3. **Given** pose outputs are persisted, **When** downstream analysis reads them, **Then** raw, smoothed, and display pose streams have clear provenance and intended usage.
4. **Given** a representative classroom clip, **When** pose quality validation runs, **Then** jitter, missing-joint windows, confidence stability, and fallback rate are reported.

---

### User Story 5 - Establish Temporal Behavior Features (Priority: P2)

As a research and product stakeholder, I need typed temporal sequences and behavior feature outputs so that the system can support behavior analytics, anomaly primitives, and future learning pipelines.

**Why this priority**: This is the transition from frame-level video analytics to temporal behavioral intelligence, but it depends on identity and pose stability.

**Independent Test**: Can be tested by processing representative video and verifying deterministic sequence records, behavior ontology versioning, feature vectors, missing-data semantics, and reproducible exports.

**Acceptance Scenarios**:

1. **Given** a student track has stable identity and pose data, **When** sequence generation runs, **Then** a deterministic temporal sequence is produced with identity, timestamps, lifecycle, pose stream reference, and event identity.
2. **Given** a student has missing pose or visibility windows, **When** feature extraction runs, **Then** missing data is explicitly represented rather than fabricated.
3. **Given** temporal windows are available, **When** behavior feature extraction runs, **Then** head, wrist, torso, motion, interaction, and attention-deviation features are produced with ontology version metadata.
4. **Given** sequence exports are requested, **When** export completes, **Then** exported datasets include schema version, ontology version, quality filters, split metadata, and reproducibility metadata.

---

### User Story 6 - Make Observability And Benchmarks Trustworthy (Priority: P2)

As an operator, reviewer, and researcher, I need telemetry and benchmark reports to reflect real system behavior so that production readiness and scientific claims are not based on synthetic availability, self-baselining, or frontend masking.

**Why this priority**: Maturity claims are invalid if telemetry or benchmark outputs can falsely report success.

**Independent Test**: Can be tested by disabling telemetry sources, replaying duplicate events, comparing explicit baseline and candidate runs, and verifying frontend metrics distinguish unknown, unavailable, and measured zero states.

**Acceptance Scenarios**:

1. **Given** a telemetry source is unavailable, **When** readiness is reported, **Then** the state is shown as unavailable or unknown rather than synthetic available.
2. **Given** duplicate runtime events are replayed, **When** event ingestion runs, **Then** metrics are not inflated.
3. **Given** a benchmark comparison lacks an explicit baseline, **When** the report is generated, **Then** the comparison is rejected.
4. **Given** backend metrics contain null, unavailable, and zero values, **When** the frontend displays them, **Then** the user can distinguish each state.
5. **Given** a benchmark claim is used for production acceptance, **When** reviewers inspect the report, **Then** at least 5 baseline runs and 5 candidate runs per profile/input, confidence intervals, variance, and pass/fail thresholds are present.
6. **Given** a benchmark claim is used in paper or research claims, **When** reviewers inspect the report, **Then** effect size, p-value or nonparametric test result, and power notes are present.

---

### User Story 7 - Govern API, WebSocket, Artifacts, And Forensic Review (Priority: P3)

As a frontend developer, API reviewer, and forensic investigator, I need governed contracts and an end-to-end trace view so that behavior evidence can be inspected without schema drift, broad field exposure, or manual artifact correlation.

**Why this priority**: Contract governance and traceability are required for maintainability and paper review, but they depend on stable runtime data.

**Independent Test**: Can be tested by validating REST and WebSocket payloads against one registry, checking public serializer exposure, and walking from a behavior event to track identity, pose stream, artifact, and benchmark context.

**Acceptance Scenarios**:

1. **Given** a REST or WebSocket payload is emitted, **When** contract validation runs, **Then** it conforms to the canonical schema registry and version policy.
2. **Given** a public serializer is reviewed, **When** exposure tests run, **Then** only explicitly approved fields are exposed.
3. **Given** a behavior event exists, **When** a reviewer opens the forensic trace, **Then** event, track, lifecycle, pose stream, feature output, artifact, and benchmark context are connected.
4. **Given** an artifact source is degraded, **When** the artifact endpoint responds, **Then** source authority and fallback behavior are visible.

---

### User Story 8 - Close Final Evidence And Paper Traceability (Priority: P3)

As a thesis reviewer and release owner, I need final acceptance evidence and paper traceability to match validated implementation state so that maturity claims are reproducible and defensible.

**Why this priority**: This completes the release and paper review loop after the engineering waves produce evidence.

**Independent Test**: Can be tested by running the final profile matrix, representative validation, full acceptance gates, xfail closure review, and paper traceability update review.

**Acceptance Scenarios**:

1. **Given** final validation is requested, **When** acceptance runs complete, **Then** real and mock executions are clearly separated.
2. **Given** a maturity row remains partial or missing, **When** paper closure is reviewed, **Then** the row is closed with evidence or formally deferred with rationale.
3. **Given** benchmark claims are included in paper artifacts, **When** reviewers inspect them, **Then** baseline, candidate, profile, hardware, data, and repeatability metadata are attached.
4. **Given** final sign-off is requested, **When** evidence artifacts are inspected, **Then** deployment, runtime, benchmark, telemetry, and behavioral maturity claims match validated system behavior.

### Edge Cases

- Active runtime mode is missing, misspelled, or conflicts with endpoint availability.
- Both live and offline endpoints are reachable, but production mode should consume only one.
- Selected endpoint is healthy but required model readiness fails.
- Live stream source lacks camera-generated timestamps.
- Queue workers are running but bound to unexpected queues.
- Reconnect succeeds after repeated failures but produces a timestamp gap.
- A frame is dropped during persistence after successful inference.
- Two cameras in one session produce identical local track identifiers.
- ReID confidence is near threshold or conflicts with camera scope or lifecycle continuity; the system records an unresolved candidate instead of creating a canonical alias.
- Interpolation confidence is too low during crossing or occlusion.
- Pose batch has mixed success and failure results.
- Pose stream smoothing hides a scientifically meaningful raw motion event.
- Feature extraction receives incomplete pose or identity data.
- Telemetry source is down while cached or stale metrics exist.
- Benchmark run attempts to compare a candidate against itself.
- Frontend receives unknown or future schema versions.
- Raw temporal sequence storage grows indefinitely unless operators apply an explicit manual soft-purge or archival decision.
- Any authenticated production user with dashboard access can view forensic traces and raw temporal sequences.
- Any authenticated production dashboard user soft-purges or archives raw temporal sequence records they can view; the action must create audit evidence, retain tombstones/recovery metadata, and must not physically delete maturity evidence.
- Paper claims are updated before implementation evidence exists.

### Mandatory Runtime Scenarios *(video/inference features)*

- **Live Stream Scenario**: A live camera stream is processed in live runtime mode using the live inference endpoint, live queue routing, explicit queue wait telemetry, source/ingest/processing/persistence timestamps, RTSP reconnect states, drop accounting, identity-scoped tracks, pose outputs, behavior feature windows, runtime telemetry, and frontend traceability. Independent validation uses real model weights and live or production-like raw stream data.
- **Offline Video Processing Scenario**: An uploaded or raw video is processed in offline runtime mode using the offline inference endpoint, offline queue routing, deterministic batch behavior, persisted tracks, pose artifacts, sequence records, feature outputs, benchmark artifacts, and playback outputs. Independent validation uses real model weights and representative raw video data.
- **Frontend-Backend Wiring**: User-visible runtime state must reflect mode selection, job/session state, queue state, telemetry readiness, identity lifecycle, pose streams, behavior features, artifact availability, and forensic trace links through governed REST and WebSocket contracts.
- **Backend-Inference Wiring**: Runtime mode must select the correct live or offline inference endpoint, validate model readiness, enforce timeout/retry/degradation semantics, record model version metadata, and reject production authority from mock or local fallback inference paths.
- **Deployment Boundary**: Development may allow relaxed fallback behavior for local work, but production deployment must be Linux-native, Triton-only, GPU-backed, no Docker runtime dependency, and no sudo operational assumption.

### Live Benchmark Runtime Failure Remediation Requirements

The specification package now treats the observed live benchmark failures as production-blocking runtime integrity findings. These findings include Triton timeout spikes around 3300ms, near-zero GPU utilization with high wall latency, duplicate fallback dispatches, unresolved partial-batch indexes, Celery `FAILURE` states while DB jobs remain `processing`, runtime exception paths bypassing status finalization, retry amplification loops, dynamic batching inefficiency, queue/Python orchestration latency dominance, Triton readiness mismatches, runtime mode ambiguity, missing end-to-end request lineage, missing per-attempt observability, missing inference-path attribution, benchmark causality gaps, and silent partial-success degradation.

#### Retry And Fallback Governance

The runtime MUST classify every retry attempt into one retry type:

- `triton_internal_scheduling`: retry or wait behavior inside Triton scheduling and dynamic batching.
- `app_level_fallback_retry`: application retry after timeout, unresolved batch item, or degraded path.
- `celery_task_retry`: Celery-level task retry after retryable worker failure.
- `degraded_execution_retry`: retry under explicitly degraded runtime policy.
- `partial_batch_retry`: selective retry for unresolved or failed batch items only.

The runtime MUST forbid uncontrolled retry recursion, duplicate unresolved-index fanout, hidden fallback amplification, and retry behavior that cannot be traced back to an original request. Runtime policy MUST define max fallback attempts, max unresolved-index retries, and max degraded retries. Every retry attempt MUST emit and persist `request_id`, `parent_request_id`, `retry_reason`, `retry_type`, `inference_path`, `original_batch_id`, `timestamp`, `queue_wait_ms`, and `gpu_wait_ms`. Retry lineage evidence MUST be exported to `ci_evidence/production/runtime/retry_lineage.json` and `ci_evidence/production/runtime/fallback_resolution_report.md`.

#### Latency Decomposition And GPU Underutilization Governance

Benchmark acceptance MUST decompose wall latency into:

```text
total_latency =
queue_wait +
orchestration +
serialization +
triton_queue +
gpu_compute +
fallback_overhead +
persistence
```

Benchmark evidence MUST measure GPU occupancy, GPU duty cycle, CPU orchestration overhead, queue wait contribution, Python scheduling overhead, serialization overhead, Triton queue delay, model execution time, fallback-path ratio, and idle GPU windows. Low GPU utilization with high wall latency invalidates throughput claims unless orchestration attribution is provided. Required evidence artifacts are `ci_evidence/production/runtime/gpu_utilization_analysis.md`, `ci_evidence/production/runtime/latency_decomposition.json`, and `ci_evidence/production/runtime/orchestration_overhead_report.md`.

#### Partial-Batch Failure Governance

Partial-batch runtime behavior MUST preserve successful items, isolate unresolved items, explicitly tag failed items, and prevent frame-wide invalidation due to a single crop failure. `unresolved_index` handling MUST be selective and bounded by retry ceilings. Per-item retry isolation MUST prevent duplicate fanout for unresolved indexes and MUST record `unresolved_index_origin`, `source_batch_size`, `effective_batch_size`, `retry_generation`, and `inference_path`.

#### Workflow Integrity Governance

Runtime workflow states MUST use the authoritative state machine:

```text
QUEUED -> DISPATCHED -> PROCESSING -> PARTIAL_FAILURE -> FAILED
                                      -> COMPLETED
                                      -> DEGRADED_COMPLETED
```

Celery result state and DB job state MUST be reconciled. Runtime exception paths MUST finalize job status through atomic terminal-state transitions. `FAILURE` while the DB state remains `processing` is forbidden. Reconciliation watchdog jobs MUST produce `ci_evidence/production/runtime/workflow_integrity_report.md` and `ci_evidence/production/runtime/terminal_state_reconciliation.json`.

#### Triton Readiness And Mode Validation Hardening

Startup MUST validate the readiness graph for active runtime mode, endpoint authority, model repository consistency, `config.pbtxt` schema, model warmup, TensorRT engine lineage, and active/inactive endpoint enforcement. Startup MUST fail if Triton readiness mismatch exists, a required endpoint is unavailable, active mode is ambiguous, model version drift is detected, or `required_offline=True required_live=True reason=forced_triton_not_ready`-style contradictory readiness is produced.

#### Inference Path Attribution And Benchmark Causality

Every inference event MUST include `inference_path` with one of `batch`, `single_fallback`, `degraded`, `retry`, or `timeout_recovery`; `batch_id`; `source_batch_size`; `effective_batch_size`; `retry_generation`; and `unresolved_index_origin`. Forensic traceability MUST link request -> queue -> Triton -> fallback -> persistence -> artifact. Benchmark reports MUST timestamp-correlate runtime failures, retry spikes, timeout spikes, queue collapse, GPU idle windows, fallback amplification, and Triton queue pressure. Required artifacts are `ci_evidence/production/runtime/benchmark_causality_report.md`, `ci_evidence/production/runtime/retry_amplification_matrix.json`, and `ci_evidence/production/runtime/timeout_correlation_report.md`.

#### Runtime Remediation Governance Artifacts

The following package artifacts are authoritative for implementation and PR review:

- `runtime_policy.md`: retry taxonomy, retry ceilings, workflow state machine, active-mode fail-closed rules.
- `triton_runtime.md`: Triton readiness graph, partial-batch salvage, dynamic batching, timeout attribution, TensorRT lineage.
- `queueing.md`: queue wait telemetry, Celery retry boundaries, starvation/collapse detection, backpressure actions.
- `telemetry.md`: per-attempt telemetry fields, inference-path attribution, latency decomposition, benchmark causality events.
- `observability.md`: GPU underutilization root-cause analysis, frontend truth semantics, benchmark causality observability.
- `benchmarking.md`: latency decomposition equation, GPU duty-cycle requirements, retry amplification rejection rules.
- `acceptance_gates.md`: runtime remediation acceptance gates for Waves 2, 4, 6, and 8.
- `evidence_requirements.md`: deterministic machine-readable artifact paths and invalid evidence conditions.
- `forensic_debugging.md`: request-to-artifact traceability and fallback/retry forensic inspection.
- `deployment.md`: startup readiness validation and active/inactive endpoint enforcement.
- `risk_register.md`: production and scientific risks added by retry/fallback/GPU/workflow failures.
- `production_readiness_checklist.md`: release-blocking runtime remediation checklist.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST preserve dual configured inference endpoints: one for live workloads and one for offline workloads.
- **FR-002**: System MUST allow only one production execution mode to be active at runtime, selected by environment configuration as `live` or `offline`.
- **FR-003**: System MUST prevent inactive-mode endpoints from receiving production inference traffic, scheduler requests, or task routing.
- **FR-004**: System MUST reject production startup when runtime mode is invalid, selected endpoint is unavailable, model readiness fails, profile settings conflict, or required production dependencies are unavailable.
- **FR-005**: System MUST expose active runtime mode, selected endpoint, model readiness, profile, and health status in diagnostics and telemetry.
- **FR-006**: System MUST maintain aligned production environment examples, deployment runbooks, worker startup guidance, health validation guidance, and benchmark configuration guidance.
- **FR-007**: System MUST distinguish development fallback behavior from production-authority behavior.
- **FR-008**: System MUST define one canonical workload queue routing policy covering live control, offline control, model-specific work, retry behavior, priority, overflow, and dead-letter handling.
- **FR-009**: System MUST record live and offline queue wait duration, enqueue time, dequeue time, worker pickup delay, and starvation indicators.
- **FR-010**: System MUST classify task failures as retryable, fatal, degraded, fallback-routed, or dropped.
- **FR-011**: System MUST implement an explicit live stream reconnect lifecycle with connected, disconnected, retry wait, reconnecting, recovered, failed, and stopped states.
- **FR-012**: System MUST preserve source, ingest, queue, processing, and persistence timestamps for live processing.
- **FR-013**: System MUST persist drop reasons for decode failure, stale discard, timeout fallback, downstream failure, queue overflow, backpressure discard, and persistence failure.
- **FR-014**: System MUST define balanced initial per-mode operational SLO gates and MUST take observable backpressure actions when any gate exceeds its threshold: live queue depth <=120 frames/camera, live p95 queue wait <=1000ms, live timeout rate <=2%, live drop rate <=5%, offline queue depth <=2000 frames/job, offline p95 queue wait <=30s, offline timeout rate <=2%, and offline decoded-frame drop rate = 0%.
- **FR-015**: System MUST scope live identity by session, camera, canonical track, and local track.
- **FR-016**: System MUST prevent session-only identity keys from causing multi-camera cross-talk.
- **FR-017**: System MUST use conservative auto-alias ReID policy: create a canonical alias only when score, camera scope, and lifecycle continuity all pass; otherwise persist an unresolved candidate with original track, candidate canonical identity, score, threshold, confidence, and provenance.
- **FR-018**: System MUST persist tracking lifecycle states as active, occluded, reidentified, lost, and ended.
- **FR-019**: System MUST replace index-order interpolation association with identity-aware matching and confidence gating.
- **FR-020**: System MUST expose identity quality metrics including ID switch count, fragmentation rate, re-entry recovery rate, occlusion recovery rate, association quality, and persistence duration.
- **FR-021**: System MUST validate pose runtime configuration for model input names, output names, dimensions, warmup configuration, and runtime binding consistency.
- **FR-022**: System MUST support partial-success pose batch behavior where successful student outputs are preserved when another crop fails.
- **FR-023**: System MUST define and persist versioned raw, smoothed, and display pose streams with clear provenance and usage.
- **FR-024**: System MUST report pose temporal quality metrics including jitter, confidence stability, missing-joint windows, key body-part stability, fallback frequency, timeout frequency, crop failure rate, and pose latency.
- **FR-025**: System MUST create deterministic, idempotent temporal sequence records for each canonical student timeline and retain raw temporal sequence records indefinitely unless explicitly soft-purged or archived.
- **FR-026**: System MUST maintain rolling temporal memory for each student with visibility, pose, motion, anomaly, and behavior history.
- **FR-027**: System MUST define a versioned behavioral ontology covering feature names, units, ranges, confidence semantics, missing-data semantics, and temporal window semantics.
- **FR-028**: System MUST produce behavior features for head movement, wrist behavior, motion patterns, torso variation, interaction cues, and attention-deviation primitives.
- **FR-029**: System MUST represent missing or low-confidence behavior inputs explicitly rather than fabricating behavior values.
- **FR-030**: System MUST export reproducible temporal sequence datasets with schema version, ontology version, feature vectors, temporal windows, quality masks, and split metadata.
- **FR-031**: System MUST replace synthetic telemetry readiness with real probe-backed status.
- **FR-032**: System MUST enforce runtime event identity and deduplication to prevent replay corruption and metric inflation.
- **FR-033**: System MUST require explicit baseline and candidate runs for benchmark comparisons.
- **FR-034**: System MUST reject benchmark pass states based on self-baselines, synthetic outputs, or missing comparison references.
- **FR-035**: System MUST apply hybrid benchmark rigor: production acceptance claims require at least 5 baseline runs and 5 candidate runs per profile/input, confidence intervals, variance reporting, stability metrics, and explicit pass/fail thresholds; paper and research benchmark claims additionally require effect sizes, p-values or nonparametric tests, and power notes.
- **FR-036**: System MUST ensure frontend metric displays distinguish unknown, measured zero, unavailable, stale, and valid states.
- **FR-037**: System MUST provide one canonical schema registry covering REST payloads, WebSocket payloads, event payloads, telemetry payloads, and artifact payloads.
- **FR-038**: System MUST enforce schema versioning across runtime and frontend-backend contracts.
- **FR-039**: System MUST expose only explicitly approved public fields in user-facing data contracts.
- **FR-040**: System MUST define artifact source authority, fallback behavior, stale handling, and cache invalidation rules.
- **FR-041**: System MUST provide an end-to-end forensic trace from event to track identity, lifecycle, pose stream, behavior feature, anomaly primitive, artifact, and benchmark context.
- **FR-041a**: System MUST allow any authenticated production user with dashboard access to view forensic traces and raw temporal sequence records.
- **FR-041b**: System MUST allow any authenticated production dashboard user to soft-purge or archive raw temporal sequence records they can view, and MUST audit actor identity, action type, affected scope, reason, timestamp, tombstone identity, recovery reference, and evidence impact.
- **FR-041c**: System MUST NOT physically delete raw temporal sequence records during maturity closure; purge semantics MUST mean soft-delete/archive only.
- **FR-042**: System MUST separate mock, synthetic, CPU, GPU, live, offline, development, and production evidence in all acceptance reports.
- **FR-042a**: System MUST use a balanced representative validation dataset for maturity acceptance containing at least 3 offline classroom videos and 2 live/RTSP streams covering normal operation, crowded crossings, occlusion/re-entry, pose partial failures, and RTSP disconnect/reconnect.
- **FR-043**: System MUST close implemented strict xfail scaffolds or document formal deferment rationale before maturity sign-off.
- **FR-044**: System MUST update paper and thesis traceability only after matching implementation evidence exists.
- **FR-045**: System MUST prevent maturity claims that exceed validated implementation state.
- **FR-046**: System MUST implement authoritative retry taxonomy covering Triton internal scheduling, app-level fallback retry, Celery task retry, degraded execution retry, and partial-batch retry.
- **FR-047**: System MUST enforce retry ceilings for max fallback attempts, max unresolved-index retries, and max degraded retries, and MUST reject uncontrolled retry recursion, duplicate unresolved-index fanout, and hidden fallback amplification.
- **FR-048**: System MUST persist retry lineage for every retry attempt with request ID, parent request ID, retry reason, retry type, inference path, original batch ID, timestamp, queue wait, and GPU wait.
- **FR-049**: System MUST attribute every inference event to an inference path: batch, single_fallback, degraded, retry, or timeout_recovery.
- **FR-050**: System MUST preserve successful batch items, isolate unresolved items, explicitly tag failed items, and forbid frame-wide invalidation due to one failed crop.
- **FR-051**: System MUST implement selective per-item retry isolation for unresolved indexes and MUST bound duplicate dispatch ratio.
- **FR-052**: System MUST decompose latency into queue wait, orchestration, serialization, Triton queue, GPU compute, fallback overhead, and persistence.
- **FR-053**: System MUST measure GPU occupancy, GPU duty cycle, CPU orchestration overhead, Python scheduling overhead, serialization overhead, Triton queue delay, model execution time, fallback-path ratio, and idle GPU windows for benchmark acceptance.
- **FR-054**: System MUST provide flamegraph or perf-style orchestration attribution reports for benchmark runs where wall latency is high or GPU utilization is low.
- **FR-055**: System MUST maintain authoritative workflow states: queued, dispatched, processing, partial failure, failed, completed, and degraded completed.
- **FR-056**: System MUST keep Celery result state and DB job state consistent through exception-safe finalization and atomic terminal-state transitions.
- **FR-057**: System MUST forbid Celery `FAILURE` while the corresponding DB job remains `processing`; reconciliation watchdog jobs MUST repair or fail such divergence.
- **FR-058**: System MUST validate startup readiness graph, endpoint authority, active/inactive mode health, Triton repository consistency, `config.pbtxt`, model warmup, and TensorRT engine lineage.
- **FR-059**: System MUST fail startup when Triton readiness mismatch, ambiguous active mode, required endpoint unavailability, or model version drift is detected.
- **FR-060**: System MUST propagate structured request lineage from request to queue, Triton attempt, fallback attempt, persistence, artifact, benchmark row, and forensic trace.
- **FR-061**: System MUST correlate benchmark artifacts with runtime causality including timeout spikes, retry spikes, queue collapse, GPU idle windows, fallback amplification, and Triton queue pressure.
- **FR-062**: System MUST reject benchmark throughput claims when low GPU utilization with high wall latency lacks orchestration attribution.
- **FR-063**: System MUST mark benchmark results with unresolved retry amplification as not scientifically trustworthy.
- **FR-064**: System MUST disclose partial-batch silent fallback execution in benchmark evidence and acceptance reports.

### Key Entities *(include if feature involves data)*

- **Runtime Mode**: The selected production execution mode that determines whether live or offline processing is active.
- **Inference Endpoint**: A configured production inference target dedicated to either live workloads or offline workloads.
- **Health Snapshot**: A timestamped record of runtime mode, endpoint readiness, backend health, model-serving readiness, queue connectivity, and production dependency state.
- **Queue Route**: A workload routing rule with queue purpose, priority, retry policy, overflow behavior, and dead-letter semantics.
- **Reconnect Event**: A live stream recovery state transition with timestamp, retry count, cause, and outcome.
- **Drop Event**: A record explaining why a frame or event was not processed or persisted.
- **Identity Scope**: The uniqueness context for a tracked person, including session, camera, canonical track, and local track.
- **Canonical Track**: The stable identity used for temporal memory, sequence records, feature extraction, and forensic traceability.
- **ReID Decision**: An auditable identity decision containing candidate, match, score, threshold, confidence, provenance, and whether the outcome is a canonical alias or unresolved candidate.
- **Tracking Lifecycle State**: The current state of a tracked person within a timeline.
- **Pose Stream**: A versioned representation of keypoints for raw scientific truth, smoothed temporal analysis, or display rendering.
- **Temporal Sequence Record**: An idempotent per-student timeline entry that connects identity, timestamps, lifecycle, pose, feature, and event identity.
- **Behavior Ontology**: The versioned definition of behavioral feature meanings, units, valid ranges, confidence semantics, and missing-data semantics.
- **Behavior Feature Window**: A temporal interval used to compute interpretable features such as glance duration, wrist visibility, motion entropy, or interaction cues.
- **Anomaly Primitive**: A basic temporal signal such as change-point, drift, repeated pattern, instability, or attention deviation.
- **Benchmark Run**: A reproducible validation run with profile, input set, runtime mode, environment, metrics, and repeatability metadata.
- **Benchmark Claim**: A benchmark conclusion classified as production acceptance, paper, or research; paper and research claims require stronger statistical evidence than production acceptance claims.
- **Contract Registry**: The authoritative schema source for external payloads and frontend-backend communication.
- **Forensic Trace**: A linked evidence path from runtime event through identity, pose, behavior, artifact, and benchmark context.
- **Retry Lineage**: A persisted parent-child request graph for retry attempts, fallback dispatches, timeout recovery, and partial-batch retry paths.
- **Inference Path**: The attributed execution path for an inference event, including batch, single fallback, degraded, retry, or timeout recovery.
- **Latency Decomposition**: A benchmark record that breaks wall time into queue wait, orchestration, serialization, Triton queue, GPU compute, fallback overhead, and persistence.
- **Workflow State**: The authoritative processing state spanning Celery and DB job status: queued, dispatched, processing, partial failure, failed, completed, or degraded completed.
- **Benchmark Causality Report**: A timestamp-correlated report connecting benchmark metrics to runtime failures, retry spikes, timeouts, queue collapse, GPU idle windows, fallback amplification, and Triton queue pressure.

### Assumptions

- Production inference is Triton-only because the user specified Triton as the production authority.
- Dual inference endpoints remain configured, but production execution consumes only the endpoint for the selected runtime mode.
- Cross-camera identity matching is not required for first maturity closure; cross-camera isolation is required.
- Local fallback inference may remain for development, but it cannot satisfy production acceptance.
- Behavioral anomaly primitives are interpretable foundations, not final cheating verdicts.
- Paper and thesis updates are downstream of implementation evidence, not substitutes for implementation evidence.
- Raw temporal sequence records are treated as long-lived research and forensic assets; manual soft-purge or archival decisions require explicit operator action and audit evidence.
- Forensic traces and raw temporal sequence records are available to authenticated production dashboard users.
- Authenticated production dashboard users may soft-purge or archive raw temporal sequence records they can view; these actions require immutable audit evidence and cannot physically delete records during maturity closure.
- Maturity acceptance uses a balanced representative validation dataset with at least 3 offline classroom videos and 2 live/RTSP streams.
- Production benchmark acceptance uses at least 5 baseline runs and 5 candidate runs per profile/input.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of production startup attempts with invalid runtime mode or unavailable active endpoint are rejected before processing begins.
- **SC-002**: 100% of production health snapshots identify active mode, active endpoint, inactive endpoint status, model readiness, backend readiness, queue readiness, and runtime profile.
- **SC-003**: 95% of live stream reconnect test cases produce the expected final state and preserve a complete state transition history.
- **SC-004**: 100% of dropped live frames or events in validation runs include a reason, timestamp context, and stream context.
- **SC-005**: Queue wait metrics are available for 100% of live and offline validation jobs.
- **SC-005a**: Live and offline validation reports include pass/fail results against the balanced initial per-mode SLO gates: live queue depth <=120 frames/camera, live p95 queue wait <=1000ms, live timeout rate <=2%, live drop rate <=5%, offline queue depth <=2000 frames/job, offline p95 queue wait <=30s, offline timeout rate <=2%, and offline decoded-frame drop rate = 0%.
- **SC-006**: Multi-camera same-session validation produces zero identity key collisions.
- **SC-007**: Representative occlusion/re-entry validation reports ID switch count, re-entry recovery rate, occlusion recovery rate, and fragmentation rate for every tested clip.
- **SC-008**: Crowded crossing validation demonstrates rejected low-confidence interpolation associations and reports association quality metrics.
- **SC-009**: Pose batch validation preserves successful student outputs in 100% of mixed success/failure batch scenarios.
- **SC-010**: Pose quality validation reports jitter, missing-joint windows, confidence stability, fallback rate, timeout rate, crop failure rate, and latency for every representative video.
- **SC-011**: 100% of temporal sequence records in validation runs are idempotent and traceable to identity, timestamp, pose stream, and source event.
- **SC-012**: Behavior feature extraction produces explicit missing-data states for 100% of incomplete-input scenarios.
- **SC-013**: Sequence export runs include schema version, ontology version, feature definitions, quality masks, and reproducibility metadata.
- **SC-013a**: Retention validation confirms raw temporal sequence records remain available after export unless an audited manual soft-purge or archive action has hidden them from active views.
- **SC-013b**: Soft-purge/archive validation confirms authenticated production dashboard users can only affect raw temporal sequence records they can view, every action records actor, action, scope, reason, timestamp, tombstone identity, recovery reference, and evidence impact, and no physical deletion occurs during maturity closure.
- **SC-014**: Telemetry readiness reports no synthetic available states in production validation.
- **SC-015**: Duplicate event replay validation results in no metric inflation.
- **SC-016**: 100% of benchmark comparison reports include explicit baseline and candidate references or fail validation.
- **SC-017**: Production benchmark reports include at least 5 baseline runs and 5 candidate runs per profile/input, variance, confidence intervals, stability metrics, and explicit pass/fail thresholds.
- **SC-017a**: Paper and research benchmark claims include effect sizes, p-values or nonparametric test results, and power notes.
- **SC-018**: Frontend validation distinguishes unknown, measured zero, unavailable, stale, and valid metric states in all tested runtime panels.
- **SC-019**: 100% of public REST and WebSocket payload families are covered by a versioned contract registry.
- **SC-020**: Public field exposure tests pass for all reviewed user-facing data contracts.
- **SC-021**: A reviewer can trace at least one validated behavior event from event source to identity, lifecycle, pose stream, behavior feature, artifact, and benchmark context.
- **SC-021a**: Access validation confirms authenticated production dashboard users can view forensic traces and raw temporal sequence records.
- **SC-022**: Final acceptance evidence clearly separates mock vs real execution, CPU vs GPU execution, synthetic vs production telemetry, and live vs offline profiles.
- **SC-022a**: Final acceptance evidence includes at least 3 offline classroom video runs and 2 live/RTSP stream runs covering normal operation, crowded crossings, occlusion/re-entry, pose partial failures, and RTSP disconnect/reconnect.
- **SC-023**: All strict xfail scaffolds related to implemented maturity behavior are removed, converted to passing tests, or formally deferred with rationale.
- **SC-024**: Paper traceability rows are updated only when matching implementation evidence artifacts exist.
- **SC-025**: A release reviewer can verify every maturity claim from attached evidence without relying on undocumented manual knowledge.
- **SC-026**: 100% of retry attempts in validation runs include persisted retry lineage with request ID, parent request ID, retry type, retry reason, inference path, original batch ID, queue wait, and GPU wait.
- **SC-027**: Partial-batch validation preserves 100% of successful items, isolates unresolved indexes, explicitly tags failed items, and reports duplicate dispatch ratio.
- **SC-028**: Benchmark reports include latency decomposition for 100% of accepted live and offline benchmark runs.
- **SC-029**: Any accepted benchmark with GPU utilization below the configured duty-cycle threshold includes orchestration attribution and idle GPU window analysis.
- **SC-030**: Celery/DB reconciliation validation produces zero cases where Celery state is `FAILURE` while DB job state remains `processing`.
- **SC-031**: Startup readiness validation fails for Triton ready mismatch, ambiguous active mode, unavailable required endpoint, config.pbtxt schema failure, warmup failure, or TensorRT engine lineage drift.
- **SC-032**: 100% of inference events in validation runs include inference path, batch ID, source batch size, effective batch size, retry generation, and unresolved-index origin where applicable.
- **SC-033**: Benchmark causality reports timestamp-correlate timeout spikes, retry spikes, queue collapse, GPU idle windows, fallback amplification, and Triton queue pressure for every accepted benchmark profile.
- **SC-034**: Benchmark results with unresolved retry amplification are rejected for scientific maturity claims.
- **SC-035**: Partial-batch fallback execution is disclosed in benchmark artifacts whenever fallback-single or retry inference paths are used.
