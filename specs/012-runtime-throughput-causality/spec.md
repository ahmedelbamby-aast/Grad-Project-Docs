# Feature Specification: Runtime Throughput Causality Governance

**Feature Branch**: `012-runtime-throughput-causality`  
**Created**: 2026-05-27  
**Status**: Draft  
**Input**: User description: "Build a Runtime Throughput, GPU Saturation, and End-to-End Pipeline Causality Governance specification as a continuity phase of the current behavioral intelligence architecture. GPU tests must be on the production server."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Prove Runtime Causality (Priority: P1)

As a runtime operator, I need every representative video workload to produce a deterministic end-to-end timing waterfall so that low throughput and near-zero accelerator activity can be attributed to a specific runtime layer instead of guessed from model load status.

**Why this priority**: The currently observed production symptom is high accelerator memory allocation with near-zero accelerator utilization while work is active. Without causality evidence, every optimization claim is speculative.

**Independent Test**: Run one representative offline workload on the production server and confirm the resulting evidence decomposes source acquisition, decode, preprocessing, transport, scheduling wait, inference, postprocessing, overlay, persistence, event emission, queue wait, retry, cache, and replay timing with a shared correlation identity.

**Acceptance Scenarios**:

1. **Given** a production offline video run with the active offline runtime profile, **When** the run completes, **Then** operators can see a frame-level or batch-level waterfall that attributes elapsed time to each runtime stage and identifies the dominant bottleneck class.
2. **Given** accelerator utilization remains below the configured target during an active run, **When** the evidence is reviewed, **Then** the system identifies whether the starvation source is decode-bound, queue-bound, orchestration-bound, transport-bound, persistence-bound, batching-bound, overlay-bound, or model-bound.
3. **Given** any stage emits a constant, placeholder, missing, or synthetic timing value, **When** the run is evaluated, **Then** the throughput evidence is rejected and the run is marked invalid rather than degraded or successful.

---

### User Story 2 - Establish Accelerator Truth (Priority: P1)

As a performance owner, I need accelerator saturation claims to be based on production-server measurements from real workloads, not on memory allocation, artifact presence, or model readiness alone.

**Why this priority**: Production acceptance requires proving useful accelerator work. Loaded models in accelerator memory do not prove inference throughput or batching effectiveness.

**Independent Test**: Execute accelerator validation only on the production server with representative live and offline workloads, then verify raw traces include utilization, occupancy where available, thermal stability, transfer overhead, per-model activity, scheduling wait, and effective throughput.

**Acceptance Scenarios**:

1. **Given** models are loaded and accelerator memory is allocated, **When** utilization remains near zero during active work, **Then** the system raises an accelerator starvation finding and blocks any production-ready throughput claim until causality evidence explains the source.
2. **Given** batched inference is enabled for a runtime profile, **When** production traces are collected, **Then** the evidence distinguishes real batching from fake batching, fragmented cadence, empty batches, undersized batches, and scheduler starvation.
3. **Given** a report claims improved accelerator throughput, **When** raw production-server traces are missing, **Then** the report is rejected as non-production evidence.

---

### User Story 3 - Govern Queue and Orchestration Economics (Priority: P2)

As a platform maintainer, I need queue depth, queue age, worker idleness, retry behavior, task fanout, and admission pressure to be visible and bounded so orchestration cannot silently starve inference or collapse under load.

**Why this priority**: Active workers do not prove efficient inference. Queue topology and orchestration fragmentation can prevent batches from forming and can hide starvation behind task churn.

**Independent Test**: Run a queue saturation and retry-stress scenario, then confirm the system exposes queue age, worker idle ratio, retry amplification, dead-letter lineage, duplicate suppression, and bounded admission behavior without uncontrolled per-frame fanout.

**Acceptance Scenarios**:

