<!--
SYNC IMPACT REPORT
==================
Version change: 2.1.0 -> 2.2.0
Bump rationale: MINOR - Adds the active Heterogeneous Production Runtime
Maturity governance model, explicitly separating Windows/Docker development
validation from native Linux RTX 5090 production authority, and adds binding
rules for replay policy, branch synchronization, runtime preflight and
production closure evidence.

Modified principles:
- Production runtime authority -> Heterogeneous production runtime authority.
- Evidence-based acceptance -> Replay and production evidence integrity.
- Benchmark/scientific acceptance -> RTX 5090 production authority for
  throughput and inference claims.
- Compliance review -> Branch parity, preflight, lifecycle, GPU telemetry,
  causality and final closure evidence gates.

Added sections:
- Heterogeneous Production Runtime Maturity Constitution
- Heterogeneous Runtime Maturity Compliance Gates
- Foundational System Doctrine
- Production Runtime Constitution
- Temporal Truth Constitution
- Identity Continuity Constitution
- Pose Runtime Constitution
- Behavioral Intelligence Constitution
- Observability and Scientific Rigor Constitution
- Queue, Orchestration, and Resilience Constitution
- API, Contract, and Schema Constitution
- Data and Storage Constitution
- Security, Stability, and Failure Constitution
- Acceptance, Validation, and Evidence Constitution
- Cross-Wave Dependency Constitution
- Anti-Regression Governance Constitution
- Runtime Truth Governance
- Production Evidence Integrity
- Benchmark Scientific Rigor
- CI/CD Authority Model
- Dev/Prod Parity Governance
- Runtime Reconciliation Guarantees
- GPU Utilization & Inference Efficiency Governance
- Queue/Retry/Backpressure Truth Governance
- Identity & Temporal Integrity Governance
- Observability Truth Enforcement
- Feature Closure Requirements
- Technical Debt Budget Governance
- XFail Governance Policy
- Runtime Drift Detection
- Artifact Authenticity Validation
- Deployment Safety Rules
- Production Rollback Governance
- Forensic Traceability Requirements
- AI/Behavioral Intelligence Scientific Integrity
- Future Expansion Safety Rules
- Definition of Production-Ready
- Definition of Scientifically Valid
- No Silent Failure Doctrine
- Anti-Regression Enforcement Matrix
- Final Architectural Positioning

Removed sections:
- Former UI-theme, generic per-file diagram, and universal commit directives
  as constitutional principles. Normal repository contribution guidance
  remains subordinate to the runtime and evidence laws in this document.

