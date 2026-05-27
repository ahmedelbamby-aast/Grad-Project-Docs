# Implementation Plan: Behavioral Semantic Intelligence Layer

**Branch**: `011-bsil-semantic-runtime` | **Date**: 2026-05-27 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `specs/011-bsil-semantic-runtime/spec.md`

## Summary

BSIL extends the existing production maturity runtime from pose/tracking
analytics into semantic behavioral intelligence. It introduces semantic pose
states, temporal behavioral windows, explainable episodes, interaction context,
adaptive anomaly accumulation, decision lineage and scientific validation while
preserving Triton authority, queue isolation, PostgreSQL-only persistence,
runtime reconciliation, benchmark causality, observability and CI/CD gates.

## Technical Context

**Language/Version**: Existing backend/frontend/runtime stack  
**Primary Dependencies**: Existing video, tracking, pose, queue, telemetry and evidence services  
**Storage**: PostgreSQL only for durable relational authority  
**Testing**: Existing backend, frontend, system, contract, evidence and CI gates  
**Target Platform**: Native Linux production with validated GPU runtime where applicable  
**Project Type**: Behavioral intelligence platform extension  
**Performance Goals**: Bounded live latency, bounded queue growth, deterministic offline replay and attributable GPU/resource usage  
**Constraints**: No silent fallback, no placeholder behavior, no accusation semantics, no production claims without runtime evidence  
**Scale/Scope**: Live RTSP and offline classroom processing with future graph/transformer expansion  
**Runtime Scenarios**: Live RTSP semantic reasoning and offline behavioral processing are both mandatory  
**Inference/Tracking Reference**: Existing detector, tracking, RTMPose and future governed behavior model contracts  
**Runtime Authority**: Production Triton-only route for governed model inference; heuristic paths explicitly labeled and non-scientific until validated  
**Temporal/Identity Authority**: Source-time windows, canonical identity continuity, lifecycle/ReID confidence and invalid-window suppression  
**Evidence/Schema Authority**: Decision traces, episode lineage, temporal reasoning audit, interaction graph audit and immutable evidence packages  
**Deployment Topology**: Existing production native Linux topology; no Docker/sudo production assumption  
**Runtime Reconciliation**: Task, queue, database, artifact, telemetry and frontend convergence checks for every behavioral record  
**Lineage/Fingerprints**: Evidence, model, runtime, deployment, benchmark, dataset, telemetry and artifact digest lineage  
**Budgets/SLOs**: Feature-specific latency, queue, retry, timeout, degradation, drift and rollback thresholds
**Runtime Anti-Fragility**: Retry ceilings, queue collapse containment, dead-letter routing, stale-state eviction, worker crash recovery and fail-closed lineage policy  
**GPU Truth**: GPU busy percent, queue saturation, effective batch size, batch collapse, model starvation, decode/inference/orchestration ratios and raw event traces  
**Scientific Governance**: Temporal replay determinism, uncertainty propagation, graph contamination prevention, baseline drift quarantine and benchmark causality  
**Frontend Governance**: Truth-state, uncertainty, degraded-state, stale-runtime, replay-mismatch and interaction-ambiguity rendering  

## Constitution Check

> **Production Runtime Authority Gate**: PASS REQUIRED. BSIL must not bypass
> single-active-mode runtime policy or production inference authority.

> **Temporal and Identity Truth Gate**: PASS REQUIRED. Semantic, temporal,
> episode and interaction records are invalid without timestamp and identity
> continuity evidence.

> **Pose and Behavior Semantics Gate**: PASS REQUIRED. Semantic outputs must
> expose confidence, uncertainty, lineage and non-accusatory semantics.

> **Queue and Failure Gate**: PASS REQUIRED. New semantic and behavioral queues
> must remain isolated by mode, bounded by retry/backpressure policy and
> observable under RTSP reconnect and queue pressure.

> **Contract and Storage Gate**: PASS REQUIRED. All new records, artifacts and
> event payloads require versioned schemas, explicit fields, PostgreSQL
> persistence and immutable lineage.

> **Observability and Scientific Evidence Gate**: PASS REQUIRED. Runtime,
> scientific validity and operational stability evidence must be separate and
> reproducible.

> **Live/Offline Validation Gate**: PASS REQUIRED. At least two live RTSP and
> three offline representative datasets are required for maturity closure.

> **Anti-Regression Runtime Truth Gate**: PASS REQUIRED. Acceptance scripts
> must inspect runtime state and artifact content, not booleans alone.

> **Evidence Integrity and Lineage Gate**: PASS REQUIRED. Placeholder,
> temporary-path-only, dev-only-for-production and SQLite-backed evidence is
> rejected.

> **XFail, Drift and Debt Gate**: PASS REQUIRED. Hidden xfails, unbounded debt
> and runtime drift block closure.

> **Runtime Anti-Fragility Requirements**: PASS REQUIRED. BSIL must define and
> enforce retry ceilings, queue depth thresholds, degraded-mode duration,
> stale-window eviction, orphan reconciliation, dead-letter routing, RTSP
> reconnect storm containment, worker crash recovery, duplicate suppression
> and fail-closed behavior when lineage is missing.

> **GPU Utilization Truth Gate**: PASS REQUIRED. GPU-backed claims require raw
> runtime traces for GPU busy percent, queue saturation, per-model occupancy,
> effective batch size, batch collapse ratio, model starvation ratio, tensor
> transfer overhead, decode-to-inference ratio and orchestration overhead
> ratio. Heavy-workload claims with 0% GPU utilization are rejected.

> **Temporal Integrity Science Gate**: PASS REQUIRED. Temporal confidence,
> uncertainty propagation, causal interruption semantics, semantic decay,
> frame ordering, cross-camera drift, contamination prevention and replay
> determinism must be validated before behavioral maturity.

> **Graph and Baseline Governance Gate**: PASS REQUIRED. Interaction edges and
> adaptive baselines require lifecycle, rollback, contamination prevention,
> reproducibility, drift quarantine and identity-confidence gates.

> **CI/CD Runtime Synchronization Gate**: PASS REQUIRED. CI green is not
> production-ready. Production acceptance requires parity checks for runtime
> topology, model repository, Triton config, queue topology, environment
> fingerprint, dependency lock, worker routing, runtime profile and migrations.

> **Maturity Authenticity Gate**: PASS REQUIRED. Presence-only acceptance,
> placeholder evidence, mocked production claims, disabled assertions, hidden
> xfails, synthetic runtime truth and benchmark summaries without raw runtime
> backing are rejected.

## Project Structure