1. **Given** a workload generates more frame work than the runtime can process immediately, **When** queues grow, **Then** queue growth remains bounded, age thresholds are visible, and backpressure or shedding follows the selected runtime profile.
2. **Given** tasks retry after transient runtime failures, **When** retries accumulate, **Then** retry lineage is traceable and amplification is bounded before it can create a retry storm.
3. **Given** live and offline processing are both configured, **When** only one active runtime mode is selected, **Then** inactive endpoint profiles and inactive queue routes do not receive production inference traffic or production-ready health status.

---

### User Story 4 - Govern Decode, Transport, and Rendering Cost (Priority: P2)

As a pipeline engineer, I need decode, frame transport, serialization, resize, crop, normalization, overlay, encoding, and artifact export costs measured independently so CPU-side work and rendering cannot hide as inference underperformance.

**Why this priority**: Accelerator starvation may be caused before inference begins or after inference completes. Overlay rendering and frame serialization must be measurable and suppressible under governed profiles.

**Independent Test**: Compare representative production-server runs with overlays disabled, lightweight overlays, and forensic overlays, then confirm the evidence isolates decode FPS, preprocessing cost, frame-copy amplification, transport overhead, overlay cost, encoding cost, and end-to-end FPS.

**Acceptance Scenarios**:

1. **Given** overlay-enabled processing has lower throughput than overlay-disabled processing, **When** the benchmark is reviewed, **Then** the system reports overlay impact separately from inference impact.
2. **Given** CPU decode or preprocessing saturates before inference begins, **When** production traces are reviewed, **Then** the system attributes accelerator starvation to decode or preprocessing imbalance.
3. **Given** a future migration path is evaluated for asynchronous decode, accelerator decode, zero-copy transport, batched transport, or adaptive frame skipping, **When** the profile is assessed, **Then** the migration decision is backed by measured current bottlenecks.

---

### User Story 5 - Enforce Benchmark and Evidence Integrity (Priority: P2)

As a reviewer, I need benchmark reports, runtime dashboards, and readiness gates to reject stale, synthetic, partial, or artifact-only evidence so accepted throughput claims remain reproducible and production-valid.

**Why this priority**: This phase is a production truth specification. CI-only claims, synthetic decomposition, stale artifacts, and memory-allocation-only evidence would create false confidence.

**Independent Test**: Submit benchmark evidence with and without raw traces, production fingerprints, representative dataset attribution, repeated-run statistics, and replay references; confirm invalid evidence is rejected and valid evidence is reproducible.

**Acceptance Scenarios**:

1. **Given** a benchmark report lacks raw runtime traces, repeated-run statistics, variance accounting, confidence intervals, runtime profile lineage, queue topology lineage, or accelerator fingerprint lineage, **When** it is evaluated, **Then** it is rejected for production acceptance.
2. **Given** CI validation passes but production-server throughput validation is missing, **When** a production GPU claim is made, **Then** the claim is rejected until production evidence exists.
3. **Given** replay validation is requested under stress, **When** the replay completes, **Then** event ordering, dropped intervals, degraded intervals, and evidence lineage are deterministic or explicitly rejected.

---

### User Story 6 - Prepare Scalable Runtime Governance (Priority: P3)

As an architecture owner, I need the runtime governance model to support future graph, transformer, multimodal, distributed, multi-accelerator, accelerator decode, zero-copy, and adaptive scheduling work without imposing unnecessary heavy behavior on lightweight deployments.

**Why this priority**: Future model classes and distributed runtime modes should inherit causality, evidence, and queue contracts instead of introducing new opaque pipelines.

**Independent Test**: Review a future runtime profile proposal and confirm it declares the same causality, accelerator truth, queue, replay, evidence, degradation, and admission contracts before it can be benchmark-eligible.

**Acceptance Scenarios**:

1. **Given** a future model family or accelerator route is introduced, **When** it is registered for production evaluation, **Then** it must declare causality stages, runtime profile compatibility, evidence obligations, and replay behavior before it can be used in production claims.
2. **Given** a lightweight deployment profile is selected, **When** optional advanced runtime capabilities are unavailable, **Then** the system remains valid by reporting unavailable capabilities truthfully without silent fallback or fake evidence.

### Edge Cases

