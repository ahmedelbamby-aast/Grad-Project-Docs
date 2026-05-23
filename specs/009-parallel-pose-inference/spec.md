# Feature Specification: Parallel Pose Inference

**Feature Branch**: `009-parallel-pose-inference`  
**Created**: 2026-05-23  
**Status**: Draft - Hybrid Offline Validation Required  
**Input**: User description: "Refactor the pose estimation pipeline to run in parallel with the object detection inference pipeline; research and implement high-throughput strategies for NVIDIA GPU and Intel GPU for offline video processing and live streaming; evaluate grayscale conversion gains and impacts; mature the system so both inference pipelines are wired together, handle edge cases and errors, and generate all object and pose artifacts correctly."

## Clarifications

### Session 2026-05-23

- Q: For live stream overload behavior, should the system skip stale frames or preserve every frame with higher latency? -> A: Preserve every frame with a configurable delayed-live buffer, defaulting to 5 minutes, so the frontend consumes persisted frames and inference data with an intentional latency window.
- Q: What validation gate should decide whether grayscale or reduced-color optimization is acceptable? -> A: Both labeled validation metrics and RGB baseline parity on representative project videos.
- Q: Should RTMPose depend on object detections or allow independent pose artifacts when object detection fails? -> A: Hybrid: use valid object detections first, but allow degraded RTMPose artifacts from validated fallback regions when object detections are unavailable.
- Q: What offline pose artifact completeness threshold should fail a job? -> A: Fail the offline job if more than 2% of decoded frames lack required pose status or any continuous pose gap exceeds 10 seconds.
- Q: How should accelerator optimization profiles be selected for operation? -> A: Use config-selectable safe-default profiles with separate live and offline profile choices plus benchmark-recommended overrides.
- Q: How should model execution workers be organized across runtime inference environments? -> A: Each model must have one dedicated Celery worker regardless of whether that model runs through Triton, OpenVINO, or another inference runtime.
- Q: What evidence is required before this feature can be considered complete? -> A: A real-video offline run must pass every inference stage on a hybrid profile using Intel GPU through OpenVINO and NVIDIA GPU through Triton, demonstrate throughput and latency improvement, and export a source-fidelity final video.
- Q: Are GPU telemetry applications optional for the required hybrid evidence run? -> A: No. Install NVIDIA Nsight Systems CLI in an instrumented Triton container validation profile and Intel PresentMon on Windows, then persist successful captures from both in the completion evidence package.
- Q: How should telemetry data serve dashboards and AI/operator clients? -> A: Dashboards use authenticated backend REST/WebSocket APIs; a mandatory, initially read-only MCP server exposes normalized persisted telemetry and controlled diagnostics to authorized AI/operator clients.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Live Dual Inference (Priority: P1)

As an exam monitoring operator, I need live camera sessions to run object detection and pose estimation together so that detections, tracks, pose keypoints, skeleton overlays, and anomaly inputs stay synchronized during delayed-live monitoring.

**Why this priority**: Live monitoring is the most time-sensitive workflow, but this system prioritizes complete inference artifacts over zero-delay viewing by using a controlled presentation delay. The feature is not successful if adding pose estimation blocks object detection, hides live status, or creates inconsistent results.

**Independent Test**: Start a live session with a representative camera stream and real model artifacts, then verify that object detections and pose artifacts are persisted and presented from the same delayed-live frame timeline with visible degraded states when either pipeline becomes unavailable.

**Acceptance Scenarios**:

1. **Given** a healthy live camera and healthy object and pose model artifacts, **When** the operator starts monitoring, **Then** object detections, tracks, pose keypoints, pose overlays, and combined anomaly inputs are produced for every ingestible frame and presented through the configured delayed-live buffer.
2. **Given** pose inference becomes temporarily unavailable while object detection remains healthy, **When** live monitoring continues, **Then** object detection results continue, the pose status is marked degraded, and the session records the failure without corrupting existing artifacts.
3. **Given** object detection becomes temporarily unavailable while pose inference remains healthy, **When** live monitoring continues, **Then** pose processing uses validated fallback person regions when available or is skipped with a clear reason, and no misleading combined artifact is produced.

---

### User Story 2 - Offline Dual Artifact Generation (Priority: P1)

As a reviewer processing uploaded videos, I need offline analysis to produce complete object and pose artifacts so that playback, reports, and downstream behavior analysis use a consistent frame-by-frame record.

**Why this priority**: Offline processing is where full evidence artifacts are expected. Missing pose artifacts or mismatched frame identifiers would make review and validation unreliable.

**Independent Test**: Upload a real representative exam video and run two heavy-workload hybrid validation runs using the full enabled model sets from `.env`: one run with all selected models bound to NVIDIA/Triton, and one run with all selected models bound to Intel GPU/OpenVINO. Verify that all inference stages complete successfully in both runs, object and pose artifacts reconcile, measured throughput and latency improve over each run's unoptimized baseline, and the exported output video preserves source fidelity outside intentional overlays.