```text
specs/011-bsil-semantic-runtime/
├── spec.md
├── plan.md
├── tasks.md
├── evidence_contract.md
├── runtime_governance.md
├── behavioral_lineage_contract.md
├── dataset_governance.md
├── acceptance_gates.md
├── observability_contract.md
└── checklists/
    └── requirements.md
```

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| Multi-phase semantic layer | Behavior requires pose semantics, temporal accumulation, episodes, interaction context and evidence governance | A single heuristic score would recreate false-confidence maturity gaps |
| Both live and offline validation | Feature spans both runtime modes | One-mode validation would not prove runtime authority or queue behavior |
| Explicit scientific governance | Behavioral claims can affect review decisions | Runtime performance evidence alone cannot prove behavioral validity |

## Phase Strategy

1. Define semantic pose states and confidence/uncertainty contract.
2. Define temporal reasoning windows, state transitions and decay rules.
3. Define episode lifecycle and immutable update semantics.
4. Define multi-person interaction graph and identity-confidence gates.
5. Define adaptive anomaly accumulation and baseline drift visibility.
6. Define decision lineage, replay/debug artifacts and reconstruction rules.
7. Define scientific dataset governance, benchmark evidence and CI gates.

## Architectural Expansion Section

### Current Architecture State

The current platform is a production-oriented video analytics runtime. It can
ingest live and offline video, run detector and RTMPose inference through the
governed runtime, track identities, materialize temporal pose sequences,
orchestrate work through isolated queues, persist governed state in
PostgreSQL, emit runtime telemetry, produce forensic/runtime causality
artifacts and enforce maturity evidence gates.

That foundation is necessary but not sufficient for behavioral intelligence.
It answers where people and joints were detected and how those observations
flowed through runtime. It does not yet provide stable semantic interpretation
of pose, online behavioral state, episode-level reasoning, interaction context
or scientifically validated anomaly accumulation.

### Remaining Intelligence Gaps

The next phase closes these gaps:

- pose-only outputs do not explain attention, engagement, posture or motion
  semantics;
- heuristic anomaly rules cannot distinguish transient noise from sustained
  behavioral evidence;
- frame intelligence cannot preserve duration, cooldown, decay, causal
  interruptions or state recovery;
- temporal storage without online state reasoning cannot support live
  behavioral awareness;
- single-person reasoning cannot explain peer context or synchronized
  interactions;
- future ST-GCN, transformer and graph layers need governed sequence,
  ontology, feature, lineage and evaluation contracts before model work starts.

The architectural transition is:

1. **Frame intelligence**: detections, keypoints and tracks for individual
   frames.
2. **Temporal intelligence**: identity-scoped ordered observations with
   source-time truth and replay determinism.
3. **Behavioral semantics**: confidence-scored, explainable interpretations
   over temporal windows.
4. **Contextual reasoning**: interaction graphs, adaptive baselines and
   episode-level evidence that remain auditable and non-accusatory.

### Why Online State Matters

Offline analysis can recompute from full history. Live behavioral
intelligence must maintain rolling per-person state without losing source-time
authority, identity continuity or replayability. The online state engine must
support real-time decisions while guaranteeing that every state can be rebuilt
from durable lineage after crash, reconnect, queue retry or deployment
cutover.

### Why Future Graph/ST-GCN Readiness Matters

Graph and neural temporal models are not part of the first BSIL acceptance
path, but their requirements must shape contracts now. Sequence schemas,
ontology versions, embeddings, interaction edges, feature lineage, dataset
provenance and evaluation protocols must be stable before learned models are
introduced. Future readiness cannot increase runtime complexity for the
lightweight default path.

## New Engine Layers

### A. Pose Decision Engine

**Purpose**: Convert low-level keypoints into interpretable motion and posture
semantics without claiming deterministic behavioral truth.

**Responsibilities**:

- pose confidence normalization;
- temporal smoothing with visible raw/smoothed lineage;
- joint reliability scoring;
- missing-keypoint interpolation policy with invalid-gap limits;
- motion vector extraction;
- posture state derivation for standing, sitting, leaning, head-down and
  related observable states;
- body orientation inference;
- motion energy estimation;
- action transition tracking.

**Runtime contracts**:

- latency budget is feature-profile specific and must be measured separately
  for semantic computation and queue wait;
- confidence propagates from joint reliability, pose missingness, identity
  continuity and timestamp validity;
- outputs use deterministic schemas and typed event envelopes;
- fallback policy is fail-closed for production-valid outputs when lineage is
  missing;
- CPU heuristic execution is allowed only when labeled heuristic and excluded
  from GPU/model scientific claims.

### B. Temporal Behavior Engine

**Purpose**: Understand behavior across source-time windows.

**Responsibilities**:

- sliding windows and temporal aggregation;
- sequence buffering with bounded memory;
- behavior state transitions;
- temporal decay logic;
- state confidence accumulation;
- event-duration tracking;
- short-term and long-term behavior memory;
- multi-resolution timelines.

**Runtime contracts**:

- online memory limits must be profile-specific and enforced with stale-window
  eviction;
- live and offline queues remain isolated;
- durable state persistence is authoritative and cache state is rebuildable;
- replay determinism is required for every accepted behavioral transition;
- timestamp truth authority follows source time first, then ingest, queue,
  inference and persistence timestamps for diagnostics only.

### C. Behavioral Semantics Layer

**Purpose**: Transform temporal features into meaningful behavioral
interpretation while preventing false confidence.

**Responsibilities**:

- attention loss;
- distraction patterns;
- prolonged absence;
- suspicious posture candidates without accusation semantics;
- repeated gaze deviation;
- social interaction inference;
- unusual movement bursts;
- inactivity anomalies;
- behavioral context windows.

**Runtime contracts**:

- semantic confidence scoring decomposes pose, time, identity, interaction,
  context and runtime validity;
- explainability payloads are required for every accepted semantic output;
- evidence attribution links to source windows, state transitions and runtime
  path;
- false-positive mitigation includes spike suppression, uncertainty
  propagation and reviewer-visible ambiguity;
- contextual weighting must be versioned and environment-sensitive without
  hidden self-modifying behavior.

### D. Online State Engine

**Purpose**: Maintain real-time per-person behavioral state with crash-safe
rebuild and replay consistency.

**Responsibilities**:

- per-track rolling state;
- temporal lifecycle;
- confidence evolution;
- state expiration;
- re-entry and occlusion continuity;
- runtime reconciliation;
- event lineage.

**Runtime contracts**:

- Redis may cache live rolling state, but PostgreSQL remains durable
  authority;