- Accelerator memory is high and models are ready, but utilization remains near zero during active production-server work.
- Workers are busy but inference batches are empty, undersized, delayed, or fragmented across routes.
- Decode or preprocessing saturates CPU before frames reach inference.
- Overlay rendering, artifact export, or websocket payload inflation collapses throughput after inference.
- Queue depth appears low while queue age is high because work is stranded in an unexpected route.
- Retry behavior creates duplicate work, recursive fallback amplification, or dead-letter growth.
- Live stream reconnects repeatedly and creates overlapping runtime tasks for the same source.
- Offline processing resumes after a worker crash and must reconcile task, queue, database, artifact, telemetry, and frontend states.
- Production and CI topology differ in runtime profile, queue ownership, dependency lock, accelerator configuration, model artifacts, or worker topology.
- Representative workloads include small videos, long videos, crowded classrooms, sparse classrooms, high motion scenes, and occlusion or re-entry scenes.
- Replayed evidence diverges from the original run because event order, frame identity, source timestamps, or degraded intervals are not deterministic.
- Telemetry values are missing, constant, stale, synthetic, or not timeline-aligned across services.
- Runtime profile changes alter overlay, batching, persistence, or telemetry behavior without recorded lineage.

### Mandatory Runtime Scenarios *(video/inference features)*