**Acceptance Scenarios**:

1. **Given** a valid uploaded video with visible students, **When** processing completes, **Then** every successfully processed frame has traceable object and pose status and the generated artifacts are linked to the same video timeline.
2. **Given** some frames cannot be decoded or are unsuitable for inference, **When** processing completes, **Then** the job reports those frames as skipped or failed with reasons and still produces valid artifacts for the remaining frames.
3. **Given** the pose pipeline fails for a subset of frames, **When** the job completes, **Then** object artifacts remain usable, pose gaps or degraded fallback-region pose artifacts are explicit, and the offline job fails if more than 2% of decoded frames lack required pose status or any continuous pose gap exceeds 10 seconds.
4. **Given** the feature is proposed as complete, **When** release evidence is reviewed, **Then** it includes a successful real-video hybrid run spanning Intel GPU/OpenVINO and NVIDIA GPU/Triton, stage-by-stage pass results, benchmark deltas, and the exported final video quality report.

---

### User Story 3 - Accelerator Throughput Optimization (Priority: P2)

As a system maintainer, I need a repeatable way to benchmark and select high-throughput execution profiles for supported NVIDIA and Intel accelerator environments so that live and offline workloads use the best profile without sacrificing correctness.

**Why this priority**: Parallel inference can improve throughput only when scheduling, batching, backpressure, and accelerator selection are validated against real workload data.

**Independent Test**: Run the benchmark suite on each supported accelerator profile with representative live and offline inputs, then compare throughput, latency, utilization, error rate, and artifact completeness against the baseline.

**Acceptance Scenarios**:

1. **Given** a supported NVIDIA accelerator profile, **When** the benchmark suite runs, **Then** it records baseline and optimized live/offline metrics and identifies the highest-throughput profile that still meets latency and artifact-quality gates.
2. **Given** a supported Intel accelerator profile, **When** the benchmark suite runs, **Then** it records baseline and optimized live/offline metrics and identifies whether throughput, latency, or fallback behavior limits the profile.
3. **Given** an optimized profile fails correctness, stability, or artifact completeness gates, **When** profile selection is evaluated, **Then** the system keeps the last known valid profile and records why the candidate was rejected.
4. **Given** live and offline workloads have different latency and throughput needs, **When** profiles are configured, **Then** the system applies separate live and offline profile choices with safe defaults and benchmark-recommended overrides.
5. **Given** multiple models are enabled across different inference runtimes, **When** live or offline processing runs, **Then** each model has one dedicated Celery worker and one model's backlog or runtime failure does not starve the other model pipelines.
6. **Given** a hybrid offline profile is validated, **When** optimized performance is compared with its unoptimized hybrid baseline on the same real video, **Then** the report shows throughput and latency changes per stage and rejects any profile without measured improvement.

---

### User Story 4 - Grayscale Optimization Decision (Priority: P2)

As a maintainer optimizing video throughput, I need grayscale conversion to be evaluated as a controlled candidate optimization so that any speed gains are accepted only when detection and pose quality remain within defined limits.

**Why this priority**: Grayscale may reduce decode, transfer, or preprocessing cost, but many vision models are trained and validated on color input. It must be treated as an evidence-gated optimization rather than a default assumption.

**Independent Test**: Compare RGB baseline processing against grayscale candidate processing on the same real live and offline inputs, then verify throughput, latency, object quality, pose quality, artifact completeness, and visual review outcomes.

**Acceptance Scenarios**:

1. **Given** a grayscale candidate profile improves throughput but reduces object or pose quality beyond the accepted threshold, **When** optimization results are reviewed, **Then** grayscale remains disabled and the report explains the rejected tradeoff.
2. **Given** a grayscale candidate profile improves throughput and stays within all quality thresholds, **When** the profile is selected, **Then** generated artifacts clearly record the input color mode used for that run.
3. **Given** grayscale processing does not reduce inference work because the selected model still requires color-shaped input, **When** the benchmark report is generated, **Then** the report separates decode, preprocessing, transfer, inference, and artifact-writing costs so the limited gain is visible.
4. **Given** a grayscale or reduced-color candidate passes speed checks, **When** validation is reviewed, **Then** it is accepted only if labeled validation metrics and RGB baseline parity on representative project videos both pass.

---

### User Story 5 - Mature Failure Handling and Evidence (Priority: P3)

As a release owner, I need the dual-pipeline system to handle edge cases, retries, and observability consistently so that the feature is safe to operate and audit in production-like environments.

**Why this priority**: Mature behavior depends on predictable degradation, clear evidence, and traceable artifacts across both inference pipelines.

**Independent Test**: Execute failure-injection and edge-case runs covering camera interruption, corrupt uploads, missing model artifacts, accelerator exhaustion, timeout, empty scenes, crowded scenes, and artifact write failures, then verify outcomes and evidence files.