- TTL policies must exist for every cache key family;
- synchronization guarantees must reconcile Redis, PostgreSQL, task state,
  artifacts and UI truth-state;
- crash recovery rebuilds from PostgreSQL lineage and replayable events;
- idempotent state rebuild prevents duplicate episodes or transitions;
- replay consistency is a release gate.

### E. Future Intelligence Readiness

The plan must preserve future compatibility with ST-GCN, transformers, graph
reasoning, contrastive learning, multimodal reasoning, VLM integration, audio
fusion and multi-camera identity graphs.

Required foundations:

- sequence contracts with stable time, identity, pose, semantic and graph
  references;
- embedding persistence strategy with model/version lineage;
- ontology versioning for semantic and behavioral labels;
- feature lineage and dataset provenance;
- evaluation protocols separating runtime performance, scientific validity
  and operational stability;
- model and data contracts that can support future learned systems without
  changing accepted historical evidence semantics.

## Performance + GPU Strategy

GPU use is required when a governed model inference path is part of the
production claim. CPU is preferred for lightweight deterministic bookkeeping,
state-machine transitions, evidence validation, schema checks, replay
orchestration and explicitly labeled heuristic rules that do not claim GPU
model maturity.

### Runtime Policy

- Online live mode is latency-first: admission control, bounded queues,
  smaller batches and fast degradation are preferred over throughput.
- Offline mode is throughput-first: larger batches, replay determinism and
  complete artifact generation are preferred over immediate response.
- Hybrid runtime is allowed only when each stage declares CPU/GPU authority,
  model/heuristic source and telemetry attribution.
- The production Triton binary remains the governed pinned path:
  `/home/bamby/services/triton_build_r2502/tritonserver/install/bin/tritonserver`.
- Dynamic batching must be profile-specific and must report effective batch
  size, queue wait, batch collapse and model starvation.
- Zero-copy or reduced-copy strategies may be introduced only after raw
  decomposition shows transfer overhead is material.

### Utilization Philosophy

Healthy GPU utilization is not a constant high percentage. A 0% snapshot can
occur during idle periods, source starvation, CPU-bound decode,
orchestration-bound queues, warmup gaps or short heuristic-only windows.
However, 0% GPU utilization during a claimed heavy GPU-backed workload is a
blocking contradiction unless raw traces prove the workload did not reach the
GPU by design.

The plan optimizes the full pipeline, not raw GPU percentage alone. It must
measure frame decode, queue wait, orchestration overhead, serialization,
deserialization, tensor transfer, model execution, persistence and artifact
writing. Orchestration bottlenecks can dominate end-to-end latency even when
model execution is fast; benchmark reports must therefore include pipeline
decomposition rather than only GPU metrics.

### Collapse Prevention

- Queue collapse prevention must shed or defer work before queue age exceeds
  the configured timeout budget.
- Retry amplification prevention must enforce one replacement work item per
  retry and dead-letter after the retry ceiling.
- Orchestration overhead minimization must be driven by measured queue,
  serialization, routing and persistence costs.

## Production Governance Additions

BSIL production governance forbids:

- silent fallback;
- fake inference outputs;
- placeholder detections or placeholder semantic outputs;
- test-mode promotion into production evidence;
- dev-network evidence treated as production evidence;
- artifact-presence-only acceptance;
- SQLite in production acceptance or evidence paths;
- hidden runtime downgrade;
- malformed evidence artifacts;
- benchmark summaries without raw runtime causality traces.

Required governance artifacts:

- evidence authenticity checks with freshness, digest and raw trace validation;
- causal provenance linking input, queue, inference, state, persistence,
  telemetry and artifact output;
- runtime truth contracts for process, endpoint, model, queue, GPU, worker and
  storage state;
- repeated-run benchmark reports with baseline/candidate comparability,
  variance, confidence intervals and failed-run accounting;
- representative dataset manifests for live, offline, crowded,
  occlusion/re-entry and long-duration scenarios.

## Hardening Strategy Inside the Plan

The next phase is feature-first intelligence evolution with built-in
anti-regression gates. Hardening must protect production truth without
blocking useful semantic capability behind research paralysis.

Required strategy:

- implement semantic and temporal capability in small vertical slices;
- run periodic stabilization waves after each major engine layer;
- track debt budget with owner, expiry, risk and acceptance impact;
- escalate governance controls as outputs move from development to staging to
  production;
- use maturity checkpoints for runtime truth, evidence authenticity,
  scientific validity and operational stability;
- regenerate rolling evidence whenever runtime, model, queue, schema,
  dependency or dataset inputs change;
- perform progressive production validation instead of one final all-or-none
  push.

The plan avoids premature over-hardening by allowing labeled heuristic
development evidence, but it does not allow heuristic evidence to close
scientific or production maturity gates.

## Runtime Anti-Fragility Requirements

BSIL execution must remain bounded under overload and partial failure. The
plan must implement the following measurable contracts before production
readiness:

| Control | Planning requirement |
| --- | --- |
| Retry ceiling | Configure maximum 3 attempts per work item and dead-letter routing after exhaustion |
| Queue depth | Define per-profile queue depth and age thresholds; sustained breach for 60 seconds marks degraded runtime |
| Degraded duration | Degraded semantic/behavioral mode expires after 5 minutes without operator-reviewed continuation |
| Stale windows | Evict or mark stale temporal windows older than 2x configured window length |
| Replay drift | Replay must match source ordering within 1 frame or 50 ms |
| Identity fragmentation | Suppress accepted behavior when representative fragmentation exceeds 5% for the window |
| Interaction ambiguity | Block strong interaction claims when ambiguity exceeds 10% of contributing evidence |

Implementation must include no retry amplification storms, bounded fallback
recursion, queue collapse containment, partial inference isolation, orphan task
reconciliation, stale-state eviction, RTSP reconnect storm containment, worker
crash recovery, idempotent state updates, duplicate event suppression, memory
pressure handling, GPU starvation detection, CPU fallback rejection, degraded
telemetry labeling and forced fail-closed behavior when lineage is missing.

## GPU Utilization Truth Contracts

GPU-backed performance claims require raw runtime traces and attribution:

- GPU occupancy and busy percentage by run window;
- inference-vs-orchestration decomposition;
- Triton queue wait and queue saturation;
- serialization/deserialization time;
- frame decode time and decode-to-inference ratio;
- model batching efficiency, dynamic batching effectiveness, effective batch
  size and batch collapse ratio;
- tensor transfer overhead;
- CPU bottleneck attribution and orchestration overhead ratio;
- per-model occupancy, model starvation ratio, average kernel occupancy where
  available and idle GPU anomaly detection.