- **Live Stream Scenario**: The live RTSP runtime must run with the active live endpoint profile only, preserve live/offline isolation, record source timestamps and runtime timestamps, maintain identity and behavior lifecycle continuity, bound reconnect loops, expose latency and backpressure state, and produce independent production-server real-accelerator validation for low-latency and reconnect-storm workloads.
- **Offline Video Processing Scenario**: The offline video runtime must run with the active offline endpoint profile only, use deterministic source-time progression, support batch and chunk attribution, preserve sequence and artifact lineage, produce replay and benchmark evidence, and include production-server real-accelerator validation for small, long, crowded, sparse, high-motion, occlusion, overlay-enabled, and overlay-disabled workloads.
- **Frontend-Backend Wiring**: User-visible runtime state must distinguish `unknown`, `unavailable`, `degraded`, `valid`, `failed`, and `blocked` throughput states. Dashboards and status views must expose schema versions, evidence freshness, benchmark eligibility, degraded intervals, and production-readiness blockers without presenting synthetic success.
- **Backend-Inference Wiring**: Production inference authority must remain the active native accelerator inference endpoint. The inactive endpoint profile must reject production traffic and must not contribute to production-ready health. Timeouts, retries, fail-stop behavior, selected model identity, artifact digest, runtime profile, and endpoint mode must be recorded for every production claim.
- **Temporal/Identity/Pose Authority**: Frame identity, source timestamps, processing timestamps, pose stream provenance, tracker identity scope, behavior ontology version, temporal window validity, missingness, and invalid-window rules must be recorded so behavioral intelligence claims remain replay-safe and causality-aligned.
- **Deployment Boundary**: Development and CI may validate non-accelerator logic and schema contracts, but all accelerator utilization, saturation, thermal, batching, and production throughput acceptance tests must run on the production server with the native Linux, NVIDIA GPU, no-Docker, no-sudo deployment boundary and the single-active-mode policy.
- **Runtime Reconciliation**: After every run, task state, queue state, relational state, artifacts, telemetry, replay references, and frontend runtime state must converge to a valid terminal state or expose a blocking mismatch. Orphan tasks, duplicate runtime claims, and hidden queue ownership are production blockers.
- **Evidence Lineage**: Evidence must include immutable snapshots, artifact digests, environment fingerprint, dependency fingerprint, accelerator fingerprint, runtime topology fingerprint, dataset attribution, queue topology lineage, runtime profile lineage, telemetry provenance, replay references, and explicit distinctions between mock, synthetic, CPU-only, CI, and production-server accelerator evidence.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST treat the full video analytics runtime as a causality-governed distributed inference system with explicit stage boundaries for acquisition, decode, preprocessing, batching, transport, scheduling, inference, postprocessing, temporal aggregation, persistence, telemetry, overlay rendering, replay, and evidence export.
- **FR-002**: System MUST produce deterministic frame-level or batch-level waterfall traces for representative workloads, including decode timing, preprocess timing, serialization timing, inference scheduling wait, inference timing, postprocess timing, overlay timing, persistence timing, websocket emission timing, queue wait timing, retry timing, dead-letter timing, replay timing, cache access timing, event emission timing, runtime profile attribution, queue route attribution, worker attribution, accelerator attribution, CPU attribution, and memory pressure attribution.
- **FR-003**: System MUST classify low-throughput or low-accelerator-activity incidents as CPU-bound, queue-bound, decode-bound, orchestration-bound, transport-bound, persistence-bound, batching-bound, overlay-bound, model-bound, or mixed, based on measured evidence rather than operator judgment.
- **FR-004**: System MUST detect accelerator starvation, model starvation, queue starvation, fake batching, low occupancy where measurable, idle accelerator anomalies, decode-to-inference imbalance, orchestration-to-inference imbalance, transfer overhead, thermal instability, and scheduling imbalance.
- **FR-005**: System MUST reject production accelerator claims that rely only on memory allocation, model loading, artifact presence, or readiness probes without raw production-server runtime traces showing useful work.
- **FR-006**: System MUST measure effective accelerator throughput per runtime profile and per model, including utilization, occupancy where available, scheduler wait, transfer overhead, batch size distribution, batch fill ratio, request cadence, thermal state, and successful output cadence.
- **FR-007**: System MUST validate batching effectiveness by distinguishing real batching from fake batching, fragmented cadence, undersized batches, delayed batches, empty batches, and route starvation.
- **FR-008**: System MUST measure decode, resize, crop generation, tensor normalization, frame copy amplification, serialization, binary tensor transport, shared-memory transport eligibility, frame encoding, websocket payload size, artifact export, and overlay compositing costs as separately attributable runtime stages.
- **FR-009**: System MUST provide governed migration evidence for asynchronous decode, accelerator decode, zero-copy transport, batched frame transport, adaptive frame skipping, and profile-governed preprocessing without requiring those capabilities to be enabled in every deployment.
- **FR-010**: System MUST govern queue economics through queue depth, queue age, worker idle ratio, admission pressure, route ownership, retry lineage, dead-letter lineage, duplicate suppression, task idempotency, and queue-to-relational-state convergence.
- **FR-011**: System MUST prevent unbounded retry storms, recursive fallback amplification, hidden queue starvation, orphan runtime tasks, implicit queue ownership, and uncontrolled per-frame task fanout.
- **FR-012**: System MUST support micro-batching, temporal batching, chunk-based orchestration, adaptive queue routing, live/offline isolation, bounded admission control, backpressure, overload shedding, orchestration fairness, worker crash reconciliation, and throughput-collapse detection according to runtime profile rules.
- **FR-013**: System MUST define governed runtime profiles named `forensic_full`, `balanced_runtime`, `throughput_max`, `benchmark_truth`, `live_low_latency`, `offline_bulk_processing`, `replay_validation`, and `evidence_generation`.
- **FR-014**: Each runtime profile MUST declare overlay behavior, batching behavior, persistence frequency, telemetry verbosity, accelerator utilization targets, acceptable queue latency, replay guarantees, evidence obligations, benchmark eligibility, degradation semantics, frame skipping rules, and adaptive throttling rules.
- **FR-015**: System MUST measure overlay rendering overhead, pose skeleton rendering overhead, graph visualization overhead, text rendering overhead, compositing cost, encoding overhead, forensic replay rendering cost, and overlay impact on throughput, latency, and accelerator starvation.
- **FR-016**: System MUST support overlay-disabled runtime, lightweight runtime overlays, forensic overlays, and adaptive overlay suppression under overload, with profile lineage recorded in evidence.
- **FR-017**: System MUST require real production throughput validation for representative offline workloads, representative live RTSP workloads, crowded scenes, sparse scenes, high-motion scenes, occlusion and re-entry scenes, long-duration runs, overnight soak runs, queue saturation, reconnect storms, memory growth, cache growth, worker stability, accelerator thermal stability, dynamic batching under real load, and replay determinism under stress.
- **FR-018**: All accelerator utilization, accelerator saturation, accelerator thermal, accelerator batching, and production throughput acceptance tests MUST run on the production server. CI and development runs may support contract validation but MUST NOT satisfy production GPU acceptance.
- **FR-019**: System MUST expose operational telemetry for accelerator utilization, accelerator occupancy where available, queue depth, queue age, scheduling wait, model throughput, decode FPS, inference FPS, end-to-end FPS, dropped frames, replay drift, worker idle ratio, retry amplification, degraded intervals, batching efficiency, CPU saturation, memory pressure, websocket latency, overlay cost, and persistence latency.
- **FR-020**: Operational telemetry MUST be timeline-aligned, correlation-linked across services, replay-safe, lineage-preserving, environment-fingerprinted, runtime-topology-fingerprinted, and safe to use as immutable evidence.
- **FR-021**: System MUST define fail-closed behavior, degraded-mode governance, bounded degraded intervals, bounded queue growth, bounded memory growth, bounded retry behavior, bounded reconnect loops, deterministic replay ordering, duplicate suppression, cache eviction governance, accelerator starvation escalation, runtime reconciliation guarantees, queue-to-relational-state convergence, telemetry truth guarantees, runtime drift detection, and environment drift detection.
- **FR-022**: System MUST define CI and production synchronization gates for throughput validation, benchmark replay validation, runtime topology parity, accelerator configuration parity, model parity, queue topology parity, dependency lock parity, worker topology parity, evidence freshness, stale artifact rejection, synthetic benchmark rejection, fake causality metric rejection, and constant decomposition rejection.
- **FR-023**: System MUST prohibit giant orchestration ownership, hidden synchronous bottlenecks, direct relational writes inside inference loops, uncontrolled shared mutable state, implicit transport ownership, mixed overlay and inference logic, hidden serialization paths, duplicate telemetry emission, and unbounded event fanout.
- **FR-024**: System MUST require architectural separation between orchestration, decode, preprocessing, inference transport, inference service interaction, batching, temporal aggregation, persistence, telemetry, overlay rendering, replay, evidence export, and runtime profiling.
- **FR-025**: System MUST require raw runtime traces, immutable benchmark lineage, representative dataset attribution, accelerator fingerprint lineage, queue topology lineage, runtime profile lineage, replay references, causality-backed throughput evidence, baseline/candidate comparison, repeated-run statistics, variance accounting, confidence intervals, benchmark reproducibility, and replay determinism evidence.
- **FR-026**: System MUST reject accelerator claims without raw traces, queue claims without telemetry, synthetic decomposition metrics, placeholder runtime summaries, stale benchmark evidence, CI-only production claims, memory-allocation-only accelerator evidence, fake batching evidence, and artifact-presence-only acceptance.
- **FR-027**: System MUST preserve PostgreSQL authority for relational state, Triton authority for production inference, runtime evidence governance, queue isolation, live/offline isolation, replay determinism, immutable lineage, observability truth, fail-closed behavior, no silent fallback, no fake production evidence, and no artifact-presence-only acceptance.
- **FR-028**: System MUST prepare governance contracts for future ST-GCN, transformer, graph reasoning, multimodal, VLM-assisted, distributed inference, multi-accelerator orchestration, adaptive scheduling, zero-copy transport, accelerator preprocessing, accelerator decode, and runtime scaling capabilities without forcing heavyweight behavior on lightweight deployments.
- **FR-029**: System MUST publish explicit production blockers and acceptance rejection conditions whenever runtime truth cannot prove causality, accelerator activity, queue stability, replay determinism, benchmark authenticity, or topology parity.
- **FR-030**: System MUST maintain maturity authenticity rules that distinguish draft capability, CI-validated capability, production-server validated capability, benchmark-eligible capability, and production-accepted capability.