**Acceptance Scenarios**:

1. **Given** a model artifact is missing or incompatible, **When** live or offline processing starts, **Then** the system fails fast or enters a documented degraded state without silently producing incomplete artifacts as complete.
2. **Given** accelerator memory or queue capacity is exhausted, **When** more frames arrive, **Then** the system applies bounded backpressure with tiered buffering (bounded RAM queue plus bounded disk spill queue), persists queue/spill telemetry, drains buffered frames in order when capacity returns, and keeps session/job state consistent with explicit overflow policy when limits are exceeded.
3. **Given** artifact storage fails during processing, **When** the run ends, **Then** incomplete artifacts are not presented as valid and the final summary identifies what was written, skipped, retried, or failed.

---

### User Story 6 - Governed Telemetry Access (Priority: P3)

As an authorized operator or AI assistant integrator, I need telemetry summaries and evidence to be accessible through the correct governed interface so that dashboards stay responsive and AI-assisted diagnostics cannot bypass security or validation rules.

**Why this priority**: Telemetry is operationally useful only when presentation, diagnostic access, and sensitive-data protections share one normalized source of truth while keeping dashboard and MCP responsibilities isolated.

**Independent Test**: Process a telemetry-producing validation run, retrieve normalized data through dashboard REST/WebSocket endpoints, and verify that the mandatory MCP service exposes equivalent authorized read-only resources and diagnostic tools while unauthorized or mutating operations are rejected and audited.

**Acceptance Scenarios**:

1. **Given** telemetry evidence is persisted and the MCP service is disabled or unavailable, **When** an operator loads dashboard telemetry panels, **Then** summary, timeline, artifact availability, runtime assignment, and benchmark comparison data remain available through backend REST/WebSocket APIs.
2. **Given** the MCP server is enabled for an authorized AI/operator client, **When** the client queries telemetry resources or read-only diagnostic tools, **Then** it receives normalized evidence derived from backend persistence rather than direct unrestricted collector or filesystem access.
3. **Given** an MCP client requests student-identifying data, unrestricted raw files, arbitrary shell execution, arbitrary Nsight arguments, or a state-changing telemetry capture action in the initial MCP release, **When** the request is evaluated, **Then** it is denied, audited, and does not affect ingestion, inference, dashboards, or stored evidence.

---

### Edge Cases

- Live camera stream disconnects, stalls, changes resolution, changes frame rate, or emits duplicate/out-of-order frames.
- Uploaded video is corrupt, variable frame rate, very short, very long, empty, low-light, motion-blurred, or contains no visible students.
- Crowded frames contain overlapping people, partial bodies, occlusions, students entering or leaving the frame, or crossing tracks.
- Object and pose pipelines complete at different speeds for the same frame.
- The delayed-live buffer has not filled yet, grows beyond the configured delay, or must report that presentation is falling behind capture time.
- One pipeline succeeds while the other fails, times out, returns low confidence, or returns no results.
- Object detection is unavailable and RTMPose receives only fallback person regions or no valid region for a frame.
- Accelerator is unavailable, overloaded, out of memory, thermally throttled, or reports degraded health.
- A mandatory telemetry application is missing, fails to start, generates an empty or unreadable capture, or adds unaccounted measurement overhead.
- MCP is disabled, unavailable, disconnected, or receives malformed/unauthorized requests while telemetry ingestion and application dashboards remain healthy.
- MCP requests expose student-identifying content, unbounded telemetry ranges, unauthorized raw trace downloads, shell execution, arbitrary profiler arguments, or state-changing capture commands.
- Live and offline optimization profiles are missing, invalid, unsafe, or accidentally swapped.
- A dedicated model worker is saturated, stopped, misconfigured, routed to the wrong runtime, or configured with zero effective capacity.
- A model moves between inference runtimes and must keep the same model-level worker isolation and artifact contract.
- A hybrid offline validation run silently executes all models on one accelerator or falls back from the required Intel/OpenVINO or NVIDIA/Triton path.
- Exported video differs from source resolution, aspect ratio, timing, frame rate, audio presence, or approved visual fidelity outside intentional overlays.
- Any decode, inference, association, tracking, artifact persistence, overlay render, export, or summary reconciliation stage fails during the required completion run.
- Candidate optimization increases throughput but regresses detection quality, pose quality, live latency, artifact completeness, or review usability.
- Grayscale removes useful color cues for clothing, background separation, skin tone, lighting artifacts, or model features learned from color training data.
- Artifact paths, metadata, or run summaries collide across concurrent jobs or sessions.
- Partial retries create duplicate frame artifacts or inconsistent final counts.
- Offline processing produces more than 2% decoded frames without required pose status or any continuous pose gap longer than 10 seconds.
- Frontend receives stale, partial, or delayed updates during degraded inference.
- Production deployment lacks development-only container services.