Acceptance rejects 0% GPU utilization under claimed heavy workloads, fake
batching metrics, constant causality metrics, synthetic decomposition values
and reports missing raw runtime event traces.

## Temporal Integrity Science Contracts

Temporal implementation must define:

- timestamp authority hierarchy and frame ordering validation;
- temporal confidence propagation and uncertainty accumulation;
- causal interruption semantics for dropped frames, disconnects, occlusions
  and re-entry;
- semantic decay curves and state transition explainability;
- temporal contamination detection and invalid-window contamination
  prevention;
- uncertainty amplification suppression and interaction contamination
  isolation;
- temporal replay determinism, identity continuity confidence scoring,
  cross-camera drift detection and re-entry contamination suppression.

## Multi-Person Graph Governance

The graph runtime must define edge lifecycle, decay semantics, confidence
decomposition, suppression lineage, contamination protection, rollback safety,
replay determinism, conflict resolution, temporal synchronization, identity
merge/split lineage and cross-window consistency validation.

Anti-false-association gates block strong interaction claims under unstable
identity continuity, graph propagation across invalid windows and interaction
inheritance after identity fragmentation.

## Adaptive Baseline Governance

Adaptive anomaly implementation must define bounded adaptation windows,
adaptive hysteresis, baseline snapshot versioning, reproducibility, rollback,
drift quarantine, contamination prevention, session-shift detection,
classroom-context reset policy and anomaly inflation protection. Every
adaptive update, baseline change and threshold shift must be lineage-backed,
explainable and reconstructable.

## Operational Failure Taxonomy

Every implementation phase must map failures to telemetry, reconciliation,
acceptance behavior and severity (`info`, `degraded`, `blocking`,
`critical`). Required categories:

| Category | Acceptance behavior |
| --- | --- |
| `runtime_failure` | block production readiness |
| `degraded_runtime` | label degraded and enforce duration budget |
| `queue_pressure` | throttle/shed/dead-letter before collapse |
| `inference_timeout` | fail affected scope without silent retry |
| `lineage_missing` | fail closed |
| `identity_uncertain` | suppress strong claims |
| `timestamp_invalid` | invalidate window |
| `replay_invalid` | block maturity closure |
| `fallback_activation` | reject production-valid output |
| `GPU_underutilized` | block heavy-workload performance claim |
| `benchmark_invalid` | rerun with raw traces |
| `evidence_invalid` | reject and supersede artifact |
| `telemetry_invalid` | block readiness claim |
| `reconciliation_failure` | quarantine workflow |
| `graph_invalid` | suppress graph output |
| `baseline_drift_exceeded` | quarantine baseline |
| `stale_artifact_detected` | require fresh evidence |
| `CI_runtime_mismatch` | require production reconciliation |

## Evidence Authenticity Enforcement

Planning must require evidence freshness windows, immutable artifact digests,
raw runtime trace retention, benchmark provenance validation, dataset
provenance hashes, runtime topology fingerprinting, worker topology
fingerprinting, Triton model fingerprinting, GPU fingerprint lineage, CI
artifact signature validation and production-vs-dev plus synthetic-vs-real
separation.

Reused artifacts, stale benchmark evidence, generated fake runtime summaries,
evidence without raw traces and benchmark summaries without replay references
are acceptance failures.

## CI/CD Runtime Synchronization

CI must verify eligibility, while production validation proves readiness.
Required parity checks include CI-to-production runtime topology, model
repository, Triton configuration, queue topology, environment fingerprint,
dependency lock, worker routing, runtime profile and migration consistency.

Production drift detection must run after deployment and before maturity
claims. CI green does not override production runtime evidence gaps.

## Implementation Decomposition Rules

Boundaries are mandatory:

| Boundary | Responsibility |
| --- | --- |
| Orchestration | admission, routing, lifecycle and reconciliation |
| Semantic reasoning | semantic pose state and uncertainty only |
| Temporal state | windows, transitions, decay, hysteresis and spike suppression |
| Episode lifecycle | creation, closure, supersession and immutable history |
| Graph reasoning | interaction edge lifecycle and graph consistency |
| Anomaly accumulation | baselines, thresholds, drift and adaptive hysteresis |
| Lineage persistence | immutable decision traces and replay references |
| Telemetry | measured event emission and truth-state reporting |
| Replay engine | deterministic reconstruction and drift detection |
| State cache | bounded runtime acceleration, never durable authority |

Forbidden implementation patterns: giant orchestration files, hidden
cross-layer coupling, direct database writes from inference paths, silent
mutable state and implicit queue ownership. Interfaces must be deterministic,
versioned, replay-safe and carried by typed event envelopes.

## Long-Running Stability Requirements

Production readiness requires endurance validation:

- minimum 8-hour soak for selected production profile before maturity closure;
- at least 30 minutes per live RTSP representative dataset;
- complete-source processing for offline datasets;
- memory stability, queue leak and Redis growth evidence;
- temporal cache pruning and worker recycling policy;
- GPU thermal/memory stability evidence for GPU-backed paths;
- reconnect endurance and degraded-mode endurance tests;
- repeated-run variance thresholds and confidence intervals.

## Maturity Authenticity Rules

Acceptance must be causality-backed, replay-backed, runtime-backed,
topology-backed and benchmark-reproducible. The plan rejects presence-only
acceptance, placeholder evidence, mocked production claims, dev-only runtime
evidence labeled production, hidden xfails, disabled assertions, skipped
runtime gates without explicit governance, synthetic runtime truth promoted as
measured telemetry, unsupported scientific claims and benchmark summaries
without raw runtime backing.

## Frontend Governance

Frontend work must render truth-state, degraded-state, uncertainty,
interaction ambiguity, stale-runtime indicators, replay mismatch indicators,
telemetry-truth indicators and confidence decomposition. Accusatory UI
language is forbidden. Frontend state must reconcile against backend,
persistence and runtime truth before showing valid behavioral output.

## Future Scalability Strategy

The default runtime profile remains lightweight. Heavy inference paths are
optional, profile-aware and selectively activated only after measured need.
Planning must include adaptive batching strategy, distributed execution
contracts, graph scalability boundaries, temporal storage retention, archival
strategy and replay acceleration without forcing ST-GCN, transformer, graph
neural network, multimodal or VLM complexity onto lightweight deployments.

## Implementation Waves

### Wave 1 - Pose Decision Intelligence