### Runtime Profile Contracts

- **forensic_full**: Preserves maximum evidence and overlay detail, allows higher latency, forbids frame skipping unless explicitly recorded as invalid evidence, and requires full replay lineage.
- **balanced_runtime**: Balances throughput and evidence detail, permits bounded adaptive throttling, requires stable queue age, and preserves enough telemetry for causality attribution.
- **throughput_max**: Prioritizes sustained throughput, disables or suppresses expensive overlays by default, permits governed frame skipping, requires accelerator saturation evidence, and blocks forensic claims unless a separate evidence run exists.
- **benchmark_truth**: Requires raw traces, repeated runs, variance and confidence reporting, fixed representative workload attribution, no synthetic decomposition, and production-server accelerator evidence for GPU claims.
- **live_low_latency**: Prioritizes bounded live latency, bounded reconnect behavior, adaptive shedding under overload, live/offline isolation, and clear degraded intervals.
- **offline_bulk_processing**: Prioritizes deterministic completion, chunk and batch efficiency, bounded queue growth, replayable artifacts, and production-server throughput validation.
- **replay_validation**: Prioritizes deterministic event ordering, lineage comparison, drift reporting, and rejection of non-reproducible runs.
- **evidence_generation**: Prioritizes immutable artifacts, digest-addressed outputs, environment and topology fingerprints, and benchmark-ready lineage.