### Mandatory Runtime Scenarios *(video/inference features)*

- **Live Stream Scenario**: A live camera session consumes a representative RTSP/WebRTC feed, shares frame identity across object and pose processing, persists every ingestible frame and inference artifact, and presents results through a configurable delayed-live buffer that defaults to 5 minutes. The system records skipped/degraded frames only for explicit decode, health, capacity, or artifact failures and recovers or degrades visibly after stream, model, or accelerator failures. Independent validation must use real model weights and real raw/live test data.
- **Offline Video Processing Scenario**: An uploaded real raw video is decoded into a traceable frame timeline, object and pose processing run in parallel where capacity allows, and completion evidence includes: (1) a hybrid profile run proving Intel/OpenVINO and NVIDIA/Triton assignment in the same run, plus (2) two full-model heavy-workload stress lanes using the selected models from `.env` (all selected models on NVIDIA/Triton in one run and all selected models on Intel/OpenVINO in another run). All artifacts are persisted with shared frame identifiers, and final job output includes playback-ready overlays, raw artifact data, stage pass/fail evidence, performance comparison against unoptimized baselines, a source-fidelity exported video, and a reconciliation summary. Independent validation must use real model weights and real raw video data.
- **Frontend-Backend Wiring**: Users must see clear session/job states for pending, buffering, running, delayed, degraded, partial success, failed, and completed runs. Live updates and offline job status must expose object and pose readiness separately and never imply pose artifacts exist when they were skipped or failed. Telemetry dashboards must obtain normalized summary, timeline, artifact, runtime-assignment, and benchmark-comparison data through authenticated REST/WebSocket interfaces without depending on MCP availability.
- **Backend-Inference Wiring**: The inference boundary must verify model artifact availability, model health, accelerator health, input compatibility, timeout behavior, retry policy, and per-pipeline failure isolation before and during processing. Both direct local inference and served inference paths must produce the same artifact contract.
- **Telemetry Wiring**: The required hybrid validation run must use an instrumented Triton validation container with NVIDIA Nsight Systems CLI installed and active for bounded NVIDIA profiling, Triton metrics capture for low-overhead serving/GPU KPIs, and Intel PresentMon installed and active on Windows together with OpenVINO device/profiling evidence. Missing mandatory telemetry application evidence fails validation.
- **MCP Observability Boundary**: The system must offer a mandatory MCP telemetry server for authorized AI/operator use over normalized persisted evidence. The initial MCP interface is read-only and may expose bounded telemetry resources and read-only diagnostic tools, but must not scrape collectors directly, execute shell commands, accept arbitrary profiler arguments, or start capture jobs.
- **Deployment Boundary**: Development and test environments may use Docker-based dependencies, but production must support native Linux operation without requiring Docker for the production inference service. Each supported deployment mode must document accelerator readiness checks and artifact storage expectations.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST process object detection and pose estimation as parallel pipelines for the same live session or offline video whenever both pipelines are enabled and healthy.
- **FR-002**: System MUST assign shared frame identifiers, timestamps, source metadata, and run identifiers so object artifacts and pose artifacts can be reconciled across both pipelines.
- **FR-003**: System MUST prevent pose pipeline failures from corrupting, blocking, or deleting valid object detection artifacts.
- **FR-004**: System MUST prevent object pipeline failures from producing misleading pose-to-object associations when required object context is unavailable.
- **FR-005**: System MUST generate pose artifacts including keypoints, confidence values, skeleton or overlay data, per-frame pose status, and run-level pose summaries.
- **FR-006**: System MUST generate combined artifacts that link object detections, tracks, and pose outputs only when their frame and identity relationships are valid.
- **FR-006a**: System MUST support RTMPose fallback-region mode for degraded pose artifacts when primary object detections are unavailable, but only from validated person regions such as last-known compatible regions, reviewer-approved regions, or whole-frame single-person regions that are explicitly marked as degraded.
- **FR-007**: System MUST support both live stream processing and offline uploaded video processing with the same artifact contract.
- **FR-008**: System MUST expose separate status for object detection, pose estimation, combined association, artifact persistence, and final run reconciliation.
- **FR-009**: System MUST apply bounded queues, backpressure, and a configurable delayed-live buffer so live streams prioritize every-frame processing while making buffer growth, presentation delay, spill-to-disk status, and any explicit overflow frame loss visible.
- **FR-009a**: System MUST implement tiered overload buffering with a bounded in-memory queue and a bounded disk-backed spill queue (preferably SSD), preserving frame order and traceability while draining buffered frames when capacity recovers.
- **FR-009b**: System MUST NOT rely on operating-system virtual-memory/pagefile growth as the primary inference buffering mechanism; buffering policy and limits must be application-controlled and observable.
- **FR-009c**: System MUST expose explicit buffering configuration keys and units for both tiers, including at minimum `buffer.ram.max_frames`, `buffer.ram.max_bytes`, `buffer.spill.max_frames`, `buffer.spill.max_bytes`, `buffer.spill.path`, and `buffer.overflow_policy`.
- **FR-009d**: System MUST support deterministic overflow policy values (`block`, `drop_oldest`, `drop_newest`) and MUST persist policy-driven frame-loss accounting in run/session summaries.
- **FR-009e**: System MUST preserve spill durability and replay integrity through atomic-enough spill writes, corruption detection, restart-safe replay bookkeeping, and explicit unreadable-spill failure reporting.
- **FR-009f**: System MUST define and enforce behavior for spill-path failure modes including disk-full, permission denied, spill I/O timeout, and partial write outcomes, with operator-visible status and auditable counters.
- **FR-010**: System MUST provide a repeatable benchmark process for baseline and optimized runs across supported NVIDIA and Intel accelerator profiles.
- **FR-011**: System MUST compare live latency, offline throughput, accelerator utilization, decode time, preprocessing time, inference time, postprocessing time, artifact-write time, error rate, and artifact completeness during optimization.
- **FR-012**: System MUST keep RGB processing as the default color mode unless grayscale or another reduced-color candidate meets all throughput and quality gates on representative data.
- **FR-013**: System MUST evaluate grayscale conversion separately for object detection, pose estimation, combined association, live streaming, and offline video processing using both labeled validation metrics and RGB baseline parity on representative project videos.
- **FR-014**: System MUST record the selected color mode, accelerator profile, live/offline optimization profile, model artifact versions, and benchmark evidence for each validated run.
- **FR-014a**: System MUST support separate config-selectable live and offline optimization profiles with safe defaults and benchmark-recommended overrides.
- **FR-014b**: System MUST provide exactly one dedicated Celery worker per model, independent of whether the model runs through Triton, OpenVINO, or another inference runtime.
- **FR-014c**: System MUST route model work by model identity and workload mode rather than by runtime environment alone, so changing a model's runtime does not collapse model-level isolation.
- **FR-014d**: System MUST require a hybrid offline validation profile that executes at least one enabled model on Intel GPU through OpenVINO and at least one enabled model on NVIDIA GPU through Triton in the same real-video processing run.
- **FR-014e**: System MUST provision NVIDIA Nsight Systems CLI inside a dedicated instrumented Triton validation container profile and Intel PresentMon on the Windows host as mandatory telemetry applications for hybrid completion evidence.
- **FR-014f**: System MUST persist installed telemetry tool identities and versions, capture configuration, capture artifacts, timestamps, and run correlation for every required hybrid validation run; missing or invalid required telemetry capture MUST fail the validation gate.
- **FR-014g**: System MUST use Triton's metrics endpoint as the low-overhead NVIDIA GPU and serving KPI source during benchmark runs and MUST NOT run a concurrent separate DCGM agent in the default Triton metrics topology unless a separately validated single-authority metrics mode replaces the conflicting source.
- **FR-014h**: System MUST require two full-model heavy-workload offline validation lanes using the selected models from `.env`: one lane with all selected models on NVIDIA/Triton and one lane with all selected models on Intel GPU/OpenVINO.
- **FR-015**: System MUST reject optimized profiles that improve throughput while failing either labeled validation metrics or RGB baseline parity on representative project videos, or while exceeding accepted regression thresholds for object quality, pose quality, live responsiveness, or artifact completeness.
- **FR-016**: System MUST detect missing, incompatible, unhealthy, or stale model artifacts before starting a run and during runtime health checks.
- **FR-017**: System MUST handle corrupt frames, empty frames, no-person frames, low-confidence outputs, and out-of-order frames without crashing the session or job.
- **FR-018**: System MUST isolate retry behavior per pipeline and prevent retries from creating duplicate artifacts or inconsistent final counts.
- **FR-019**: System MUST persist artifacts atomically enough that users can distinguish complete, partial, skipped, failed, and retry-generated outputs.
- **FR-019a**: System MUST fail offline processing when more than 2% of decoded frames lack required pose status or any continuous pose gap exceeds 10 seconds.
- **FR-020**: System MUST provide operator-visible degraded states and actionable error messages for accelerator failures, model failures, stream failures, upload failures, timeout, and artifact storage failures.
- **FR-021**: System MUST produce run summaries that reconcile decoded frames, submitted frames, processed frames, buffered frames, delayed frames, skipped frames, failed frames, object artifact counts, pose artifact counts, and combined artifact counts.
- **FR-022**: System MUST preserve existing object detection behavior and artifact compatibility unless a measured optimization is explicitly accepted.
- **FR-023**: System MUST support concurrent jobs or sessions without cross-run artifact contamination.
- **FR-024**: System MUST include automated validation using real model weights and real raw/live video data for all runtime scenarios.
- **FR-025**: System MUST maintain the project coverage requirement for affected modules and include tests for success, degraded, partial, and failure outcomes.
- **FR-026**: System MUST NOT mark this feature complete until a real-video hybrid offline processing run has passed decode, model inference, object-pose association, tracking, artifact persistence, overlay rendering, final video export, and summary reconciliation without unresolved errors.
- **FR-027**: System MUST measure optimized hybrid offline throughput and latency against an unoptimized hybrid baseline using the same real video, model artifacts, accelerator assignment, and artifact contract.
- **FR-027a**: System MUST compare baseline and optimized hybrid runs under equivalent mandatory telemetry instrumentation settings and record telemetry overhead characterization so reported optimization gains are not caused by unequal measurement modes.
- **FR-027b**: System MUST measure tiered-buffering overhead against a non-spill baseline under representative heavy workload and reject acceptance claims that omit queue/spill overhead attribution.
- **FR-028**: System MUST export a final annotated output video whose resolution, aspect ratio, nominal frame rate, timeline duration, and audio presence match the input video, with intentional overlays excluded from visual fidelity comparison.
- **FR-029**: System MUST retain objective export-quality evidence showing that non-overlay regions do not regress beyond the approved encoding baseline.
- **FR-030**: System MUST normalize Nsight Systems, Triton metrics, PresentMon, and OpenVINO device/profiling evidence through backend ingestion and persist run-correlated summaries and artifact references before exposing telemetry to downstream consumers.
- **FR-031**: System MUST expose authenticated REST/WebSocket telemetry interfaces for dashboard summary, timeline, artifact availability, runtime assignment, and benchmark comparison views, and dashboard functionality MUST remain independent of MCP enablement or availability.
- **FR-031a**: System MUST enforce aligned authentication and authorization semantics for telemetry REST and WebSocket interfaces at the same job/run scope, including equivalent role checks and redaction behavior.
- **FR-031b**: System MUST version telemetry REST/WebSocket schemas and include explicit schema/version traceability in run evidence so compatibility checks are deterministic.
- **FR-032**: System MUST provide a mandatory MCP telemetry server for authorized AI/operator clients that reads from normalized backend telemetry/evidence persistence rather than directly scraping raw tools or unrestricted artifact storage.
- **FR-032a**: System MUST define MCP service health expectations including health endpoint behavior, degraded status signaling, and operational recovery targets without making inference or dashboard flows depend on MCP availability.
- **FR-033**: The initial MCP telemetry interface MUST be read-only and support bounded evidence resources for summary, timeline, assignments, artifacts, fidelity, and benchmark comparison plus read-only diagnostic tools for accelerator health, run comparison, run-failure explanation, and evidence-package validation.
- **FR-033a**: System MUST quantify bounded MCP and dashboard projections with enforceable limits (for example max timeline window, max sample count, and server-side downsampling rules) and reject out-of-bounds requests deterministically.
- **FR-034**: The initial MCP telemetry interface MUST NOT provide arbitrary shell execution, arbitrary Nsight or PresentMon command arguments, unbounded raw file reads, or a `start_validation_capture` operation; any future mutating MCP operation requires separate approved authorization, auditing, and validation requirements.
- **FR-034a**: System MUST define mandatory MCP audit fields (principal, role, operation, target, bounds requested/enforced, verdict, redaction outcome, timestamp, reason code) and a minimum audit-retention policy.
- **FR-035**: System MUST enforce role-based authorization, PII-safe telemetry projection, bounded query/capture artifact access, raw trace access restrictions for authorized admin or release-validation roles, and auditable MCP access/tool invocation records.
- **FR-036**: System MUST isolate MCP failures, disablement, overload, and authorization failures from inference execution, mandatory telemetry collection, evidence persistence, and frontend dashboard operation.
- **FR-036a**: System MUST define authorized MCP role categories with measurable permission boundaries (for example operator, release-validation, admin) so access decisions are objective and testable.