Templates requiring updates:
- updated: .specify/templates/plan-template.md
- updated: .specify/templates/spec-template.md
- updated: .specify/templates/tasks-template.md
- not present: .specify/templates/commands/*.md
- reviewed/no change: docs/backend/architecture/triton-operations.md
- reviewed/no change: docs/backend/architecture/data-flow.md
- reviewed/no change: docs/backend/architecture/deployment-topology.md
- reviewed/no change: docs/backend/architecture/observability-runbook.md
- reviewed/no change: docs/ARCHITECTURE.md
- reviewed/no change: README.md
- reviewed/no change: docs/linux_production_optimization_execution_phases.md
- reviewed/already aligned: AGENTS.md

Follow-up TODOs:
- Existing evidence generated before this amendment remains historical only
  until it is reconciled against the new anti-regression matrix.
- Implementation gaps identified by this constitution are acceptance blockers
  for affected maturity claims; they are not placeholder governance text.
- The active runtime maturity plan remains not completed until real production
  evidence is captured under `ci_evidence/production/runtime_maturity/final/`.
-->

# Production Behavioral Intelligence Platform Constitution

## Core Principles

### Credential Evidence Boundary

CI inspection credentials MUST be sourced from the untracked local path
`.local/secrets/github.env` via `GITHUB_TOKEN`. Raw tokens MUST NOT appear in
tracked governance, evidence, logs, or acceptance artifacts. Any credential
shared through chat or captured in output is compromised and MUST be rotated
before use for maturity closure.

### 1. Foundational System Doctrine

#### 1.1 System Identity and Architectural Philosophy

This repository governs a production-grade temporal behavioral intelligence
platform. It ingests live and offline video, executes GPU-backed detection and
RTMPose inference through Triton, preserves person identity through time,
extracts interpretable temporal features, exposes governed evidence, and
eventually enables anomaly, contrastive, graph, transformer, teacher-student,
and multimodal/VLM pipelines.

The architecture MUST be temporal-first, identity-scoped, evidence-producing,
and fail-closed for scientific claims. A feature is not mature because it
renders an overlay, produces a JSON artifact, or returns an HTTP success
response. It is mature only when its inputs, runtime route, time basis,
identity basis, model version, transformation history, confidence semantics,
failure semantics, and validation evidence are recoverable and reviewable.

The platform has five constitutional authority layers:

| Authority layer | Binding responsibility | Prohibited substitution |
| --- | --- | --- |
| Runtime authority | Select and validate the real production inference route | Local/mock inference presented as production |
| Temporal authority | Preserve ordering, duration, gaps, replay semantics | Frame index presented as source time without provenance |
| Identity authority | Scope and audit who a sequence describes | Track labels assumed stable without continuity proof |
| Scientific authority | Define behavior meaning, quality, and claim evidence | UI appearance or synthetic metrics used as validation |
| Contract authority | Version schemas, artifacts, and forensic lineage | Silent payload, serializer, or artifact drift |

#### 1.2 Production Behavioral Intelligence Definition

Production behavioral intelligence is the ability to derive reviewable,
time-bounded behavioral observations from identity-continuous, timestamp-valid,
pose- and context-supported sequences under a validated production runtime.
Each output MUST state whether it is an observation, feature, heuristic event,
model inference, anomaly candidate, or adjudicated outcome. No output MAY imply
intent, misconduct, or anomaly truth merely from position, pose, attention
direction, or a statistical score.

A behavioral output is eligible for production use only when it carries:

| Required provenance | Minimum content |
| --- | --- |
| Subject scope | session, camera, canonical identity or explicitly unresolved identity |
| Temporal scope | window start/end in canonical time, coverage, gaps, drift state |
| Input authority | source frames or sequence reference, pose stream, feature version |
| Runtime authority | active mode, Triton model/version/config, hardware/run identifier |
| Decision semantics | ontology label, confidence, ambiguity/missing-data state |
| Traceability | event ID, contract version, persistence/artifact references |

#### 1.3 Temporal-First Reasoning Doctrine

Frame-centric detections answer where a candidate body region appears at an
instant. Behavior requires duration, recurrence, rate of change, transition
order, interactions, recovery after missing data, and reference to preceding
context. Therefore:

- Single-frame results MUST NOT be classified as temporal behavior.
- Temporal features MUST be computed over explicit windows with source-time
  boundaries, sample coverage, gap masks, and feature-definition versions.
- Every behavioral event MUST link to the ordered observations that support it.
- Loss of continuity MUST reduce confidence or invalidate a window; it MUST NOT
  be bridged silently for a visually smooth overlay.

Frame-level person detection and pose estimation are necessary substrates, but
pose overlays are not behavioral intelligence. An overlay is a presentation
artifact. It does not prove gaze duration, repeated wrist activity, sustained
posture deviation, interaction sequence, anomalous transition, or causal
meaning.

#### 1.4 Identity Continuity Doctrine

Behavior attaches to a temporally coherent subject, not to an arbitrary local
tracker label. Any sequence containing unacknowledged identity switches,
cross-camera collisions, unsafe merges, or unresolved association gaps is
ineligible for behavioral model training, anomaly scoring, or maturity claims.

Identity continuity MUST be:

- scoped to session and camera before any wider aliasing;
- represented separately from local tracking assignment;
- measured with switch, fragmentation, occlusion recovery, and re-entry
  recovery metrics;
- auditable through lifecycle transitions and ReID evidence;
- conservative under uncertainty, preferring unresolved gaps to wrong merges.

#### 1.5 Scientific Reproducibility Doctrine

Every performance or behavioral-quality claim MUST be reproducible from a
versioned input manifest, source media digest or approved live-capture
manifest, runtime configuration, model artifacts, schema/ontology versions,
code revision, environment/hardware description, and raw metric output.

Production acceptance MAY use bounded operational thresholds with repeated
real runs and confidence intervals. Research, thesis, or comparative algorithm
claims MUST additionally disclose study design, variance, effect size,
statistical method, and limitations. A demonstration, mock run, screen capture,
or single successful execution is not scientific evidence.

#### 1.6 Observability Truth Doctrine

Telemetry is an evidence source, not decoration. A metric MUST be emitted from
measured execution, a validated aggregation of measured events, or an explicit
unknown/unavailable state. The system MUST NOT synthesize availability,
fabricate healthy status, convert missing values to measured zeros, or hide
failed sub-stages behind aggregate success.

All operator and forensic surfaces MUST distinguish:

| State | Meaning |
| --- | --- |
| measured zero | Source ran successfully and observed no occurrences |
| unavailable | Source could not provide a measurement |
| unknown | Measurement has not been established for this interval |
| stale | Last value exists but exceeds freshness contract |
| degraded | Output exists but one or more required authorities failed |
| valid | Measurement satisfies its source and freshness contract |

#### 1.7 Deployment Authority Doctrine

Production deployment is native Linux, NVIDIA GPU-backed, Triton-only, with no
Docker dependency and no sudo assumption. User-space processes, service
scripts, environment files, model repositories, queue workers, and health
checks MUST operate under the deployed account's permissions. Docker MAY be
used in development or controlled testing but MUST NOT be an unstated
production precondition.

Production relational persistence authority is PostgreSQL only. SQLite MUST
NOT be used as a production runtime database, benchmark database, acceptance
database, migration validation authority, or fallback execution database. Any
path that passes under SQLite but has not been validated against PostgreSQL is
non-authoritative and MUST NOT be used to claim runtime readiness, maturity
closure, compatibility, or scientific reproducibility.

For the current production estate, CUDA authority is fixed at **CUDA 12.8**.
All GPU-serving dependencies (Triton runtime line, TensorRT runtime/Python,
engine builds, and backend integration libraries) MUST remain version-compatible
with CUDA 12.8 unless an explicit migration plan updates this constitution.
For the current pinned production runtime, Triton binary authority is
`/home/bamby/services/triton_build_r2502/tritonserver/install/bin/tritonserver`.

Production shell resolution is constitutional runtime authority. User-local
path pinning in `~/.bashrc` and `~/.profile` MUST provide deterministic
resolution for Python (backend venv), Triton server binary, and package cache
location. Conflicting or duplicate exports that create non-deterministic
runtime behavior are operational drift and MUST be remediated.

#### 1.8 Runtime Determinism Doctrine

Runtime choices MUST be explicit, versioned, diagnosable, and stable for the
duration of an accepted run. Mode selection, endpoint binding, queue route,
model config, batching policy, feature version, artifact naming, and
degradation state MUST be reconstructable. Environment ambiguity, fallback
chosen without recorded authority, or mixed-mode results invalidate acceptance
evidence.

### 2. Production Runtime Constitution

#### 2.1 Triton Runtime Authority

Triton Inference Server is the sole production authority for detector,
RTMPose, and any future deployed GPU-backed behavior model inference. A local
PyTorch, ONNX Runtime, OpenVINO, mock, stub, cached prediction, or frontend
calculation MAY support development tests or explicitly labeled experiments,
but MUST NOT:

- satisfy production readiness;
- be reported as a production prediction;
- silently substitute for an unavailable Triton result;
- populate a benchmark candidate identified as a production run;
- create a behavior, anomaly, or forensic claim marked production-valid.

If required Triton inference is unavailable in production, the inference
operation MUST fail closed or produce an explicitly degraded, non-authoritative
artifact that cannot be promoted to behavioral evidence.

#### 2.2 Endpoint and Execution Mode Governance

The platform maintains two defined endpoint profiles:

| Mode profile | HTTP / gRPC / metrics ports | Scheduling intent | Production activation |
| --- | --- | --- | --- |
| `live` | `39000 / 39001 / 39002` | latency-first RTSP/live processing | Active only when selected |
| `offline` | `39100 / 39101 / 39102` | throughput-first offline processing | Active only when selected |

The phrase dual Triton endpoint architecture means both endpoint profiles,
model configuration families, and route contracts exist. It does not authorize
two live Triton server processes in production. Production MUST run exactly one
active Triton endpoint profile at a time. The inactive profile MUST be
unreachable and MUST receive no inference, health acceptance, queue dispatch,
benchmark samples, or scheduler traffic.

When the operational deployment binds both application workload URL settings
to the selected endpoint, that binding MUST be recorded in the startup
diagnostic manifest. Workloads incompatible with the selected mode MUST be
rejected, queued for later activation, or explicitly run in a separately
controlled mode window; they MUST NOT cause a second endpoint to start.

The selected profile MUST bind to one declared Triton binary path for the run.
That binary path and effective dependency chain MUST be captured in startup
metadata for reproducibility and incident forensics.

#### 2.3 Startup Validation Rules

Before backend workers accept production inference work, an automated preflight
MUST execute and persist its result. Startup is successful only when every
required check passes:

| Gate | Required verification | Rejection condition |
| --- | --- | --- |
| Deployment gate | Linux host, intended environment, non-root operational account | Docker-only/sudo-dependent assumption or wrong environment |
| Mode gate | Exactly one allowed mode value and one selected profile | Missing, unknown, contradictory, or multi-active mode |
| Port gate | Active ports bound; inactive profile ports unbound | Both profiles reachable or active profile missing |
| Triton gate | Active `/v2/health/live` and `/v2/health/ready` succeed | Non-ready active server |
| Model gate | Required detector/RTMPose model readiness, version and config match selected profile | Missing, wrong config, wrong I/O, failed warmup |
| GPU gate | Required GPU visible; memory/capability sufficient for loaded models | CPU/local substitution or capacity rejection |
| Queue gate | Only allowed mode routes are enabled; worker queues and concurrency declared | Cross-mode consumption or uncontrolled queue |
| Schema gate | Contract, event, artifact and feature versions configured | Unversioned payload/persistence |
| Telemetry gate | Readiness and failure emission stores/probes are writable | Readiness cannot be measured |
| Toolchain gate | `python` resolves to backend venv; Triton resolves to pinned binary; TensorRT import/version check passes | Ambiguous path, wrong interpreter/binary, or incompatible TensorRT runtime |
| Engine compatibility gate | Required models load `READY` with runtime-compatible serialized plans | Version-tag/serialization mismatch or model load failure |

An invalid configuration MUST terminate startup or keep affected workers out of
service. Logging a warning and continuing is prohibited when a required
authority gate fails.

#### 2.4 Health Validation Hierarchy

Health is hierarchical and MUST NOT be collapsed into one boolean:

1. Process liveness proves the selected Triton process exists and responds.
2. Triton readiness proves the selected server accepts requests.
3. Model readiness proves each required versioned model is loaded.
4. Contract readiness proves expected input/output tensor signatures and
   configuration profiles match application expectations.
5. Functional canary readiness proves a real sanctioned inference sample
   returns schema-valid output.
6. Pipeline readiness proves enqueue, worker pickup, inference, persistence,
   telemetry, and contract emission operate.
7. Behavioral readiness proves temporal and identity authorities are valid for
   feature or anomaly usage.

A higher-level ready state MUST NOT be emitted unless every lower required
level is valid or the higher level explicitly states its unavailable/degraded
dependency.

#### 2.5 Startup, Shutdown, GPU, Scheduler, and Queue Ownership

The active mode owns its Triton process, loaded model configuration, GPU
capacity reservation, scheduler parameters, and accepted workload queues for
the activation interval. A mode change is an operational cutover, not an
ordinary request-level branch.

- Startup MUST drain or reject stale incompatible work before declaring the
  selected mode ready.
- Shutdown MUST stop admission, record cutover intent, drain or terminate
  bounded in-flight work according to policy, flush evidence/telemetry, and
  verify endpoint unreachability before activating another profile.
- GPU overcommit, OOM, allocation failure, model unload, or unexpected GPU
  reset MUST place the affected mode into failure or degraded state and MUST
  invalidate incomplete benchmark/behavior evidence.
- Scheduler parameters such as batching, queue delay, instance group, memory
  pool, and priority MUST be profile-versioned and captured in manifests.
- Celery queues MUST be declared by mode and stage. Live workers MUST NOT
  consume offline inference queues and offline workers MUST NOT consume live
  inference queues during production operation.

Constitutional queue families include:

| Workload | Detector queue | Pose queue | Behavior queue |
| --- | --- | --- | --- |
| Live | `pipeline.live.person_detector.worker` | `pipeline.live.rtmpose_model.worker` | `pipeline.live.behavior.worker` |
| Offline | `pipeline.offline.person_detector.worker` | `pipeline.offline.rtmpose_model.worker` | `pipeline.offline.behavior.worker` |

#### 2.6 Runtime Fault Classes and Isolation Semantics

Every runtime failure MUST be classified, exposed, and handled using a bounded
policy:

| Fault class | Example | Permitted handling | Evidence consequence |
| --- | --- | --- | --- |
| Configuration fatal | Invalid mode, wrong endpoint/profile binding | Reject startup | No production run exists |
| Dependency fatal | Selected Triton/model not ready | Reject or stop admission | No authoritative inference |
| Capacity degradation | Queue SLO breach, GPU pressure, bounded timeout | Shed/throttle with explicit state | Exclude degraded windows unless policy accepts them |
| Data-local failure | Corrupt frame, failed crop, missing timestamp | Preserve valid siblings; mark item failure | Invalidate affected item/window only |
| Integrity failure | Identity collision, schema mismatch, event duplication corruption | Quarantine/fail stop | Block behavior/scientific maturity |
| Security failure | Unauthorized data/event access, tampering | Isolate and audit | Block release pending incident handling |

Silent runtime degradation, local production fallback, continued admission
after fatal authority loss, simultaneous profile activity, and unrecorded
manual routing changes are prohibited.

### 3. Temporal Truth Constitution

#### 3.1 Canonical Timestamp Semantics

Every frame, crop inference, pose observation, identity transition, feature
window, event, artifact, and benchmark sample MUST identify its time basis. The
canonical behavior time is `source_timestamp`, derived from the source stream
or media timeline and never silently replaced by processing wall-clock time.

| Timestamp | Definition | Required use |
| --- | --- | --- |
| `source_timestamp` | Media presentation/capture time on the source timeline | Ordering, windows, behavioral duration, replay |
| `ingest_timestamp` | Monotonic/UTC receipt time at system ingress | Network/decoder delay and operational latency |
| `queue_timestamp` | Enqueue time and dequeue/pickup time for a task | Queue wait and starvation analysis |
| `inference_timestamp` | Start/end time of actual model execution/request | Compute/service latency and timeout analysis |
| `persistence_timestamp` | Commit/write completion time for durable output | Evidence freshness and end-to-end latency |

All persisted time-bearing records MUST store timezone semantics for wall-clock
fields and units for timeline values. Millisecond integers without declared
origin, unit, and time basis are insufficient for scientific exports.

#### 3.2 Timestamp Propagation Contract

The ingest boundary MUST create or validate a temporal envelope containing:
`source_id`, `session_id`, `camera_id` where applicable, `frame_id`,
`source_timestamp`, `ingest_timestamp`, time-base metadata, frame sequence
number, and correlation ID. Each downstream stage MUST propagate the envelope,
append its stage timestamps and outcome, and MUST NOT rewrite earlier
timestamps.

When a source cannot provide genuine capture timestamps:

- offline media MUST derive source time from decoded presentation timestamp,
  time base, and file digest;
- live media MUST record the timestamp origin as derived/arrival-based, report
  the limitation, and exclude unsupported precision claims;
- any correction, synchronization, or drift adjustment MUST retain original
  and corrected values plus method/version.

#### 3.3 Temporal Continuity and Sequence Integrity Laws

A canonical sequence is ordered by source time within a subject scope. It MUST
carry expected cadence, observed cadence, gaps, drops, duplicate handling,
reordering decisions, interpolation markers, and coverage ratio.

- No sequence window MAY contain records from different sessions, cameras, or
  canonical identities unless its schema explicitly represents an interaction.
- Duplicate event or frame keys MUST be idempotently rejected or linked as
  retries; they MUST NOT inflate counts or durations.
- Out-of-order arrivals MAY be reordered only within an explicitly bounded
  watermark; late arrivals beyond that watermark MUST be recorded as late and
  handled according to window finalization policy.
- Dropped or failed frames MUST remain visible as missing intervals with
  reasons. A gap MUST NOT disappear because display interpolation exists.
- Interpolated values MUST never overwrite measured values or be exported as
  raw measurements.

#### 3.4 Temporal Memory Authority and Replay Guarantees

Rolling temporal memory is an identity-scoped materialization, never the sole
source of truth. Durable observation and lifecycle records remain authoritative
for reconstruction. Redis/cache state MAY accelerate active sessions but MUST
be rebuildable and MUST use keys including runtime mode, session, camera, and
canonical/local identity scope as applicable.

A replay MUST pin:

- source input manifest and digest;
- decoder/time-base version and correction policy;
- event/schema/pose/feature/ontology versions;
- identity decisions and any permitted reevaluation policy;
- selected runtime/model/configuration when replaying inference;
- deterministic seed or nondeterminism disclosure;
- replay identifier linking derived output to its parent evidence.

Replay output is invalid if it silently changes model versions, identity merge
decisions, time correction, excluded intervals, feature definitions, or data
filters relative to its declared manifest.

#### 3.5 Temporal Corruption and Drift Governance

The following are temporal corruption conditions and MUST block affected
behavior claims until resolved or explicitly excluded:

| Invalid state | Required action |
| --- | --- |
| Missing source timestamp origin | Mark non-authoritative for temporal behavior |
| Timestamp regression beyond permitted reorder window | Quarantine interval and emit integrity failure |
| Duplicate persisted frame/event treated as new | Correct aggregation and invalidate contaminated metric |
| Source-to-ingest drift exceeds configured per-mode tolerance | Mark drift breach and invalidate unsupported latency/sequence claims |
| Inference/persistence time predates required upstream event without clock explanation | Fail validation |
| Gap bridged without missing/interpolation marker | Invalidate sequence/export |
| Mode cutover mixed inside a benchmark or feature window | Split or reject window |

Drift tolerances MUST be numeric, mode-specific, versioned, and justified in a
plan or acceptance artifact. This constitution does not invent acceptable
numeric values; a feature cannot be accepted until its plan defines and tests
them using representative data.

### 4. Identity Continuity Constitution

#### 4.1 Canonical Identity Schema and Namespace Rules

The canonical identity key for person-specific temporal evidence MUST include:

```text
identity_scope = session_id + camera_id + canonical_track_id
observation_key = identity_scope + source_timestamp + frame_id
```

`local_track_id` is the immediate tracker assignment in one camera stream.
`canonical_track_id` is the continuity identity used for sequences only after
provenance rules are satisfied. A local ID MUST NOT be assumed globally unique
within a session, across cameras, across jobs, across reconnects, or across
runtime activations.

Each identity-bearing record MUST expose:

| Field | Requirement |
| --- | --- |
| `session_id` / `job_id` | Defines processing lifecycle boundary |
| `camera_id` or offline source ID | Prevents multi-source collision |
| `local_track_id` | Preserves tracker output provenance |
| `canonical_track_id` | Present only for authorized continuity identity |
| `lifecycle_state` | `active`, `occluded`, `reidentified`, `lost`, `ended`, or `unresolved` |
| `identity_confidence` | Calibrated score/state and threshold metadata |
| `identity_provenance` | Algorithm, version, source observations, decision reason |
| `decision_timestamp` | Time the association/merge was committed |

#### 4.2 Multi-Camera Isolation and Cross-Talk Prevention

Camera namespaces are isolated by default. Equal local IDs from two cameras
MUST resolve to different scoped identities unless an explicit cross-camera
ReID feature is approved, versioned, audited, and evidence-backed. Cache keys,
WebSocket groups, artifacts, database uniqueness constraints, event keys, and
frontend routes MUST preserve camera scope.

Cross-camera or cross-session merges are forbidden by default for behavioral
evidence. An implementation claiming such continuity MUST introduce a separate
governed contract and validation protocol; ordinary single-camera ReID
thresholds cannot authorize it.

#### 4.3 ReID Authority and Merge Semantics

ReID is a conservative aliasing authority, not a convenience label generator.
It MAY establish a canonical alias only when all policy predicates pass:

1. camera and session scope permit comparison;
2. lifecycle transition is physically and temporally plausible;
3. similarity/confidence meets a versioned calibrated threshold;
4. competing candidates do not create ambiguity above the conflict tolerance;
5. observation quality is sufficient and no integrity hold exists.

If predicates do not all pass, the system MUST persist an unresolved candidate
rather than forcing a merge. The candidate record MUST retain source local
track, possible canonical identity, scores, thresholds, features/model
version, reason for non-merge, and resolution status.

A merge MUST be append-only/auditable: original local observations remain
immutable; canonical identity is attached through a decision record. A
reversal MUST create a compensating identity decision and identify derived
windows or artifacts requiring invalidation/recomputation.

#### 4.4 Lifecycle and Occlusion Recovery Doctrine

Lifecycle events MUST reflect observed continuity:

| Transition | Required evidence |
| --- | --- |
| `active -> occluded` | Last valid observation and occlusion/gap trigger |
| `occluded -> reidentified` | ReID/association evidence meeting policy |
| `occluded -> lost` | Bounded recovery window exceeded or confidence failure |
| `active/reidentified -> ended` | Stream/job termination or explicit exit |
| any -> `unresolved` | Conflict, ambiguity, collision, or integrity hold |

Occlusion recovery MUST NOT treat proximity or box order alone as identity
proof in crowded or crossing scenes. Association MAY use IoU, motion,
appearance/ReID, pose consistency, and lifecycle priors only when each factor
has documented semantics and thresholds. Unsafe interpolation MUST be rejected.

#### 4.5 Forbidden Identity States and Trust Scoring

Forbidden states include identity without camera scope, a canonical identity
created without provenance, one local observation simultaneously assigned to
multiple canonical identities, merged identities across incompatible sessions,
silent ID-switch repair, behavior windows spanning unresolved identity
conflicts, and metrics that conceal identity failures.

Identity trust scoring MUST report at least:

- ID switch count/rate;
- fragmentation rate and average uninterrupted duration;
- unresolved association count/rate;
- occlusion recovery success and false-recovery review outcomes;
- re-entry recovery rate;
- merge/reversal counts and confidence distribution;
- proportion of windows eligible for behavioral inference.

Thresholds for behavioral eligibility MUST be stated in a feature plan and
validated on representative multi-person, occlusion, and crossing evidence.

### 5. Pose Runtime Constitution

#### 5.1 RTMPose Authority and Model Contract

RTMPose is the pose-estimation authority for governed pose streams unless a
future constitutional amendment approves an alternative. Production RTMPose
inference MUST be served through the selected active Triton profile. Model
repository version, TensorRT/serialized artifact digest, tensor names,
dimensions, coordinate system, skeleton/keypoint order, confidence definition,
preprocessing, postprocessing, and warmup/batching configuration MUST be
captured for acceptance.

A detector crop, person association, and RTMPose result MUST carry compatible
frame, identity, and source-time envelopes. A pose attached to the wrong
person, wrong frame, or unknown crop is not partial success; it is an integrity
failure for that record.

#### 5.2 Canonical Pose Streams

Three distinct streams MUST be represented when used:

| Stream | Definition | Intended usage | Scientific authority |
| --- | --- | --- | --- |
| `raw_keypoints` | Direct validated RTMPose output plus confidence/visibility, without temporal smoothing | Reproducibility, model evaluation, downstream reprocessing | Primary measurement authority |
| `smoothed_keypoints` | Temporally filtered transform of raw output with method/version/parameters and masks | Approved feature extraction where method is declared | Derived authority only |
| `display_keypoints` | Visibility-filtered and presentation-adjusted points for overlay/UI | Rendering and operator visualization | Never scientific input unless separately validated |

The streams MUST NOT be conflated. UI-ready filtering or smoothing MUST NOT
overwrite raw output. Behavioral feature definitions MUST name the accepted
stream and its transform version. Exports MUST preserve masks and lineage from
derived streams to raw inputs.

#### 5.3 Visibility, Smoothing, and Interpolation Semantics

Each keypoint record MUST contain a score or explicit absence and an approved
visibility threshold contract. A keypoint below threshold is missing/low
visibility; it MUST NOT be converted to a confident coordinate for behavior
features.

Smoothing MUST declare filter algorithm, parameters, warmup, causal versus
acausal behavior, handling of gaps, and induced latency. Offline acausal
smoothing cannot be represented as live-real-time behavior without disclosure.
Interpolation MUST:

- be bounded in time and only bridge explicitly allowed short missing intervals;
- identify each interpolated sample and its source anchors;
- be blocked across identity discontinuity, mode cutover, invalid source time,
  or confidence failure;
- never count as raw pose evidence;
- be excluded or separately evaluated in quality metrics unless the metric
  explicitly addresses interpolation.

#### 5.4 Batch Inference and Partial Failure Laws

Batching is permitted only when each crop/result retains deterministic mapping
to frame, identity candidate, source time, batch index, and result status.
Dynamic batching or concurrent execution MUST NOT alter result attribution.

When one item in a batch fails:

- valid siblings MAY be persisted with `success` and batch provenance;
- the failed item MUST be persisted or emitted as failed/missing with reason;
- a whole-frame behavior window MUST evaluate resulting coverage and
  eligibility, not assume success because some poses exist;
- a contract mismatch, response-shape ambiguity, or mapping failure MUST fail
  the affected batch rather than guess.

#### 5.5 Fallback Governance and Behavioral Readiness

Production local model fallback is prohibited. Permitted degradation consists
of explicit missing pose, bounded previously measured display continuation for
UI only, or failure state publication. A reused or interpolated display pose
MUST NOT silently feed behavioral features.

Pose output becomes behavior-ready only when model contract validation, stream
lineage, identity continuity, temporal coverage, visibility/missingness, and
quality acceptance thresholds pass. Pose-overlay availability alone is not a
maturity criterion.

### 6. Behavioral Intelligence Constitution

#### 6.1 Behavioral Ontology Governance

Behavioral semantics MUST reside in a versioned ontology and feature registry,
not in implicit frontend labels, ad hoc Python conditions, or undocumented
model classes. Each ontology entry MUST define:

| Definition component | Required content |
| --- | --- |
| Name and version | Stable identifier and semantic version |
| Claim class | observation, derived feature, event candidate, anomaly score, adjudicated label |
| Required input | pose stream, identity trust, context, time window, visibility coverage |
| Operational definition | measurable condition or model contract |
| Confidence semantics | calibration/rule thresholds and ambiguity state |
| Invalidating conditions | missing data, identity fault, drift, low coverage, unavailable context |
| Ethical interpretation | what output does not prove |

Implicit behavioral semantics are forbidden. Labels such as suspicious,
cheating, inattentive, interaction, anomaly, or risk MUST NOT be emitted
without ontology ownership, evidence links, and approved interpretation limits.

#### 6.2 Temporal Feature Authority

Features are computed from canonical identity-scoped windows with declared time
bounds, sample policy, selected pose stream, missingness mask, context
requirements, and feature implementation version. Feature computation MUST be
deterministic for identical input records and versioned configuration, or must
record nondeterminism and seeds.

Mandatory feature families for any behavior-readiness claim are:

| Family | Examples of valid feature concepts | Critical guards |
| --- | --- | --- |
| Head behavior | orientation proxy, turn duration, repeated direction transition | visibility, calibration, no intent inference |
| Wrist behavior | hand elevation, disappearance duration, hand-to-region motion | occlusion, crop quality, context boundary |
| Motion behavior | velocity, acceleration, stillness duration, periodic movement | source-time accuracy, gap masks |
| Posture behavior | torso lean, pose symmetry, sustained posture transition | skeleton validity, camera perspective limitation |
| Interaction behavior | relative proximity/motion or synchronized temporal events | separate identities, same calibrated scene/time |

Feature output MUST contain coverage, quality flags, source interval, identity
trust, pose stream version, ontology version, and reason for suppression where
not emitted.

#### 6.3 Temporal Window and Missing-Data Semantics

Window definitions MUST specify duration, stride, minimum measurements,
permitted gap length, visibility requirement, late-arrival closure policy, and
whether interpolation contributes. Missing observations MUST be represented by
masks/reasons and MUST NOT default to zero movement, neutral posture, absent
interaction, or normal behavior.

A feature or behavioral event MUST be invalid/suppressed when:

- identity is unresolved or trust is below approved threshold;
- source time is invalid or coverage/drift exceeds window policy;
- pose stream is missing, wrong-versioned, or insufficiently visible;
- required context, baseline, or interacting identity is absent;
- data was generated by unauthorized runtime fallback;
- ontology/schema/model version is unknown;
- a conflict makes more than one interpretation plausible and no ambiguity
  output is defined.

#### 6.4 Anomaly and Sequence Intelligence Semantics

Anomaly output is a deviation score or event candidate relative to a documented
reference distribution, not an automatic finding about a student. Before
learned anomaly deployment, interpretable feature and primitive-event outputs
MUST exist so decisions can be audited. An anomaly model MUST name its training
dataset manifest, split policy, feature/pose/identity eligibility filters,
model digest, calibration procedure, threshold policy, limitations, and
monitoring/drift plan.

Future temporal transformers, ST-GCN/CTR-GCN models, contrastive encoders,
interaction graphs, teacher-student models, or VLM fusion MUST consume governed
sequence contracts. They MUST NOT bypass timestamp, identity, pose-stream,
ontology, access, evidence, or API governance because their architecture is
more expressive.

#### 6.5 Ambiguity and Confidence Rules

Confidence MUST refer to an explicitly defined quantity: pose visibility,
identity association trust, feature quality, rule certainty, anomaly score
calibration, or model posterior. Scores from different stages MUST NOT be
collapsed into one display number without a documented combination method and
validation.

Ambiguous intervals MUST remain ambiguous. The system MAY display candidate
evidence for review, but MUST suppress authoritative behavioral or anomaly
assertions when required evidence is conflicting or incomplete.

### 7. Observability and Scientific Rigor Constitution

#### 7.1 Telemetry Truth Rules

Telemetry MUST be probe-backed or derived from immutable, deduplicated runtime
events. Every metric MUST declare source, aggregation interval, dimensions,
units, freshness, missing-state behavior, and retention. Critical dimensions
include runtime mode, endpoint profile, model/version, job/session/camera,
queue, worker, stage, outcome, and evidence run ID where applicable.

Required telemetry domains include:

- Triton liveness/readiness/model readiness and request failures;
- model latency, queue duration, batching, timeout, GPU memory/utilization and
  OOM/resource events;
- Celery enqueue/pickup/ack/retry/dead-letter/starvation observations;
- frame/drop/gap/late/reordered/persistence-failure counts;
- identity switches, fragmentation, unresolved associations and recovery;
- pose success/missingness/jitter/visibility/partial failure;
- behavior feature eligibility/suppression and anomaly output lineage;
- API/WS schema validation failures and artifact availability.

Synthetic telemetry acceptance, hardcoded availability, fake readiness states,
default-success branches, silently masked KPI failures, and dashboard
replacement of unavailable values with zeros are constitutional violations.

#### 7.2 Benchmark Integrity Laws

A benchmark run MUST be immutable and manifest-driven. At minimum it stores:

| Manifest domain | Required content |
| --- | --- |
| Code/runtime | commit hash, dirty-state disclosure, OS, service versions |
| Hardware | GPU model/driver/runtime, CPU/RAM where relevant |
| Workload | source media/live capture manifest, digest, duration, mode, concurrency |
| Inference | active endpoint, model/config/artifact versions, batching/scheduler parameters |
| Data contracts | schema, pose stream, feature, ontology and eligibility versions |
| Results | raw measurements, aggregations, failed/dropped samples, logs/artifacts |
| Comparison | explicit independent baseline and candidate identifiers |

Self-baseline comparison, candidate-versus-identical-candidate scoring, mock-only
production benchmarking, omitted failed attempts, synthetic pass records,
mixed-mode profile comparisons, changed input without disclosure, and reported
availability not supported by raw measurements are forbidden.

#### 7.3 Statistical Rigor Requirements

Operational production acceptance MUST use repeated real executions with a
predeclared pass/fail rule. For each reported primary performance or quality
metric, reports MUST include sample count, central estimate, dispersion, and
confidence interval or a documented reason a CI is inappropriate.

Research and paper claims comparing methods or configurations MUST additionally
include:

- hypothesis or estimand and primary/secondary metrics declared before result
  interpretation;
- baseline and candidate run grouping with comparable workloads;
- effect size and uncertainty;
- suitable parametric or nonparametric test with assumptions noted;
- multiple-comparison handling when evaluating many alternatives;
- power or sample-size limitation statement;
- treatment of failures, missing intervals, excluded sequences, and outliers.

Variance MUST be reported, not hidden behind averaged throughput or latency.
Quality metrics such as identity switches, pose jitter, feature eligibility,
coverage and event precision/recall MUST accompany latency claims whenever
scientific behavior readiness is asserted.

#### 7.4 Evidence Authority and Reproducibility

Evidence precedence is:

1. immutable raw observations/runtime events and source manifests;
2. versioned durable database records;
3. generated artifacts with manifest/digest lineage;
4. aggregated telemetry computed from authoritative records;
5. frontend presentation derived from versioned API contracts.

A cache, dashboard rendering, log excerpt without provenance, or hand-written
summary MUST NOT override a lower-numbered authority. Evidence artifacts MUST
be addressable, permission-controlled, integrity-checkable, and linked from
the claim or acceptance gate they support.

### 8. Queue, Orchestration, and Resilience Constitution

#### 8.1 Queue Governance and Worker Isolation

Celery orchestration MUST isolate live and offline work by named queue and
stage, bind workers deliberately, and expose bindings in diagnostics. Queues
MUST declare admission, concurrency, acknowledgment, prefetch, timeout,
retry, priority, overflow, dead-letter, and drain/cutover behavior.

`worker_prefetch_multiplier=1` and late acknowledgment are the required
starting policy for long-lived inference tasks unless an evidence-backed
amendment proves another configuration preserves correctness under worker loss
and overload. A worker MUST NOT consume queues outside its approved runtime
mode activation.

#### 8.2 Retry, DLQ, and Poison Message Laws

Retries MUST be bounded by fault class. Every retryable task MUST define a
maximum attempt count, bounded backoff with maximum delay, task deadline or
staleness rule, idempotency key, and terminal behavior. Unbounded retry loops
are prohibited.

| Failure class | Retry handling |
| --- | --- |
| Transient network/service timeout with healthy authority | Bounded retry while within deadline |
| Stale live frame after latency budget | Do not retry inference; account as drop/stale |
| Invalid schema, corrupt payload, unauthorized request | No retry; quarantine/DLQ and alert |
| Triton/model authority unavailable in production | Fail/degrade explicitly; no local fallback |
| Persistence conflict with idempotency recovery path | Bounded retry, deduplicate on commit |

Dead-letter records MUST include original message identity, mode/queue, retry
history, failure class, payload reference or sanitized digest, time envelope,
operator resolution state, and linked evidence. A poison message MUST NOT
cycle between retry queues or block healthy work indefinitely.

#### 8.3 Backpressure, Starvation, and Collapse Prevention

Each mode MUST define numeric SLO gates for queue depth, p95 queue wait, timeout
rate, and drop rate before acceptance. Backpressure actions MUST be explicit
and observable:

- live mode MAY shed stale frames, reduce governed cadence, suspend optional
  feature stages, or enter degraded presentation while preserving drop/gap
  truth and forbidding unsupported behavior claims;
- offline mode MAY pause admissions, throttle batch dispatch, or fail a job
  with resumable evidence; it MUST NOT discard frames silently for throughput;
- priority MUST not permit indefinite starvation; age and starvation metrics
  MUST trigger bounded escalation or admission control;
- queue collapse MUST not merge live/offline routes, start unauthorized
  endpoints, or disable evidence writes.

#### 8.4 RTSP State Machine Doctrine

Each live stream MUST have an auditable state machine:

```text
configured -> connecting -> connected -> degraded/disconnected
degraded/disconnected -> retry_wait -> reconnecting -> recovered/failed
connected/recovered/degraded -> stopping -> stopped
```

State transitions MUST emit source/session/camera scope, timestamps, attempt
number, reason, next action, and resulting temporal gap. Reconnect attempts
MUST have ceilings, bounded exponential backoff with jitter where applicable,
operator-stop cancellation, and terminal failure behavior. Reconnection MUST
not fabricate uninterrupted source time or silently reuse a prior identity
beyond validated recovery rules.

#### 8.5 Fail-Stop and Ingestion Resilience Semantics

Ingestion MUST reject corrupt/unsupported media or record per-frame decode
failures with reasons. Failure to persist essential event, time, or identity
records after inference MUST mark affected outputs non-authoritative and
surface an operator fault. A mode MUST fail stop when integrity or production
inference authority is lost; it MAY continue non-authoritative visualization
only when clearly labeled and segregated from accepted evidence.

### 9. API, Contract, and Schema Constitution

#### 9.1 Contract Authority and Versioning

REST, WebSocket, runtime event, artifact manifest, sequence export, telemetry,
and forensic trace payloads MUST be governed by a canonical contract registry.
Each public payload MUST include or be negotiated against a version, and its
producer, consumers, compatibility promise, validation tests, and migration
path MUST be recorded.

No public payload MAY change silently. Adding, removing, renaming, changing
meaning, changing nullability, changing units, or changing enumeration meaning
requires a schema change decision and appropriate version bump.

#### 9.2 Serializer Governance

Public serializers MUST explicitly enumerate exposed fields and access
permissions. `fields = '__all__'`, equivalent broad automatic exposure, and
reflection of sensitive model state into public endpoints are prohibited.
Temporal raw data, forensic traces, source identifiers, model internals, and
behavioral outputs require intentional authorization and auditability.

Serialization MUST preserve distinctions between missing, unavailable,
degraded, invalid, redacted, and measured-zero values. A client MUST not need
to infer these states from absence or string conventions.

#### 9.3 REST and WebSocket Governance

REST endpoints MUST expose deterministic idempotency, authorization,
pagination/filter semantics where needed, and versioned error contracts.
WebSocket messages MUST specify message type, version, sequence/order or replay
behavior, reconnection/catch-up semantics, duplicate handling, authorization,
and subscription lifecycle. WebSocket presentation events MUST NOT become an
untracked parallel truth source independent of durable evidence.

Generated frontend types or validation clients MUST be produced from the
canonical registry or tested against it. Handwritten client DTOs MAY exist
only when contract parity is verified in CI.

#### 9.4 Artifact Authority Hierarchy and Compatibility Rules

Artifacts are derived evidence unless explicitly declared raw acquisition.
Public artifact endpoints MUST state artifact type, version, producing run,
input references, content digest, creation time, quality/degradation flags,
and retention status.

Compatibility guarantees:

- additive optional fields MAY occur in a compatible minor contract update
  only when consumers are verified tolerant;
- changed semantics, units, required fields, identifiers, or privacy boundary
  require a breaking version or negotiated migration;
- database migrations and backfills MUST preserve provenance and be
  reversible or explicitly non-reversible with backup/evidence plan;
- unknown future versions MUST be rejected or safely reported as unsupported,
  not parsed as the current contract.

Silent contract drift, unversioned payload evolution, undocumented mutation,
and frontend-only repair of incorrect backend contracts are forbidden.

### 10. Data and Storage Constitution

#### 10.1 Typed Temporal Sequence Doctrine

Canonical sequence storage MUST represent observations as typed,
identity-scoped, source-time-ordered records rather than opaque result blobs.
At minimum, a sequence schema MUST carry source/session/camera identity,
canonical/local track provenance, temporal envelope, pose stream references,
visibility/missing masks, lifecycle events, feature/ontology versions,
runtime/model authority, quality flags, and event/artifact linkage.

Raw observations, derived transforms, behavioral features, anomaly candidates,
operator adjudications, and UI render artifacts MUST remain distinguishable in
storage and exports.

PostgreSQL is the canonical relational store for governed runtime state,
typed temporal sequence state, event ledgers, forensic lineage, benchmark
records, acceptance manifests and schema authority. SQLite-backed execution is
constitutionally forbidden for production maturity work because it does not
exercise the deployed transaction semantics, locking behavior, concurrency
behavior, indexing strategy, JSON/query behavior, migration characteristics or
operational failure modes that govern this platform in production.

#### 10.2 Persistence, Event Identity, and Idempotency

Durable persistence is authoritative for accepted operational and scientific
outputs. Every durable event/observation MUST have a stable idempotency key
derived from its scope and semantic identity, not from a transient retry
attempt. Replayed tasks MUST update or reject duplicates according to contract;
they MUST NOT create duplicate evidence or inflate telemetry.

Events MUST carry correlation identifiers spanning ingest, queue task, Triton
request, track/sequence, behavior output, artifact, API event and benchmark run
where applicable. Database and artifact writes MUST define transaction and
partial-write recovery behavior.

Every migration, schema constraint, query path, idempotency guarantee and
acceptance test that touches durable state MUST be reviewed and validated
against PostgreSQL semantics. A test, script or documented workflow that
silently swaps to SQLite, creates an ad hoc SQLite database, or presents
SQLite as an accepted execution path is invalid. Repository examples, CI
settings, developer bootstrap scripts and agent instructions MUST present
PostgreSQL as the single authoritative relational backend for this system.

#### 10.3 Artifact Lifecycle, Retention, and Compaction

Artifacts MUST have lifecycle states such as pending, complete, degraded,
invalidated, superseded, archived and deleted, with reasons and manifest
history. Artifact cleanup MUST never delete authoritative evidence needed by an
active acceptance, investigation, model dataset, or paper claim without a
recorded retention decision.

Raw temporal sequence retention currently follows the feature decision to keep
raw sequence records indefinitely unless manually purged. A future retention
change is a governance and data-migration decision and MUST specify legal,
privacy, storage, reproducibility and dataset consequences.

Compaction MAY materialize summaries or archive raw partitions only when it
preserves digests, provenance, schema readability, retrieval authority and
replay requirements. Destructive compaction without verified lineage and
retention authority is prohibited.

#### 10.4 Export and ML-Readiness Governance

Dataset or evidence exports MUST include source manifest/digests, eligibility
filters, excluded/missing intervals, identity/pose quality metrics,
schema/feature/ontology versions, model/runtime configuration where derived,
split assignment policy, label provenance, access policy, and generation code
revision.

No dataset is ready for ST-GCN/CTR-GCN, temporal transformers, contrastive
learning, graph reasoning, teacher-student distillation, anomaly training or
VLM fusion until identity and temporal quality gates pass and exclusions are
recorded. Data volume cannot substitute for valid lineage.

### 11. Security, Stability, and Failure Constitution

#### 11.1 Runtime Safety and Corruption Prevention

Video, identity, behavioral, and forensic data are sensitive. Authentication,
authorization, audit logs, least privilege, transport security, secret
management, input/path validation, and log redaction MUST cover ingest,
artifacts, raw sequence access, APIs, WebSockets, operations scripts and
exports. Production must not require privileged escalation to operate.

At minimum, the system MUST prevent:

- unauthorized access to raw video, raw sequences, forensic traces or
  behavioral outputs;
- command/path injection through media, artifact or deployment parameters;
- secrets, raw identity data or sensitive source URLs in unsafe logs;
- artifact tampering or mutation without digest/audit evidence;
- public schema exposure of unapproved private model fields;
- acceptance of data whose integrity lineage cannot be established.

#### 11.2 Failure Isolation and Degradation Policies

Partial degradation is allowed only when explicitly represented and scoped.
A presentation overlay may continue with marked stale/display-only points; a
behavioral event may not continue from the same data unless its input authority
still passes. A valid pose for one identity may persist when a sibling crop
fails; a corrupted response mapping cannot be salvaged by guessing.

Safe fallback semantics mean reduced capability or fail-stop with truth, not an
unrecorded alternative inference provider. Development fallback paths MUST be
labeled non-production and excluded from production evidence.

#### 11.3 Invalid and Catastrophic Runtime States

| State | Classification | Required response |
| --- | --- | --- |
| Both Triton mode profiles active in production | Invalid authority state | Stop admission, isolate, invalidate mixed evidence |
| Production inference completed by local/mock runtime | Catastrophic evidence violation | Quarantine outputs and block release/claim |
| Identity collisions or unsafe merges affecting features | Scientific integrity failure | Invalidate affected windows/datasets |
| Temporal corruption or unexplained ordering drift | Scientific integrity failure | Stop feature/anomaly acceptance for intervals |
| Telemetry reports ready while probes are unavailable | Observability integrity failure | Correct state, audit affected acceptance |
| Queue runaway/retry storm risking loss or starvation | Operational critical failure | Apply admission control/fail-stop policy |
| Schema exposure or unauthorized raw trace access | Security incident | Contain, audit, and follow incident recovery |

#### 11.4 Recovery Rules

Recovery MUST be bounded, observable and evidence-preserving. Operational
recovery procedures MUST specify detection, isolation, stop/admission action,
data integrity assessment, restart/cutover validation, evidence invalidation or
reprocessing, and sign-off. Recovery from a fatal authority or integrity fault
requires fresh validation; process uptime alone cannot restore maturity state.

### 12. Acceptance, Validation, and Evidence Constitution

#### 12.1 Maturity Closure Authority

Maturity is closed only by evidence, not by implementation intent, issue status,
UI availability, or mock tests. Every maturity claim MUST identify an owner,
required gate, execution environment, evidence artifacts, result, remaining
limitations and approval record.

PRs implementing constitutional concerns MUST include:

- mapped constitutional laws and requirements;
- contract/schema/ontology/migration changes where applicable;
- tests for nominal, failure, overload and integrity cases;
- real-production-route validation plan for runtime/ML claims;
- observable acceptance metrics and evidence paths;
- rollback/invalidation strategy;
- explicit deferred gaps that are not claimed mature.

#### 12.2 Production Readiness Gates

No production sign-off for affected capability is valid without passing the
applicable gates:

| Gate | Minimum evidence |
| --- | --- |
| Runtime mode | Native Linux startup preflight; active endpoint healthy; inactive endpoint unreachable |
| Triton/model | Required model readiness, contract signature, model/config/artifact manifest, GPU execution |
| Queue/resilience | Queue binding, bounded retries/DLQ/backpressure, failure and reconnect validation |
| Temporal | Timestamp envelope, gap/drift/replay validation using representative inputs |
| Identity | Multi-camera isolation, crossing/occlusion/ReID evaluation and trust metrics |
| Pose | Raw/smoothed/display lineage, batch partial-failure tests, pose quality measures |
| Behavior | Versioned ontology/features, missingness/ambiguity guards, interpretable traces |
| Observability | Probe-backed metrics, unknown/unavailable truth states, dedup tests |
| Contract/storage | Versioned schemas, explicit serializers, migrations/idempotency/retention checks |
| Benchmark | Real baseline/candidate repeats, confidence/variance and workload comparability |
| Security | Access/audit/redaction/secret/input/artifact integrity checks |

#### 12.3 GPU, Live, and Offline Validation Obligations

Production inference acceptance MUST execute on the intended NVIDIA GPU through
the active native-Linux Triton endpoint using declared model artifacts and
configuration. CPU runs, mocks, local fallback, Docker-only tests and
development demonstrations can support engineering but cannot close production
GPU validation.

Live validation MUST cover RTSP state transitions, reconnect boundaries,
bounded latency/backpressure behavior, source-time gaps, identity recovery,
display versus behavioral degradation, telemetry truth and active live-profile
routing. Offline validation MUST cover deterministic decoding/time base,
batching, full artifact persistence, replay, sequence/export integrity,
throughput and active offline-profile routing. A capability spanning both modes
requires evidence from both modes in separate valid activation windows.

#### 12.4 Benchmark and Scientific Acceptance

A production performance acceptance report MUST include repeated real runs,
explicit baseline and candidate, comparable workloads, raw result artifacts,
mode-specific thresholds, failed-run handling, sample count, variance and
confidence intervals. Scientific acceptance for behavior or anomaly claims
MUST additionally include quality metrics, eligible-window criteria, label or
reference truth protocol, effect size/statistical method and study limitations.

Mock-only maturity claims, synthetic telemetry acceptance, fake benchmark pass
logic, screenshots without raw evidence, self-baseline comparisons and silent
KPI masking are prohibited.

### 13. Cross-Wave Dependency Constitution

#### 13.1 Dependency Hierarchy

Implementation waves MUST honor dependencies in this order:

| Wave | Authority established | Blocks |
| --- | --- | --- |
| Wave 1: Runtime authority | Production route, endpoint mode, Linux/GPU/Triton validation | All production evidence |
| Wave 2: Queue and ingestion truth | Routing, backpressure, retry/DLQ, RTSP/time envelope | Continuity and live validity |
| Wave 3: Identity and temporal continuity | Canonical identity, lifecycle, association, sequence truth | Behavioral features |
| Wave 4: Pose behavioral substrate | Stream lineage, quality, batch/failure semantics | Scientific pose/feature quality |
| Wave 5: Behavioral feature layer | Ontology, typed windows, interpretable features/primitives | Learned anomaly maturity |
| Wave 6: Observability and benchmark truth | Evidence, metrics, repeated benchmark/statistics | Acceptance and paper claims |
| Wave 7: Contract and forensic governance | Stable external schemas and trace resolution | Auditable investigation |
| Wave 8: Final evidence closure | Full validation and recorded limitations | Production maturity claim |

No wave MAY be marked mature merely because code exists. A blocking predecessor
requires accepted evidence or an explicit decision that dependent maturity is
not claimed.

#### 13.2 Why Wave 3 Blocks Wave 5

Wave 3 establishes who an observation represents and whether ordered
observations remain temporally coherent. Wave 5 computes duration, repetition,
transition and interaction features attached to a subject. Without Wave 3,
feature windows may span ID switches, cross-camera collisions, unsafe
interpolation or unresolved re-entry. The resulting feature is not noisy
behavior; it is semantically assigned to the wrong entity. Therefore behavior
feature authority, anomaly primitive validity and training-export eligibility
MUST remain blocked until identity and temporal continuity pass.

#### 13.3 Why Wave 4 Blocks Wave 6

Wave 4 establishes pose-stream definitions, RTMPose contract validity,
visibility/missingness, smoothing/interpolation lineage and quality. Wave 6
reports benchmark and observability evidence about behavior readiness and
scientific quality. Without Wave 4, a benchmark can measure fast response while
the input substrate is jittery, missing, presentation-filtered, wrongly
attributed or batch-corrupted. Operational latency alone cannot validate a
scientific pipeline. Wave 6 may report runtime performance earlier, but MUST
not report pose/behavior maturity or comparative scientific conclusions until
Wave 4 evidence exists.

#### 13.4 Why Observability Blocks Scientific Maturity

Scientific maturity requires the ability to detect missing measurements,
failed stages, data exclusions, quality drift, runtime changes and repeated-run
variance. If telemetry fabricates readiness, suppresses failure, loses event
deduplication or treats unavailable data as zero, the platform cannot determine
which samples are valid. Any behavior/anomaly claim then lacks a trustworthy
denominator and execution context. Probe-backed observability and evidence
lineage are therefore preconditions to scientific acceptance.

#### 13.5 Why Contract Governance Blocks Forensic Maturity

Forensic review must traverse an event through identity, temporal window, pose,
feature, model/runtime, artifact and UI/API presentation without ambiguity.
Unversioned payloads, broad serializers or silent WebSocket mutation break the
chain: historical evidence may no longer mean what a reviewer sees. A forensic
surface without schema authority can be convenient but cannot be authoritative.
Contract governance is therefore a prerequisite for forensic maturity and
paper traceability.

### 14. Anti-Regression Governance Constitution

This section converts historical production-closure failures into permanent
blocking rules. A maturity claim, release approval, benchmark statement, paper
result, or production-readiness declaration is invalid when any applicable rule
below lacks measured, reproducible, durable evidence.

#### 14.1 Runtime Truth Governance

Runtime truth is the measured state of the executing system, not the intended
configuration. Every runtime claim MUST be backed by recorded process, port,
endpoint, queue, model, GPU, dependency, environment and telemetry evidence
captured during the claimed run. A health endpoint, acceptance script or CI job
MUST NOT infer readiness from configuration files, artifact presence or boolean
flags alone.

Runtime validation MUST include these layers when applicable:

| Layer | Required proof | Blocks when missing |
| --- | --- | --- |
| Process | owning user, command, binary path, PID, start time | deployment readiness |
| Network | bound active ports, unreachable inactive ports, source reachability | endpoint readiness |
| Triton | live/ready probes, model `READY`, config digest, canary inference | inference readiness |
| GPU | device identity, CUDA/TensorRT versions, utilization, memory, errors | GPU maturity |
| Queue | route binding, consumer identity, backlog, retry/DLQ state | orchestration maturity |
| Persistence | PostgreSQL transaction and state reconciliation | evidence maturity |
| Telemetry | fresh probe-backed metrics and unknown/degraded states | observability maturity |

Constitutional blocker: if runtime truth disagrees with declared configuration,
runtime truth wins and the claim is blocked until reconciled.

#### 14.2 Production Evidence Integrity

Production evidence MUST be durable, immutable after publication, reproducible
from source manifests, and attributable to a real execution environment.
Evidence packages MUST include artifact digests, code revision, environment
fingerprint, runtime profile fingerprint, dependency fingerprint, GPU/runtime
fingerprint, dataset provenance, telemetry provenance, and command lineage.

Evidence MUST explicitly distinguish:

- mock, simulated, synthetic, replayed, development, staging and production
  sources;
- CPU, GPU, Triton, local fallback and unsupported execution paths;
- live stream, offline processing and generated test fixtures;
- canonical execution, degraded execution, fallback execution and failed
  execution;
- temporary pytest paths, durable evidence directories and archived snapshots.

Placeholder-only directories, empty manifests, copied screenshots without raw
evidence, manifests pointing only to temporary pytest paths, mutable acceptance
artifacts, and production claims based on dev-only runs are prohibited.

#### 14.3 Benchmark Scientific Rigor

Benchmark acceptance MUST measure causality between a defined baseline and a
defined candidate. A benchmark report MUST include workload manifest, source
media or dataset digest, baseline/candidate code revisions, runtime profiles,
model/config digests, warmup policy, sample count, failed-run handling,
variance, confidence interval, effect size where relevant, latency/throughput
distribution, resource counters, and raw output artifacts.

Mandatory statistical requirements:

- at least two independent real runs for operational smoke comparisons and at
  least three for maturity closure unless a stricter plan threshold exists;
- confidence intervals or an explicit non-parametric interval method for
  repeated performance claims;
- no self-baselining, no candidate-only pass state, no synthetic-only pass
  state, and no exclusion of failed runs without recorded rationale;
- separate reporting for live, offline, CPU, GPU, synthetic, production and
  degraded windows.

Benchmark artifact presence is never sufficient. Acceptance scripts MUST
validate run causality, raw metrics, variance, thresholds, failure accounting
and runtime fingerprints.

#### 14.4 CI/CD Authority Model

CI is authoritative only when it verifies runtime-relevant truth. A green CI
status MUST NOT override broken production behavior, missing evidence,
unreconciled runtime state, hidden xfails, failed critical suites, placeholder
artifacts, SQLite-backed evidence, or degraded production probes.

Required CI layers:

| Layer | Minimum enforcement |
| --- | --- |
| Static/schema | explicit contracts, migrations, serializers, artifact schemas |
| Unit/contract | deterministic behavior and API compatibility |
| Integration/system | PostgreSQL, queues, telemetry and runtime reconciliation |
| Runtime gate | native Linux/Triton/GPU evidence for production claims |
| Evidence gate | artifact digests, non-placeholder content, lineage, environment |
| Benchmark gate | baseline/candidate causality and statistical report validation |
| XFail gate | no hidden `xfail`; all strict xfails require formal deferral |

Critical suite failures, missing required evidence, or runtime reconciliation
gaps are acceptance vetoes even when non-critical CI jobs pass.

#### 14.5 Dev/Prod Parity Governance

Development evidence can support engineering progress but cannot close
production readiness unless it matches the required production authorities or
is explicitly scoped as non-production. Any dev/prod difference in OS,
hardware, CUDA/TensorRT/Triton version, model artifact, environment file,
queue route, database, port, worker script, dependency lock, timeout, feature
flag or fallback path MUST be recorded and assessed.

Production runtime authority is `backend/.env`; repo-root `.env` MUST NOT
participate in backend startup. PostgreSQL is the only relational authority.
SQLite fallback in evidence paths, benchmark paths, migrations, acceptance
scripts or runtime claims is constitutionally invalid.

#### 14.6 Runtime Reconciliation Guarantees

The platform MUST reconcile runtime state against persistent state for every
long-running or asynchronous workflow. A task, DB row, artifact, queue message,
telemetry event and frontend state MUST converge to one accountable lifecycle
or expose a reconciliation fault.

Required reconciliation checks include:

- Celery task terminal state versus database processing/completed/failed state;
- retry attempt lineage versus persisted idempotency key;
- artifact manifest state versus file existence, digest and schema validity;
- model-serving health versus actual Triton/model/GPU canary state;
- frontend status versus backend and persistence truth;
- runtime mode and queue route versus production environment authority.

`Celery FAILURE` while the database remains `processing`, stale frontend
success while backend failed, and acceptance scripts validating booleans
without real runtime proof are blockers.

#### 14.7 GPU Utilization & Inference Efficiency Governance

GPU-backed production claims MUST include attribution for utilization,
latency, queue wait, batching, model execution and CPU/GPU transfer time.
Underutilization is not a maturity failure by itself, but unexplained
underutilization during throughput claims is a benchmark and deployment
blocker.

Triton batching efficiency validation MUST record batch-size distribution,
dynamic-batch delay, model instance count, GPU memory, compute utilization,
request concurrency, rejected/timeout requests and per-stage latency. TensorRT
engine rebuilds MUST be tied to CUDA/TensorRT fingerprints and Triton
`READY` evidence before acceptance.

#### 14.8 Queue/Retry/Backpressure Truth Governance

Queue behavior MUST be bounded, observable and lineage-preserving. Every
retry MUST emit attempt number, parent task, correlation ID, reason, delay,
queue name, worker identity and terminal outcome. Backpressure MUST expose
admission decisions, dropped/deferred work, queue age, retry budget burn,
timeout budget burn and DLQ counts.

Queue collapse resistance and retry amplification tests are mandatory for
streaming, offline batch, inference and artifact workflows. Silent retry
loops, unbounded retries, queue mixing between live/offline modes, and
lineage-free replays are prohibited.

#### 14.9 Identity & Temporal Integrity Governance

Canonical identity continuity and timestamp truth are non-negotiable for
behavioral claims. Multi-camera identity isolation, crowded-scene temporal
continuity, occlusion/re-entry handling, source-time drift, frame drops,
interpolation boundaries and identity switch metrics MUST be measured before
behavior, anomaly, training-export or forensic claims are accepted.

Unscoped identity keys, local track IDs presented as canonical identity,
unattributed dropped frames/events, silent interpolation across invalid gaps,
and temporal windows lacking canonical start/end time are blockers.

#### 14.10 Observability Truth Enforcement

Operational dashboards are mandatory for production capabilities and MUST show
runtime mode, Triton/model readiness, GPU utilization, queue depth/age,
latency percentiles, retry/DLQ counts, frame/event drop attribution,
degradation states, evidence generation state, benchmark lineage and
frontend/backend consistency.

SLO, latency, queue, retry, timeout and degradation budgets MUST be declared
per feature or release. A budget breach MUST produce telemetry, evidence and
an operational decision: accept as degraded with scope, throttle, fail closed,
rollback or freeze production claims.

#### 14.11 Feature Closure Requirements

A feature wave closes only when code, contracts, tests, runtime validation,
evidence, observability, rollback plan, deferred-risk record and owner sign-off
are complete. Issue/task completion, merged code, passing unit tests or
artifact creation cannot close a feature when runtime truth remains broken.

Live streaming validation MUST cover RTSP connect/disconnect/reconnect, frame
timeout behavior, latency/backpressure, dropped-frame attribution, source-time
gaps and recovery. Offline validation MUST cover deterministic decoding,
replay, batching, artifact persistence and PostgreSQL-backed reconciliation.
Long-duration soak tests MUST cover leak/drift/backlog behavior for the
declared operating window.

#### 14.12 Technical Debt Budget Governance

Technical debt MAY be accepted only as a recorded budget item with owner,
scope, affected constitutional rule, risk, expiry condition and blocking
threshold. Unbounded debt accumulation, repeated deferrals without runtime
evidence, or debt that masks production truth is prohibited.

Debt becomes a release blocker when it affects production inference authority,
evidence integrity, benchmark validity, identity/temporal truth, PostgreSQL
semantics, CI gates, observability, rollback safety or security boundaries.

#### 14.13 XFail Governance Policy

`xfail` is allowed only for explicit scaffold or deferred behavior with a
linked requirement, owner, reason, expiry/exit condition and strict marker
where supported. Hidden xfails, broad xfail patterns, xfails in critical
production-readiness gates, and xfails that allow maturity closure are
prohibited.

Every release gate MUST report xfailed tests separately from passed tests.
When behavior is implemented, the same change set MUST remove the xfail and
convert the assertion to a normal passing test.

#### 14.14 Runtime Drift Detection

Runtime drift exists when the observed runtime differs from the governed
profile. Drift detection MUST cover environment variables, `.env` authority,
shell path resolution, binary paths, model repository digests, TensorRT engine
digests, Python dependency lock, CUDA/TensorRT versions, queue definitions,
port bindings, worker commands, database host/role and telemetry exporters.

Detected drift blocks production-readiness claims until either corrected or
formally accepted as a new governed profile with evidence. Environment drift
tolerance, untracked runtime overrides and undocumented manual fixes are
prohibited.

#### 14.15 Artifact Authenticity Validation

Acceptance artifacts MUST be authentic, schema-valid, digest-addressed and
traceable to their producing command. Validation MUST inspect content, not
only paths. Minimum checks include non-empty payload, schema version, digest,
source inputs, runtime fingerprint, code revision, timestamps, producer
command, environment, and cross-links to raw metrics or logs.

Mutable acceptance evidence MUST be superseded by a new immutable snapshot
rather than edited in place after publication. Any corrected artifact MUST
record the superseded digest and reason.

#### 14.16 Deployment Safety Rules

Deployment MUST be deterministic, reversible and traceable. A production
deployment requires branch/hash parity, dependency fingerprint, environment
fingerprint, database migration status, active runtime profile, inactive
profile isolation, worker command inventory, health/canary proof, evidence
writeability and rollback instructions.

Unsupported fallback execution paths, hidden local inference, production
Docker/sudo assumptions, unresolved shell export drift, and backend startup
from repo-root `.env` are deployment blockers.

#### 14.17 Production Rollback Governance

Rollback is mandatory when a deployment causes fatal runtime authority loss,
data corruption risk, unauthorized exposure, unreconciled queue collapse,
model-serving failure, repeated SLO breach beyond the declared degradation
budget, or evidence-integrity failure. Rollback MUST preserve evidence,
record trigger and scope, stop admission where needed, verify restored runtime
truth, and mark affected artifacts invalid or requiring reprocessing.

Emergency remediation MAY prioritize containment over full analysis, but MUST
produce a post-remediation record with root cause, impacted evidence,
preventive control and reopened validation gates.

#### 14.18 Forensic Traceability Requirements

Every accepted runtime, behavioral, benchmark or production-readiness claim
MUST be traceable across evidence lineage, model lineage, runtime lineage,
deployment lineage, benchmark lineage, artifact digests, dataset provenance
and telemetry provenance. A reviewer MUST be able to reconstruct the path from
source media or live-capture manifest to frames, detections, pose, identity,
features, anomaly candidates, persistence rows, API/WS payloads, frontend
state and evidence artifacts.

Weak operational traceability is a blocker for forensic maturity and paper
claims.

#### 14.19 AI/Behavioral Intelligence Scientific Integrity

Behavioral intelligence and future AI layers MUST distinguish observation,
heuristic, learned inference, anomaly candidate and adjudicated outcome.
Anomaly pipeline causality MUST identify input windows, feature versions,
model versions, thresholds, calibration data, uncertainty, exclusions and
operator/adjudication status.

Scientific validity requires dataset provenance, label provenance,
train/validation/test split policy, leakage controls, quality filters,
baseline comparison, statistical method, limitations and reproducible rerun
instructions. Explainability readiness requires each candidate event to expose
the features, temporal windows and model/run lineage supporting it.

#### 14.20 Future Expansion Safety Rules

Graph intelligence, transformers, contrastive learning, multimodal fusion,
distributed inference, adaptive orchestration, VLM integration and future
cognitive AI layers inherit every runtime, identity, temporal, evidence,
benchmark, queue, observability, deployment and scientific rule in this
constitution. New model sophistication MUST NOT bypass canonical identity,
timestamp truth, Triton production authority, telemetry truthfulness,
benchmark reproducibility or explainability readiness.

Distributed runtime scaling MUST add topology, clock, queue, shard, model,
GPU and evidence-lineage fingerprints before production claims. Multimodal
systems MUST record modality provenance, synchronization quality, missingness,
privacy boundaries and fusion-version lineage.

#### 14.21 Definition of Production-Ready

A capability is production-ready only when all applicable criteria are met:

| Criterion | Measurable acceptance |
| --- | --- |
| Runtime | active native Linux Triton/GPU route passes canary; inactive route unreachable |
| Persistence | PostgreSQL-backed state is reconciled with task/artifact/runtime state |
| Evidence | immutable non-placeholder snapshot with digests and lineage exists |
| CI/CD | critical suites and evidence gates pass with no hidden xfails |
| Observability | dashboards and alerts expose valid/degraded/unavailable states |
| Resilience | timeout, retry, backpressure, reconnect and rollback behavior verified |
| Security | secrets, raw media, identity and artifact access controls verified |
| Parity | dev/prod differences documented and non-blocking |
| Rollback | tested rollback or fail-closed recovery path exists |

Any critical runtime failure, unreconciled state, fake/placeholder evidence,
unsupported fallback path, SQLite evidence path, or failed critical suite
blocks production-ready status.

#### 14.22 Definition of Scientifically Valid

A claim is scientifically valid only when it is reproducible, statistically
supported, lineage-complete and limitation-aware. Minimum measurable criteria:

- versioned input dataset or live-capture manifest with digests and provenance;
- canonical timestamp and identity quality eligibility criteria;
- model, feature, ontology, runtime and dependency versions;
- baseline/candidate comparison when making improvement claims;
- repeated runs with variance, confidence interval and failure accounting;
- explicit mock/synthetic/production distinction;
- exclusion criteria, uncertainty, limitations and rerun instructions;
- raw metrics and artifact digests stored durably.

Scientific invalidity is a constitutional blocker for papers, thesis claims,
model maturity, anomaly effectiveness and comparative performance statements.

#### 14.23 No Silent Failure Doctrine

Every degraded path MUST emit telemetry. Every fallback path MUST emit
telemetry and be labeled non-canonical unless formally governed. Every retry
MUST emit lineage. Every runtime inconsistency MUST become observable. Every
dropped frame, event, queue item, artifact write, inference timeout and
identity/temporal invalidation MUST be attributable to a cause, scope and
decision.

Silent retry loops, fake health checks, synthetic readiness, hidden fallback
execution, KPI masking, unavailable-as-zero metrics and unobserved degraded
runtime states are prohibited.

#### 14.24 Mandatory Final Sign-Off Requirements

Final maturity sign-off requires named approval records for runtime,
observability, benchmarks, AI/scientific validity, deployment, CI/CD, forensic
traceability and operational survivability. Each approval MUST identify
evidence paths, run identifiers, remaining accepted risks, rollback criteria
and the constitutional gates reviewed.

An acceptance veto applies when any sign-off domain lacks evidence, contains a
critical failure, relies on mock/dev-only proof for production claims, hides an
xfail, bypasses PostgreSQL, uses unsupported fallback execution, or cannot
reproduce the claimed result.

#### 14.25 Anti-Regression Enforcement Matrix

| Failure class | Detection layer | Blocking layer | Evidence requirement | CI enforcement | Production enforcement | Rollback behavior | Escalation policy |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Fake maturity signaling | Spec/PR/evidence review | Feature closure | mapped gates and real run evidence | fail maturity gate | block sign-off | invalidate claim | owner + reviewer sign-off |
| Placeholder evidence | Artifact validation | Evidence integrity | non-empty schema-valid digested artifact | fail evidence job | reject manifest | supersede artifact | evidence owner |
| Synthetic benchmark claim | Benchmark validator | Scientific rigor | real baseline/candidate runs and variance | fail benchmark gate | exclude from SLO claim | rerun benchmark | runtime + research leads |
| Green CI, broken prod | Runtime reconciliation | CI/CD authority | prod health/canary/reconciliation snapshot | fail release gate | freeze readiness | rollback if deployed | release owner |
| SQLite fallback | Config/test audit | Data authority | PostgreSQL DSN and transaction proof | fail DB gate | stop affected workflow | invalidate outputs | DBA/runtime owner |
| Hidden xfail | Test metadata scan | XFail governance | strict xfail registry or removal | fail xfail gate | block closure | none unless deployed | test owner |
| Silent degradation | Telemetry audit | Observability truth | degraded/fallback metrics and alerts | fail observability gate | mark degraded/fail closed | rollback on budget breach | ops owner |
| Runtime drift | Fingerprint diff | Deployment safety | env/bin/model/queue/port digests | fail drift gate | stop admission | restore governed profile | deployment owner |
| Queue retry amplification | Queue telemetry | Queue truth | retry lineage, DLQ, backlog, budget burn | fail resilience gate | throttle/fail closed | drain/rollback | ops + backend owners |
| Celery/DB mismatch | Reconciliation job | Runtime reconciliation | task, DB and artifact lifecycle join | fail system gate | quarantine workflow | reprocess or rollback | backend owner |
| GPU underutilization claim | GPU/latency telemetry | GPU efficiency | utilization, batch, latency attribution | fail perf gate | block throughput claim | rerun tuned profile | AI infra owner |
| Identity/temporal corruption | Sequence quality gate | Scientific integrity | switch/gap/drift metrics and exclusions | fail integrity gate | suppress behavior claim | reprocess affected windows | AI validity owner |
| Artifact tampering | Digest/audit check | Authenticity | immutable snapshot and supersession record | fail artifact gate | reject artifact | restore prior snapshot | evidence owner |
| Deployment unsafe | Preflight/canary | Deployment safety | parity, migration, health, rollback proof | fail release gate | stop deployment | rollback immediately | release owner |
| Unbounded debt | Debt ledger review | Debt budget | owner, expiry, risk and blocking threshold | fail planning gate | freeze affected claim | schedule remediation | tech lead |
| Unsupported future AI path | Design review | Expansion safety | lineage, modality, runtime and validation plan | fail design gate | block activation | disable feature flag | architecture owner |

### 15. Final Architectural Positioning

#### 15.1 Current Maturity

The implemented system currently provides meaningful video analytics
infrastructure: ingestion, live/offline workflows, Triton-related runtime
routing and model calls, RTMPose-oriented outputs, tracking, queue
orchestration, telemetry surfaces, artifacts, event persistence and frontend
overlays. Repository evidence also shows partial queue isolation, execution
profile configuration, queue wait capture, ReID-related code, pose display
records and active work on production optimization.

It is not yet constitutionally mature behavioral intelligence. Existing
implementation and documents still contain partial or conflicting production
routing assumptions, incomplete single-active-endpoint enforcement, incomplete
identity/temporal proof, display-oriented pose surfaces, pending dynamic-batch
and benchmark sign-off work, and unclosed scientific evidence requirements.

#### 15.2 Transitional Architecture State

The transitional system is an evidence-constrained video and pose analytics
platform moving toward temporal behavioral intelligence. During transition:

- frame, pose and overlay functionality MAY be delivered as analytics
  capability;
- behavioral features MAY be implemented experimentally when explicitly
  labeled and gated;
- no anomaly, scientific, forensic or production maturity claim MAY bypass the
  runtime, temporal, identity, pose, telemetry and schema requirements above;
- current optimization milestones MUST close their evidence gates rather than
  treating intended architecture as implemented fact.

#### 15.3 Target Production Identity

The target platform is a native-Linux, Triton-only, single-active-mode,
GPU-backed behavioral intelligence system that deterministically transforms
governed live or offline inputs into identity-continuous temporal sequences,
versioned pose streams, interpretable behavioral features, traceable event
candidates, trustworthy metrics and reproducible evidence.

True temporal intelligence exists when an output can answer, with evidence:
what was observed, for whom, over which valid interval, using which source and
model authority, under which missingness and confidence constraints, according
to which behavioral definition, and with which reproducible validation.

#### 15.4 Future AI-Readiness Posture

Learned anomaly AI is scientifically valid only after:

- production Triton execution and mode authority are verified;
- source-time, sequence replay and drift laws are enforced;
- identity trust and exclusion thresholds pass representative validation;
- pose streams and quality semantics are versioned and measured;
- behavioral ontology and interpretable features are accepted;
- datasets carry lineage, filters, splits and quality manifests;
- telemetry and benchmarks expose real failures and uncertainty;
- APIs/artifacts preserve forensic traceability and access control.

Only then may temporal transformers, ST-GCN/CTR-GCN, contrastive methods,
interaction graphs, teacher-student learning or VLM fusion be evaluated as
behavioral intelligence candidates. They remain subject to this constitution;
model sophistication never waives truth, identity, evidence or operational
safety.

### 16. Heterogeneous Production Runtime Maturity Constitution

#### 16.1 Heterogeneous Runtime Governance

Development and production are intentionally heterogeneous and MUST NOT be
forced into false environment symmetry. Windows, Docker and RTX 3050 Ti local
development MAY validate code behavior, contracts, schemas, replay logic, queue
routing logic, env parsing and non-GPU causality exports. Native Linux,
no-Docker, no-sudo RTX 5090 production is the only authority for GPU execution,
inference truth, throughput claims, lifecycle acceptance and final evidence.

Rationale: Local hardware, OS, process model, CUDA/PyTorch stack and VRAM limits
cannot prove production behavior on the RTX 5090 server.

#### 16.2 Production Runtime Authority

Production authority MUST be isolated to a committed Git SHA deployed on the
production branch, `backend/.env` as the only backend runtime environment
authority, PostgreSQL relational state, Redis broker/cache behavior, Celery
orchestration, Triton inference and evidence under `ci_evidence/production/...`.
Repo-root `.env` MUST NOT participate in backend production startup.

Rationale: Hidden environment files, dirty runtime state and unreviewed
production-only changes detach evidence from the code and configuration that
produced it.

#### 16.3 PostgreSQL-Only Persistence

PostgreSQL is the only accepted relational backend for runtime, tests,
migrations, benchmarks, acceptance and evidence. SQLite fallback is forbidden.
Any persistence, ORM, transaction, migration, constraint or evidence-storage
change MUST be validated against PostgreSQL semantics. If tests fail because a
PostgreSQL role lacks `CREATEDB`, agents MUST surface the required DBA/bootstrap
action and MUST NOT switch to SQLite.

Rationale: SQLite success does not prove PostgreSQL locking, transaction,
constraint, JSON-field or concurrency semantics.

#### 16.4 Triton-Only Production Inference

Production MUST run with `INFERENCE_STRATEGY=triton_only`. Local Ultralytics,
PyTorch CUDA, OpenCV DNN GPU, ONNX Runtime, OpenVINO or direct application model
inference paths MUST fail closed in production. No fallback from Triton to local
CUDA or CPU is valid for production claims. Triton is the compatibility boundary
between application code and production GPU execution.

Rationale: RTX 5090 `sm_120` compatibility exposed that local CUDA/PyTorch
execution can be invalid while Triton remains the governed serving boundary.

#### 16.5 Single Active Triton Endpoint Policy

Live and offline Triton endpoint profiles MAY both be configured, but exactly
one profile MAY be production-active at a time. The active mode MUST be selected
by `TRITON_EXECUTION_MODE=live` or `TRITON_EXECUTION_MODE=offline` in
`backend/.env`. The inactive endpoint MUST be unreachable and MUST NOT receive
production inference traffic, scheduler requests, Celery routing or
production-ready health status.

Rationale: Dual configured profiles without single-active enforcement create
cross-talk, fake health and evidence that cannot be tied to one runtime mode.

#### 16.6 Production Preflight Before Work

Production MUST fail before processing any video when runtime contract checks
fail. Required preflight evidence includes native Linux/no-Docker/no-sudo
runtime shape, backend venv Python path, RTX 5090 GPU identity and compatibility,
TensorRT runtime and engine digest compatibility, pinned Triton binary, active
endpoint readiness, inactive endpoint unreachability, required model configs and
artifact hashes, `backend/.env` fingerprint, Celery route/worker/concurrency and
prefetch policy, PostgreSQL authority, Redis reachability and evidence output
paths.

Rationale: Expensive or authoritative work must not begin against stale engines,
wrong endpoints, wrong env files, ambiguous queues or invalid model artifacts.

#### 16.7 Replay and Evidence Integrity

`runtime_ingest_video` MUST use an explicit replay policy. `reuse-success` MAY
reuse only a previous completed successful job. `fail-on-existing` MUST reject
any existing replay key. `new-attempt` MUST create a fresh job linked to replay
lineage. Failed replay lineage MUST NEVER be accepted as production evidence.
Benchmark and acceptance runs MUST default to fresh validation or successful
replay only. Evidence MUST include job ID, replay key, replay policy, source
hash, Git SHA, env fingerprint, runtime profile, model artifact hashes, queue
topology, GPU trace and causality export.

Rationale: Reusing failed lineage as `"replayed": true` can convert an old
failure into apparent acceptance and hide current runtime drift.

#### 16.8 Branch and Sync Governance

Runtime stabilization MUST use the short-lived
`release/prod-runtime-stabilization` branch. A permanent production-only fork is
forbidden. All production fixes MUST merge back into the active development
branch after sign-off. Production deploy acceptance requires local HEAD equals
origin HEAD, production HEAD equals origin HEAD, the production tracked working
tree has no unexplained changes and production stash entries are reviewed,
archived and dropped only when understood.

Rationale: Production-only hotfixes and stale dirty trees make evidence
unreviewable and detach deployed behavior from source control.

#### 16.9 Validation Hierarchy

Local validation MAY gate code correctness, but MUST NOT certify production GPU
behavior. Production validation MUST include real upload lifecycle through
canonical API admission, Celery, Triton, PostgreSQL and evidence export; GPU
utilization and saturation checks; TensorRT/Triton readiness; live/offline
endpoint isolation; queue and worker telemetry; dynamic batching evidence; and
long-running runtime stability evidence. CI MAY certify production GPU behavior
only when running on a production-equivalent self-hosted GPU lane with durable
production evidence.

Rationale: The risk being validated determines the valid test environment; a
dev smoke test cannot certify production GPU acceptance.

#### 16.10 Operational Safety

Production startup MUST prefer fail-closed behavior over degraded inference when
authority contracts are violated. Developers MAY keep inconsistent local
CUDA/PyTorch stacks because production never trusts local inference paths. Weak
local GPUs are acceptable because local tests are not throughput gates. No-sudo
production MUST remain viable through user-space pinning of Python, Triton,
TensorRT artifacts, environment files and evidence scripts.

Rationale: Reliability comes from narrowing authority and failing closed, not
pretending every machine is equivalent.

#### 16.11 Evidence and Closure

Runtime maturity closure requires packaged evidence under
`ci_evidence/production/runtime_maturity/final/`, final hash parity, final
production stash/tree hygiene, successful preflight, successful fresh lifecycle
job, GPU telemetry for the accepted job, causality export for the accepted job,
an evidence manifest tied to Git SHA, env fingerprint, model hashes, queue
topology, replay metadata and runtime profile, and merge of
`release/prod-runtime-stabilization` back into the active development branch.

Rationale: Closure is reproducible only when evidence, branch state, runtime
state and accepted job lineage all identify the same governed execution.

#### 16.12 Heterogeneous Runtime Maturity Compliance Gates

| Principle | Required gate |
| --- | --- |
| Heterogeneous runtime governance | Local evidence labeled contract-only; production GPU claims tied to RTX 5090 evidence |
| Production runtime authority | Git SHA, `backend/.env` fingerprint, PostgreSQL, Redis, Celery, Triton and evidence-root manifest |
| PostgreSQL-only persistence | PostgreSQL DSN/test role and no SQLite fallback in runtime, tests, migration, benchmark or evidence paths |
| Triton-only inference | `INFERENCE_STRATEGY=triton_only`, local inference rejection tests and production preflight proof |
| Single active endpoint | Active endpoint ready, inactive endpoint unreachable and no inactive queue/scheduler traffic |
| Production preflight | OS, Python, GPU, TensorRT, Triton, model, env, queue, storage and evidence checks captured before work |
| Replay integrity | Explicit replay policy tests and rejection of failed replay lineage for acceptance |
| Branch and sync governance | Local/origin/production hash parity plus reviewed stash/tree hygiene |
| Validation hierarchy | Local contract tests separated from production lifecycle/GPU/queue/causality evidence |
| Operational safety | Fail-closed paths and no-sudo user-space pinning verified |
| Evidence and closure | Final package under `ci_evidence/production/runtime_maturity/final/` with accepted job manifest and merge record |

## Governance

This constitution is the binding engineering, runtime and scientific maturity
authority for the Production Behavioral Intelligence Platform. It supersedes
conflicting generic project governance and any documentation that permits
production conduct forbidden here. Implementation may lag a newly ratified
law, but such lag is an explicit gap and cannot be presented as accepted
maturity.

### Amendment Procedure

Every amendment MUST:

1. state the governing problem, affected laws and implementation/evidence
   consequences;
2. modify this document with a Sync Impact Report;
3. assign a semantic version bump and ISO-formatted amendment date;
4. propagate changed gates to Spec Kit templates and affected runtime guidance;
5. identify any implementation gaps, migration needs or invalidated evidence;
6. undergo review before its dependent capability is accepted.

Version policy is strict: MAJOR for incompatible authority or principle
redefinition; MINOR for additive principles or materially expanded mandatory
governance; PATCH for non-semantic clarification or corrected wording.

### Compliance and Review Authority

Specs, plans, tasks, pull requests, deployments, benchmark reports and
scientific outputs MUST map applicable constitutional rules to evidence or
explicitly state why a rule is not applicable. Review MUST block:

- production local inference fallback or mixed active endpoint profiles;
- temporal or identity integrity gaps used for behavioral claims;
- unversioned public contracts, broad serializer exposure or undocumented
  artifact mutation;
- unbounded retries, uncontrolled collapse behavior or hidden degradation;
- synthetic/readiness/benchmark evidence that is not derived from measured
  production-authority execution;
- maturity claims without reproducible evidence;
- CI success while production runtime reconciliation gaps remain;
- benchmark pass states without baseline/candidate causality;
- evidence artifacts that are placeholder-only, mutable, temporary-path-only,
  synthetic-only, dev-only for production claims, or SQLite-backed;
- hidden xfails, silent degraded runtime states, unsupported fallback paths,
  untracked runtime overrides, or environment drift tolerance;
- task completion claims while the real runtime, persistence or telemetry
  state remains broken.

Operational SLO numbers, model-quality thresholds, retention changes, access
policy changes, and behavioral ontology decisions MUST be recorded in the
feature plan and evidence artifacts when they are not fixed by this
constitution. Such values are engineering decisions subject to validation, not
license to weaken these laws.

**Version**: 2.2.0 | **Ratified**: 2026-02-27 | **Last Amended**: 2026-05-27