### Key Entities *(include if feature involves data)*

- **Runtime Trace**: A timeline-aligned record of one run, source, frame, batch, task, or chunk with correlation identity, stage timings, runtime profile, queue route, worker identity, accelerator identity, CPU and memory pressure, degraded intervals, and evidence references.
- **Runtime Waterfall**: A deterministic decomposition of elapsed time across acquisition, decode, preprocessing, transport, scheduling, inference, postprocessing, overlay, persistence, eventing, replay, and export stages.
- **Accelerator Truth Record**: Evidence describing useful accelerator work, starvation state, utilization, occupancy where measurable, thermal state, per-model activity, transfer overhead, scheduling wait, batch effectiveness, and production-server fingerprint.
- **Queue Economics Record**: Evidence describing queue depth, queue age, admission pressure, route ownership, worker idle ratio, retry lineage, dead-letter lineage, duplicate suppression, and queue-to-relational-state convergence.
- **Runtime Profile**: A governed operating mode with declared overlay, batching, persistence, telemetry, accelerator target, queue latency, replay, evidence, degradation, frame skipping, and throttling semantics.
- **Benchmark Evidence Package**: Immutable evidence bundle containing raw traces, workload identity, dataset attribution, runtime and environment fingerprints, topology lineage, repeated-run statistics, variance, confidence intervals, baseline/candidate comparisons, and replay references.
- **Representative Workload**: A production-valid media or live-stream scenario covering small, long, crowded, sparse, high-motion, occlusion/re-entry, overlay-enabled, overlay-disabled, live low-latency, and offline bulk-processing behavior.
- **Production Readiness Gate**: A blocking acceptance rule that evaluates runtime causality, accelerator truth, queue stability, topology parity, evidence freshness, replay determinism, and benchmark authenticity.

### Assumptions