### Key Entities *(include if feature involves data)*

- **Inference Run**: A live session or offline job execution with run identifier, mode, source, accelerator profile, selected color mode, model artifact versions, start/end state, and summary metrics.
- **Frame Record**: A decoded frame on the source timeline with frame identifier, timestamp, source metadata, decode status, processing status, and skip/failure reason if applicable.
- **Object Artifact**: Detection and tracking output for a frame, including detected entities, confidence, bounding regions, track identity, and artifact status.
- **Pose Artifact**: Pose output for a frame or detected person, including keypoints, confidence, skeleton data, association status, and artifact status.
- **Fallback Person Region**: A person region used for RTMPose when primary object detection is unavailable, including source, confidence, validity window, and degraded association reason.
- **Combined Association Artifact**: Relationship between object detections/tracks and pose outputs when both are valid for the same frame timeline.
- **Optimization Profile**: A benchmarked configuration choice with accelerator profile, color mode, workload mode, live/offline applicability, target objective, measured metrics, acceptance decision, and rejection reason when applicable.
- **Model Worker**: Dedicated execution capacity for one model, including queue identity, configured worker identity, runtime target, health state, backlog, and failure isolation status.
- **Hybrid Validation Evidence Package**: Proof of a real-video offline run containing model-to-runtime assignment, accelerator confirmation, stage pass/fail outcomes, baseline-versus-optimized metrics, artifact reconciliation, and exported video fidelity results.
- **Telemetry Tool Installation**: Required evidence identifying the installed NVIDIA Nsight Systems CLI package/image layer and Intel PresentMon installation, version, readiness result, and capture capability.
- **Telemetry Capture Session**: Run-correlated NVIDIA and Intel evidence containing Nsight trace output, Triton metrics samples, PresentMon output, OpenVINO profiling/device proof, timestamps, configuration, and availability verdict.
- **Normalized Telemetry Record**: Backend-ingested run-correlated representation of accelerator samples, stage metrics, source identity, artifact references, and redaction status used by dashboard and MCP projections.
- **Dashboard Telemetry Projection**: Authenticated REST/WebSocket projection of normalized summary, timeline, artifact, assignment, and comparison data for frontend panels.
- **MCP Telemetry Resource**: Bounded read-only resource projection over normalized evidence for authorized AI/operator clients, including summary, timeline, assignments, artifacts, fidelity, and comparison views.
- **MCP Access Audit Event**: Security and operations record of MCP client identity, resource or diagnostic tool invoked, authorization verdict, redaction behavior, timestamp, and denial/failure reason when applicable.
- **Exported Output Video**: Final annotated video output with source presentation properties, quality comparison result, overlay exclusions, and linkage to its inference run.
- **Run Summary**: Final reconciliation record for live or offline processing that reports counts, metrics, degraded periods, errors, retries, and artifact completeness.