**Objectives**: Normalize pose confidence, derive posture/motion semantics and
emit deterministic semantic pose states.  
**Architectural scope**: Pose Decision Engine plus semantic output contracts.  
**Files/modules impacted**: `backend/apps/behavior/*`,
`backend/apps/video_analysis/*`, `backend/apps/pipeline/*`,
`frontend/src/*`, `scripts/ci/*`, `ci_evidence/*`.  
**Runtime impact**: Adds semantic processing stage after validated pose
output; no new production fallback authority.  
**DB impact**: Adds semantic pose state and confidence decomposition records.  
**API impact**: Adds read-only semantic state and uncertainty fields.  
**WebSocket impact**: Adds truth-state semantic updates with confidence and
uncertainty.  
**CI impact**: Adds schema, lineage, no-placeholder and replay unit/contract
checks.  
**Production impact**: Development-first, then staging; production acceptance
requires real live/offline evidence.  
**Evidence requirements**: semantic trace exports, pose lineage audit and
low-confidence suppression cases.  
**Benchmark requirements**: semantic latency, queue wait and orchestration
overhead decomposition.  
**Rollback strategy**: Disable semantic output admission and continue pose
analytics without marking semantic behavior valid.  
**Acceptance criteria**: 95% accepted semantic outputs include lineage,
confidence, uncertainty and runtime attribution.  
**Explicit non-goals**: no anomaly maturity, no graph reasoning, no learned
behavior model claim.

### Wave 2 - Temporal Understanding Engine

**Objectives**: Add sliding windows, temporal aggregation, state transitions,
decay, cooldown and replay determinism.  
**Architectural scope**: Temporal Behavior Engine and Online State Engine
foundation.  
**Files/modules impacted**: `backend/apps/behavior/*`,
`backend/apps/tracking/*`, `backend/apps/pipeline/*`,
`backend/apps/runtime/*`, `frontend/src/*`, `tools/prod/*`.  
**Runtime impact**: Adds bounded state memory and temporal queues per runtime
mode.  
**DB impact**: Adds temporal reasoning windows and state transition history.  
**API impact**: Adds state timeline and invalid-window visibility.  
**WebSocket impact**: Adds live state transitions with truth-state and replay
metadata.  
**CI impact**: Adds deterministic replay, timestamp authority and spike
suppression tests.  
**Production impact**: Requires RTSP reconnect and crash recovery validation.  
**Evidence requirements**: temporal replay audit, spike-only rejection and
state-decay examples.  
**Benchmark requirements**: aggregation latency, memory use and queue age.  
**Rollback strategy**: Stop temporal state admission and mark active windows
invalidated/superseded.  
**Acceptance criteria**: no frame-level spike creates sustained state; replay
matches within 1 frame or 50 ms.  
**Explicit non-goals**: no multi-person graph, no adaptive baseline.

### Wave 3 - Behavioral Semantics and Episode Lifecycle

**Objectives**: Convert temporal states into explainable episodes and
non-accusatory behavioral summaries.  
**Architectural scope**: Behavioral Semantics Layer and Episode Lifecycle
Engine.  
**Files/modules impacted**: `backend/apps/behavior/*`,
`backend/apps/anomalies/*`, `backend/apps/forensics/*`,
`frontend/src/*`, `ci_evidence/*`.  
**Runtime impact**: Adds episode generation, suppression and supersession
events.  
**DB impact**: Adds behavioral episode, causal chain and immutable update
history.  
**API impact**: Adds episode read models with confidence decomposition.  
**WebSocket impact**: Adds episode lifecycle updates.  
**CI impact**: Adds no-accusation-language, causal-chain and mutation-safety
tests.  
**Production impact**: Requires reviewer-visible uncertainty and invalidation.  
**Evidence requirements**: episode lineage report and suppression examples.  
**Benchmark requirements**: episode creation/update latency and artifact write
latency.  
**Rollback strategy**: Supersede generated episodes and disable episode
promotion while preserving raw semantic/temporal evidence.  
**Acceptance criteria**: zero accepted episodes from missing or invalid
evidence.  
**Explicit non-goals**: no scientific anomaly effectiveness claim.

### Wave 4 - Multi-Person Interaction Graph

**Objectives**: Add identity-gated interaction edges and temporal graph
context.  
**Architectural scope**: Interaction Graph Runtime.  
**Files/modules impacted**: `backend/apps/behavior/*`,
`backend/apps/tracking/*`, `backend/apps/cameras/*`,
`backend/apps/forensics/*`, `frontend/src/*`.  
**Runtime impact**: Adds graph cache and graph reasoning queues per runtime
mode.  
**DB impact**: Adds interaction edge lifecycle and graph audit persistence.  
**API impact**: Adds graph context endpoints/read models.  
**WebSocket impact**: Adds interaction ambiguity and suppression updates.  
**CI impact**: Adds graph consistency, identity fragmentation and invalid
window propagation tests.  
**Production impact**: Requires crowded, crossing, occlusion and re-entry
validation.  
**Evidence requirements**: interaction graph audit exports.  
**Benchmark requirements**: graph update latency and edge count distribution.  
**Rollback strategy**: Disable graph promotion and suppress interaction claims
without affecting single-person temporal states.  
**Acceptance criteria**: no strong interaction under unstable identity
continuity.  
**Explicit non-goals**: no GNN inference path.

### Wave 5 - Adaptive Baseline and Anomaly Accumulation

**Objectives**: Add bounded, explainable contextual baselines and anomaly
accumulation.  
**Architectural scope**: Adaptive Baseline Runtime and anomaly integration.  
**Files/modules impacted**: `backend/apps/anomalies/*`,
`backend/apps/behavior/*`, `backend/apps/runtime/*`,
`backend/apps/forensics/*`, `frontend/src/*`.  
**Runtime impact**: Adds baseline update, quarantine and drift telemetry.  
**DB impact**: Adds baseline snapshots, threshold shifts and drift states.  
**API impact**: Adds anomaly candidate context and baseline provenance.  
**WebSocket impact**: Adds anomaly candidate truth-state and baseline drift
indicators.  
**CI impact**: Adds baseline reproducibility, drift quarantine and
contamination tests.  
**Production impact**: Requires scientific validation before any maturity
claim.  
**Evidence requirements**: uncertainty audit, false-positive/false-negative
review and baseline replay report.  
**Benchmark requirements**: anomaly scoring latency and threshold-change
variance.  
**Rollback strategy**: Freeze baseline, supersede affected anomaly candidates
and revert to last valid snapshot.  
**Acceptance criteria**: every threshold shift is reconstructable.  
**Explicit non-goals**: no unsupervised self-modifying production behavior.

### Wave 6 - Decision Lineage, Replay and Forensic Review