- The existing behavioral intelligence architecture already contains backend orchestration, queueing, cache, relational authority, accelerator inference authority, pose, tracking, semantic and temporal reasoning, observability, evidence governance, CI/CD validation, live runtime, offline runtime, replay, and lineage systems.
- The production server is the only valid authority for accelerator utilization, accelerator saturation, thermal stability, batching effectiveness, and production throughput claims.
- Development and CI environments may validate schemas, decomposition contracts, replay logic, evidence packaging, and non-accelerator behavior, but cannot satisfy production GPU acceptance.
- Representative workload definitions may evolve, but accepted evidence must always include workload attribution and must distinguish production, CI, synthetic, mock, CPU-only, and degraded runs.
- Runtime truth is favored over permissive availability: unknown or unmeasured causality produces a blocked or invalid state, not a successful production claim.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: For every accepted representative production run, at least 95% of processed frames or batches have complete causality decomposition across required runtime stages, and the remaining missing intervals are explicitly marked degraded or invalid.
- **SC-002**: Within one representative production-server run, operators can identify the dominant bottleneck class for low throughput or low accelerator activity with evidence-backed attribution and without relying on model readiness or memory allocation.
- **SC-003**: All accepted GPU throughput, saturation, batching, and thermal claims are backed by raw production-server traces, accelerator fingerprint lineage, runtime profile lineage, and representative workload attribution.
- **SC-004**: Queue growth remains bounded during representative stress runs, and queue age, worker idle ratio, retry amplification, dead-letter lineage, duplicate suppression, and convergence state are visible for every active route.
- **SC-005**: Live stream scenarios meet the selected profile's latency, reconnect, backpressure, isolation, and evidence targets through the active production route, or the run is reported as degraded, failed, or blocked with causal evidence.
- **SC-006**: Offline video scenarios complete with deterministic source-time progression, replayable artifacts, bounded queue growth, and production-server accelerator evidence for small, long, crowded, sparse, high-motion, occlusion, overlay-enabled, and overlay-disabled workloads.
- **SC-007**: Overlay-disabled, lightweight-overlay, and forensic-overlay runs report measurable overlay impact on throughput, latency, and accelerator starvation, with overlay cost separated from inference cost.
- **SC-008**: Dynamic batching validation distinguishes real batching from fake batching and reports batch fill ratio, request cadence, scheduling wait, and per-model effective throughput under real production load.
- **SC-009**: Long-duration and overnight production-server runs expose memory growth, cache growth, worker stability, degraded intervals, accelerator thermal stability, and replay determinism without silent fallback.
- **SC-010**: Benchmark reports include baseline/candidate comparisons, repeated-run statistics, variance accounting, confidence intervals, raw traces, immutable lineage, and replay references; reports missing these elements are rejected.
- **SC-011**: CI and production synchronization gates detect runtime topology drift, accelerator configuration drift, model artifact drift, queue topology drift, dependency drift, worker topology drift, stale artifacts, synthetic metrics, constant decomposition, and fake causality before production acceptance.
- **SC-012**: Runtime reconciliation proves task, queue, relational, artifact, telemetry, replay, and frontend states converge after success, failure, crash, retry, or reconnect scenarios, or exposes a production blocker.
- **SC-013**: No production readiness report is accepted when evidence is CI-only, synthetic-only, CPU-only for GPU claims, artifact-presence-only, memory-allocation-only, stale, missing raw traces, missing queue telemetry, or missing replay determinism evidence.
- **SC-014**: Future runtime capabilities can be evaluated by declaring the same causality, accelerator truth, queue, replay, evidence, degradation, and admission contracts without requiring heavyweight optional capabilities in lightweight profiles.
- **SC-015**: Production acceptance explicitly preserves PostgreSQL authority, Triton authority, runtime evidence governance, queue isolation, live/offline isolation, replay determinism, immutable lineage, observability truth, fail-closed behavior, no silent fallback, no fake production evidence, and no artifact-presence-only acceptance.

### Production Blockers and Acceptance Rejection Conditions

- Accelerator claims are blocked unless GPU-related tests and evidence collection run on the production server.
- Production throughput claims are blocked when causality decomposition is missing, constant, synthetic, stale, or not timeline-aligned.
- Production readiness is rejected when active and inactive endpoint profiles both receive production inference traffic or both report production-ready status.
- Benchmark acceptance is rejected when representative workload attribution, raw traces, repeated-run statistics, variance, confidence intervals, or replay references are missing.
- Queue stability acceptance is rejected when retry storms, recursive fallback amplification, hidden starvation, orphan tasks, implicit queue ownership, or uncontrolled per-frame fanout are observed.
- Runtime maturity is rejected when evidence cannot distinguish draft, CI-validated, production-server validated, benchmark-eligible, and production-accepted capability states.