## Assumptions

- The current object detection pipeline remains the baseline artifact contract, and pose artifacts must integrate with it rather than replace it.
- Supported accelerator profiles include at least one NVIDIA GPU environment and one Intel GPU environment available to the project for validation.
- Live and offline workloads may select different safe-default optimization profiles because live monitoring optimizes delayed presentation stability while offline processing optimizes total throughput and artifact completeness.
- Model-level worker isolation is required even when several models share the same runtime service, because runtime-level routing alone cannot expose per-model backlog, failure, or capacity.
- Implementation and validation work will be decomposed into independent workstreams assigned across multiple agents, with integration ownership and non-overlapping edit scopes made explicit during planning.
- "Same quality as the input video" means preserving source presentation properties and demonstrating no unapproved visual degradation outside intentional annotation overlays; it does not require overlay pixels to remain identical to the unannotated source.
- "Very optimistic optimization" means targeting large measured gains while keeping correctness gates mandatory; speed-only gains are not accepted if artifacts become misleading.
- Parallel inference may increase total device pressure, so success requires measured throughput, latency, utilization, and error-rate evidence rather than assuming parallelism is always faster.
- Grayscale can reduce raw pixel payload and may reduce decode, transfer, or preprocessing cost, but many detection and pose models are trained and validated on color inputs. If a selected model still requires color-shaped input, grayscale may provide little or no inference-speed gain and may reduce quality. The feature therefore treats grayscale as an evidence-gated candidate.
- RTMPose is treated as a top-down pose estimator for this feature: it should use valid person regions from object detection when available. Detector-free degraded pose is only acceptable through an explicitly validated fallback-region source and must not be reported as fully associated multi-person pose.
- MCP is an independently deployable AI/operator observability interface and not the application's frontend telemetry data plane; disabling MCP must not prevent dashboard telemetry, inference, or required validation evidence generation.
- The initial MCP implementation is intentionally read-only; privileged capture-start controls are deferred until a separate authorization and operational-safety decision is approved.
- Telemetry authorization infrastructure, Streamable HTTP endpoint security (including Origin validation), and MCP audit persistence are assumed available in deployment environments targeted by this feature.
- Research inputs for planning should include current vendor guidance on batching, concurrency, asynchronous execution, accelerator-specific throughput modes, and model input color assumptions.