**Objectives**: Make every semantic, temporal, episode, graph and anomaly
decision reconstructable.  
**Architectural scope**: Decision Lineage & Replay Engine.  
**Files/modules impacted**: `backend/apps/forensics/*`,
`backend/apps/behavior/*`, `backend/apps/anomalies/*`,
`tools/prod/*`, `scripts/ci/*`, `ci_evidence/*`.  
**Runtime impact**: Adds replay jobs and artifact authenticity checks.  
**DB impact**: Adds lineage indexes and replay references.  
**API impact**: Adds forensic trace lookup surfaces.  
**WebSocket impact**: Adds replay mismatch and stale artifact indicators.  
**CI impact**: Adds artifact digest, replay determinism and raw trace checks.  
**Production impact**: Enables production sign-off evidence packages.  
**Evidence requirements**: decision trace, temporal audit, episode lineage and
graph audit artifacts.  
**Benchmark requirements**: replay cost and trace size.  
**Rollback strategy**: Invalidate affected artifacts and regenerate from raw
trace.  
**Acceptance criteria**: accepted decisions are reconstructable from raw
lineage.  
**Explicit non-goals**: no UI-only forensic proof.

### Wave 7 - GPU/Queue Causality and Runtime Telemetry

**Objectives**: Attribute latency, queue wait, GPU utilization and runtime
degradation truth.  
**Architectural scope**: Runtime Telemetry Expansion and GPU/Queue Causality
Instrumentation.  
**Files/modules impacted**: `backend/apps/runtime/*`,
`backend/apps/pipeline/*`, `backend/apps/video_analysis/*`,
`tools/prod/*`, `scripts/ci/*`, `ci_evidence/*`.  
**Runtime impact**: Adds per-stage runtime event traces and budget alerts.  
**DB impact**: Adds benchmark/run telemetry records or artifact links.  
**API impact**: Adds telemetry status read models where needed.  
**WebSocket impact**: Adds runtime truth and degraded-state status updates.  
**CI impact**: Adds fake metric, 0% GPU contradiction and raw-trace checks.  
**Production impact**: Requires production runtime snapshot before claims.  
**Evidence requirements**: GPU, queue, decode, orchestration, batching and
model execution reports.  
**Benchmark requirements**: repeated runs with confidence intervals and
pipeline decomposition.  
**Rollback strategy**: Block performance claims; do not roll back functional
behavior unless runtime budgets are breached.  
**Acceptance criteria**: no benchmark summary without raw event traces.  
**Explicit non-goals**: no optimization without measured bottleneck.

### Wave 8 - Scientific Validation and Production Closure

**Objectives**: Validate representative datasets, soak, failure injection,
CI/CD parity, drift detection and final maturity evidence.  
**Architectural scope**: Dataset governance, benchmark integrity, CI/CD
runtime reconciliation and production readiness closure.  
**Files/modules impacted**: `ci_evidence/*`, `scripts/ci/*`,
`tools/prod/*`, `.specify/*`, `AGENTS.md`, `constitution.md`,
`backend/apps/*`, `frontend/src/*`.  
**Runtime impact**: Runs controlled production validation and evidence
regeneration.  
**DB impact**: Verifies migrations, indexes, reconciliation and PostgreSQL
authority.  
**API impact**: Freezes accepted contract versions.  
**WebSocket impact**: Validates frontend/backend runtime truth consistency.  
**CI impact**: Blocks hidden xfails, skipped gates, placeholders and CI/prod
mismatch.  
**Production impact**: Requires drift-free profile and rollback-ready
deployment.  
**Evidence requirements**: final immutable evidence package with raw traces,
digests and representative datasets.  
**Benchmark requirements**: final repeated-run report separating runtime
performance, scientific validity and operational stability.  
**Rollback strategy**: Freeze maturity claim, supersede invalid artifacts and
roll back deployed changes if runtime safety is affected.  
**Acceptance criteria**: all BSIL gates pass with no fake maturity vectors.  
**Explicit non-goals**: no closure by artifact presence or CI green alone.

## File-Level Execution Plan

| Area | Evolution | Responsibility boundary | Anti-patterns to avoid |
| --- | --- | --- | --- |
| `backend/apps/behavior/*` | Own semantic pose state, temporal state, episodes, graph context and behavior contracts | Behavioral logic and schemas only; no raw runtime orchestration ownership | god service, direct inference fallback, unversioned ontology |
| `backend/apps/anomalies/*` | Add adaptive baseline, anomaly accumulation, drift quarantine and false-positive review | Anomaly candidate scoring and baseline governance | hidden self-modifying thresholds, scientific claims without datasets |
| `backend/apps/tracking/*` | Provide identity continuity, fragmentation, re-entry and merge/split lineage | Identity authority inputs for behavior and graph gates | local track IDs as canonical truth |
| `backend/apps/pipeline/*` | Route semantic, temporal, graph and anomaly work through mode-isolated queues | Orchestration, admission, retry, DLQ and runtime reconciliation | implicit queue ownership, retry recursion |
| `backend/apps/video_analysis/*` | Continue source media, pose and temporal sequence materialization feeding BSIL | Frame/pose source authority and source-time envelopes | pose overlay treated as behavior |
| `backend/apps/runtime/*` | Add runtime anti-fragility, GPU/queue causality and drift checks | Runtime truth and telemetry authority | fake health, unavailable-as-zero metrics |
| `backend/apps/forensics/*` | Add decision lineage lookup, replay traces and evidence authenticity | Forensic reconstruction and audit surfaces | UI-only evidence, mutable trace history |
| `backend/apps/cameras/*` | Expose RTSP reconnect, camera/source health and live source truth to BSIL | Camera/source authority and reconnect state | reconnect loops without telemetry |
| `frontend/src/*` | Render truth states, confidence decomposition, uncertainty, ambiguity and stale/replay indicators | Review and operator presentation | accusatory language, frontend-only truth repair |
| `tools/prod/*` | Validate production runtime, workers, queues, Triton, evidence and drift | Production operational evidence collection | dev assumptions, malformed artifacts |
| `scripts/ci/*` | Enforce lineage, no fallback, PostgreSQL-only, artifact authenticity and benchmark causality gates | CI eligibility and regression prevention | CI green equals production-ready |
| `ci_evidence/*` | Store immutable evidence snapshots, raw traces, reports, manifests and digests | Evidence authority | placeholder folders, reused stale artifacts |
| `.specify/*` | Keep spec/plan/task governance aligned with constitution and feature scope | Planning and governance memory | template regressions, stale feature pointer |
| `AGENTS.md` | Reference active BSIL plan and production rules when the phase becomes active | Agent execution guidance | conflicting runtime instructions |
| `constitution.md` | Remain binding authority; update only if BSIL reveals new general law | Constitutional governance | weakening existing maturity gates |

Responsibilities must be separated by deterministic contracts. Inference paths
produce outputs and runtime events; persistence paths write through governed
services; orchestration paths route work and reconcile state; frontend paths
render truth without inventing it.

## Anti-Pattern Prevention

The plan explicitly forbids:

- giant god-orchestrator files;
- mixed runtime authority;
- hidden retry recursion;
- fake telemetry;
- heuristic drift without validation;
- uncontrolled queue growth;
- non-deterministic replay;
- runtime-only state without persistence strategy;
- model-version ambiguity;
- weak ontology governance;
- benchmark gaming;
- CI-green equals production-ready assumptions;
- direct database writes from inference paths;
- frontend state that masks backend/runtime failure.

## Acceptance + Evidence Strategy

| Acceptance level | Required proof |
| --- | --- |
| Development acceptance | unit/contract tests, heuristic labels, local replay, no placeholders |
| Staging acceptance | PostgreSQL-backed integration, queue isolation, replay determinism, artifact digests |
| Production acceptance | live/offline production runtime evidence, drift-free topology, rollback criteria |
| Behavioral validation | semantic confidence, temporal state, episode and graph evidence with uncertainty |
| Runtime causality | task, queue, inference, GPU/CPU, DB, artifact and UI reconciliation |
| Temporal consistency | source-time ordering, replay validation, invalid-window suppression |
| Identity continuity | fragmentation, re-entry, merge/split and confidence gates |
| Live RTSP resilience | reconnect, dropped frames, backlog and degraded-state evidence |
| Long-duration soak | minimum 8-hour soak for maturity closure and 30-minute live datasets |

Evidence artifacts must include freshness timestamp, schema version, source
manifest, artifact digest, runtime topology fingerprint, worker topology
fingerprint, model/config digest, dataset provenance hash, raw runtime traces,
replay validation result and benchmark run identifiers.

Representative validation requires at least three offline classroom datasets,
at least two live RTSP datasets, crowded interaction coverage,
occlusion/re-entry coverage and long-duration stability coverage. Repeated
benchmarks must report baseline/candidate causality, variance, confidence
intervals, failed-run accounting and raw trace references.

## Execution Layers

Each execution layer must produce objective evidence, rollback criteria,
reconciliation checks and anti-regression gates.

### 1. Semantic Runtime Core

**Objectives**: Define semantic pose states, attention categories, engagement,
posture, hand activity, desk interaction, confidence and uncertainty.  
**Risks**: heuristic labels presented as truth, missing pose lineage, low
confidence promoted to valid state.  
**Runtime gates**: lineage required, confidence required, invalid-pose
suppression, heuristic labeling.  
**Evidence requirements**: semantic traces over representative clear, partial
and occluded pose sequences.  
**Rollback conditions**: missing lineage, confidence decomposition absent,
semantic output generated from fallback payload.  
**Benchmark expectations**: semantic latency and orchestration overhead
reported separately.  
**Reconciliation checks**: semantic task state, durable state, artifact and UI
truth-state converge.  
**Anti-regression rules**: no placeholder semantic outputs and no deterministic
behavioral truth language.

### 2. Temporal State Engine

**Objectives**: Implement memory windows, smoothing, hysteresis, cooldown,
decay, spike suppression and long-duration continuity.  
**Risks**: one-frame escalation, temporal contamination, replay drift.  
**Runtime gates**: frame ordering, timestamp hierarchy, invalid-window
exclusion and uncertainty propagation.  
**Evidence requirements**: temporal replay audit, sustained-state examples,
spike-only rejection and continuity-gap handling.  
**Rollback conditions**: replay invalid, timestamp invalid or state mutation
without lineage.  
**Benchmark expectations**: aggregation latency and window counts by run.  
**Reconciliation checks**: transition events match durable state and replay
results.  
**Anti-regression rules**: no accepted state without replay-safe evidence.

### 3. Episode Lifecycle Engine

**Objectives**: Create, extend, close, suppress and supersede behavioral
episodes with immutable causal evidence chains.  
**Risks**: irreversible mutation, missing evidence, accusation semantics.  
**Runtime gates**: causal chain completeness, start/end timestamps,
rollback-safe update records.  
**Evidence requirements**: episode lineage report and suppression cases.  
**Rollback conditions**: episode includes unresolved lineage or invalid
window contamination.  
**Benchmark expectations**: episode update cost and artifact write latency.  
**Reconciliation checks**: episode state, task terminal state, artifact digest
and frontend status converge.  
**Anti-regression rules**: no episode from missing evidence.

### 4. Interaction Graph Runtime

**Objectives**: Produce interaction edges, synchronized attention context,
motion synchronization and crowd context.  
**Risks**: false association, graph propagation across invalid windows,
identity split/merge contamination.  
**Runtime gates**: identity continuity confidence, edge lifecycle, edge decay
and suppression lineage.  
**Evidence requirements**: interaction graph audit over crowded, crossing,
occlusion and re-entry samples.  
**Rollback conditions**: graph invalid, interaction inheritance after
fragmentation or unstable identity continuity.  
**Benchmark expectations**: graph update latency and edge count distribution.  
**Reconciliation checks**: edge records, graph cache, artifact and UI graph
state match.  
**Anti-regression rules**: no strong interaction claim under unstable identity.

### 5. Adaptive Baseline Runtime

**Objectives**: Establish session/person/context baselines, bounded threshold
adaptation and drift visibility.  
**Risks**: uncontrolled drift, contaminated baseline, anomaly inflation.  
**Runtime gates**: baseline snapshot, update lineage, drift budget, reset
policy and quarantine state.  
**Evidence requirements**: baseline reproducibility report and drift
quarantine cases.  
**Rollback conditions**: drift budget exceeded, update not reconstructable or
baseline contamination detected.  
**Benchmark expectations**: anomaly scoring latency and threshold-change
counts.  
**Reconciliation checks**: baseline version, anomaly score, threshold context
and evidence artifacts align.  
**Anti-regression rules**: no hidden self-modifying anomaly behavior.

### 6. Decision Lineage & Replay Engine

**Objectives**: Persist decision traces and deterministic replay for semantic
states, temporal windows, episodes, interactions and anomaly candidates.  
**Risks**: non-reconstructable behavior, mutable evidence, stale traces.  
**Runtime gates**: immutable digests, source manifests, runtime fingerprints
and replay drift limits.  
**Evidence requirements**: behavioral decision trace, temporal reasoning
audit, episode lineage and interaction graph audit.  
**Rollback conditions**: missing raw trace, replay mismatch or stale artifact.  
**Benchmark expectations**: replay time and trace size by scenario.  
**Reconciliation checks**: raw trace, derived artifact and durable record
cross-link.  
**Anti-regression rules**: no maturity claim without replay-backed evidence.