## Current Validation Gaps

- Separate OpenVINO and Triton benchmark or probe evidence is available, but it does not prove a single real-video offline run that simultaneously executes required models on Intel GPU/OpenVINO and NVIDIA GPU/Triton.
- Existing offline output evidence includes rendered output files, but no approved source-fidelity validation report yet proves required resolution, timing, audio-presence, and non-overlay quality preservation.
- Completion evidence must prove actual per-model runtime and accelerator assignment; configuration intent or fallback-capable routing alone is insufficient.
- Any export path that can complete a job without a verified playable final video, or omit required source presentation properties, must be corrected before the completion gate can pass.
- Existing runtime endpoints and persistence offer foundations for telemetry display, but no specified normalized telemetry API/MCP projection or MCP access-audit evidence yet proves the new observability boundary.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Live monitoring with both object and pose pipelines enabled presents results through a configurable delayed-live buffer defaulting to 5 minutes, and once the buffer is established, p95 presentation jitter stays under 1.5 seconds relative to the scheduled delayed timeline.
- **SC-002**: Offline processing achieves at least 2.0x throughput improvement over the current sequential dual-pipeline baseline on each supported accelerator profile, measured on representative raw videos.
- **SC-003**: At least 99% of successfully decoded frames in validation runs have reconciled object, pose, combined-association, skipped, or failed status in the final run summary, and offline runs fail when more than 2% of decoded frames lack required pose status or any continuous pose gap exceeds 10 seconds.
- **SC-004**: Accepted optimization profiles produce zero missing mandatory artifact categories for successful runs and no duplicate frame artifacts after retries.
- **SC-005**: Accepted grayscale profiles must improve measured throughput by at least 15% over RGB on the same workload, keep labeled object and pose quality regression within 2% of the RGB baseline, and pass RGB baseline parity on representative project videos; otherwise grayscale remains disabled.
- **SC-006**: A single-pipeline failure is reflected in user-visible state within 10 seconds and does not invalidate already completed artifacts from the healthy pipeline.
- **SC-006a**: When object detection is unavailable, 100% of generated RTMPose fallback artifacts are marked degraded with fallback-region source and are excluded from fully associated object-pose counts.
- **SC-007**: Benchmark reports separate decode, preprocessing, inference, postprocessing, artifact writing, queue wait, and total runtime costs for every supported accelerator, workload mode, and live/offline optimization profile.
- **SC-007a**: Live and offline runs each start with a safe default profile, and any benchmark-recommended override records the measured improvement and the validation gates it passed.
- **SC-007b**: Benchmark and validation reports include per-model worker backlog, throughput, error rate, timeout rate, and effective worker health for every enabled model and workload mode.
- **SC-008**: 95% of validation runs across normal, degraded, and failure scenarios end with a clear completed, partial success, degraded, or failed state and an actionable final summary.
- **SC-009**: Affected modules maintain 100% line and branch coverage for new and changed behavior, including live, offline, success, partial, and failure paths.
- **SC-010**: The feature is not eligible for completion until at least one real-video offline validation run proves concurrent use of Intel GPU through OpenVINO and NVIDIA GPU through Triton and reports zero unresolved failures in all required processing and export stages.
- **SC-010a**: Completion evidence includes two full-model stress validations on the same representative source class: all `.env` selected models on NVIDIA/Triton and all `.env` selected models on Intel/OpenVINO, each with a baseline-versus-optimized comparison and no unresolved blocking-stage failures.
- **SC-011**: The accepted hybrid offline optimization profile achieves the defined offline throughput target and reduces p95 processing latency by at least 20% compared with the unoptimized hybrid baseline on the same real video.
- **SC-012**: 100% of required final-video validation outputs preserve input resolution, aspect ratio, nominal frame rate, duration within one source frame interval, and audio presence when audio exists; non-overlay visual fidelity passes the approved encoding-baseline comparison.
- **SC-013**: 100% of passing required hybrid validation runs include verified telemetry installation/version evidence and readable non-empty capture artifacts from NVIDIA Nsight Systems CLI within the Triton validation container and Intel PresentMon on Windows, plus correlated Triton and OpenVINO runtime evidence.
- **SC-014**: Baseline and optimized performance comparisons use the same mandatory telemetry mode, and each comparison reports instrumentation overhead or an equivalent-control justification before optimization gains are accepted.
- **SC-015**: 100% of telemetry dashboard integration validation cases retrieve normalized summary, timeline, artifact, runtime-assignment, and benchmark-comparison data through authenticated REST/WebSocket interfaces while MCP is disabled or unavailable.
- **SC-016**: 100% of MCP contract/security validation cases confirm read-only access to normalized permitted evidence, denial and audit of mutating or unauthorized requests, no student-identifying telemetry exposure, and no unrestricted raw artifact or profiler-command access.
- **SC-017**: In failure tests, MCP disablement, disconnection, overload, or authorization rejection causes zero interruption to inference processing, mandatory telemetry capture persistence, or frontend dashboard telemetry delivery.
- **SC-017a**: MCP health checks and status endpoints detect outage/degraded states within 30 seconds and surface recovery state within 60 seconds after service restoration.
- **SC-018**: Under defined overload tests, queueing requirements are measurable: p95 spill write latency, spill drain throughput, overflow count by policy, and queue delay are recorded; runs exceeding configured buffering SLO thresholds are flagged failed or degraded by policy.
- **SC-019**: 100% of MCP denial/allow audit events in validation tests include all required FR-034a fields and satisfy configured minimum audit-retention policy.
- **SC-020**: 100% of telemetry REST/WebSocket and MCP contract validations report explicit schema versions and pass compatibility checks against versioned contracts for the tested run.