### 7. Runtime Telemetry Expansion

**Objectives**: Emit semantic, temporal, episode, graph, confidence,
uncertainty, queue, failure and reconciliation telemetry.  
**Risks**: fake health, unavailable-as-zero metrics, hidden degraded states.  
**Runtime gates**: truth-state metrics and failure taxonomy coverage.  
**Evidence requirements**: dashboard snapshots plus raw metric exports.  
**Rollback conditions**: telemetry invalid or degraded path unlabeled.  
**Benchmark expectations**: telemetry overhead bounded and reported.  
**Reconciliation checks**: telemetry state matches runtime and persistence.  
**Anti-regression rules**: no silent failure.

### 8. GPU/Queue Causality Instrumentation

**Objectives**: Attribute decode, orchestration, queue wait, tensor transfer,
model execution, batching and persistence cost.  
**Risks**: fake GPU utilization, constant causality metrics, hidden CPU
bottleneck.  
**Runtime gates**: raw runtime event traces and GPU/queue decomposition.  
**Evidence requirements**: GPU busy percent, effective batch size, batch
collapse, model starvation and queue saturation reports.  
**Rollback conditions**: 0% GPU utilization under heavy workload claim or
missing raw traces.  
**Benchmark expectations**: repeated-run statistics with confidence intervals.  
**Reconciliation checks**: benchmark run ID links to GPU, queue and artifact
events.  
**Anti-regression rules**: no synthetic decomposition values.

### 9. Long-Running Stability & Soak Validation

**Objectives**: Prove endurance under live, offline, degraded and reconnect
conditions.  
**Risks**: memory leak, queue leak, stale cache, worker crash, GPU thermal
instability.  
**Runtime gates**: 8-hour soak for maturity, 30-minute live samples, complete
offline samples.  
**Evidence requirements**: memory, Redis growth, queue age, cache pruning,
worker recycling and GPU thermal reports.  
**Rollback conditions**: unbounded growth, repeated worker crash or degraded
duration breach.  
**Benchmark expectations**: stability confidence intervals and variance
thresholds.  
**Reconciliation checks**: no orphan tasks, duplicate events or stale windows
after soak.  
**Anti-regression rules**: no short-run-only maturity closure.

### 10. Scientific Dataset Governance

**Objectives**: Govern representative datasets, labels, splits, uncertainty,
false-positive and false-negative review.  
**Risks**: dataset leakage, synthetic-only evidence, invalid labels.  
**Runtime gates**: dataset provenance hashes, label provenance, exclusion
rules and split policy.  
**Evidence requirements**: at least three offline and two live datasets plus
crowded, occlusion/re-entry and long-duration coverage.  
**Rollback conditions**: dataset provenance invalid or label protocol missing.  
**Benchmark expectations**: separated runtime, scientific and operational
metrics.  
**Reconciliation checks**: dataset manifests link to artifacts and results.  
**Anti-regression rules**: no scientific claim without dataset evidence.

### 11. Benchmark Integrity & Statistical Validation

**Objectives**: Produce realistic repeated-run benchmark reports.  
**Risks**: self-baselining, candidate-only pass, stale benchmark reuse.  
**Runtime gates**: baseline/candidate causality, raw traces, variance and
confidence intervals.  
**Evidence requirements**: benchmark provenance, run manifests and failure
accounting.  
**Rollback conditions**: benchmark invalid, stale or missing replay reference.  
**Benchmark expectations**: repeated runs with comparable workload and
reported variance.  
**Reconciliation checks**: benchmark summaries match raw event traces.  
**Anti-regression rules**: no benchmark summary without raw backing.

### 12. CI/CD Runtime Reconciliation

**Objectives**: Align CI, production topology, model/config repositories,
queues, migrations and worker routing.  
**Risks**: green CI but broken production behavior.  
**Runtime gates**: parity checks, production validation and drift detection.  
**Evidence requirements**: CI artifact signatures, environment fingerprints
and production runtime evidence.  
**Rollback conditions**: CI/runtime mismatch or production drift unresolved.  
**Benchmark expectations**: CI benchmark eligibility separated from
production benchmark acceptance.  
**Reconciliation checks**: branch/hash, dependency lock, migrations and runtime
profile match.  
**Anti-regression rules**: CI green is not production-ready automatically.

### 13. Production Drift Detection

**Objectives**: Detect changes in environment, runtime profile, workers,
models, queues, dependencies, GPU runtime and evidence paths.  
**Risks**: untracked override, stale worker route, wrong model artifact.  
**Runtime gates**: topology fingerprints and startup/runtime drift checks.  
**Evidence requirements**: drift snapshot before acceptance.  
**Rollback conditions**: unreconciled drift in any authoritative profile.  
**Benchmark expectations**: drift-free profile required before benchmark run.  
**Reconciliation checks**: fingerprint deltas explained or reverted.  
**Anti-regression rules**: no environment drift tolerance.

### 14. Failure Injection & Chaos Validation

**Objectives**: Verify fail-closed behavior for runtime, queue, lineage,
timestamp, graph, baseline and telemetry failures.  
**Risks**: hidden fallback, retry amplification, orphan state.  
**Runtime gates**: failure taxonomy emits telemetry and acceptance behavior.  
**Evidence requirements**: injected failure reports and reconciliation
snapshots.  
**Rollback conditions**: any injected critical failure continues as valid
behavioral output.  
**Benchmark expectations**: degraded overhead and recovery time reported.  
**Reconciliation checks**: terminal failure states converge after injection.  
**Anti-regression rules**: no silent degraded path.

### 15. Production Readiness Closure

**Objectives**: Close BSIL only when runtime, evidence, science, CI,
observability, deployment and rollback gates pass.  
**Risks**: maturity theater, placeholder evidence, unresolved xfails.  
**Runtime gates**: all constitutional and BSIL acceptance gates pass.  
**Evidence requirements**: immutable final evidence snapshot with digests and
raw traces.  
**Rollback conditions**: any critical gate failure, hidden xfail, unresolved
reconciliation or invalid evidence.  
**Benchmark expectations**: final benchmark report separates runtime
performance, scientific validity and operational stability.  
**Reconciliation checks**: task, database, queue, artifact, telemetry, UI and
production topology converge.  
**Anti-regression rules**: no fake maturity closure.
