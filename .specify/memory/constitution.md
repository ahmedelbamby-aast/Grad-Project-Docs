<!--
SYNC IMPACT REPORT
==================
Version change: 1.8.1 -> 2.0.0
Bump rationale: MAJOR - The prior project-wide quality and documentation
charter is replaced by binding runtime, temporal, identity, scientific,
contract, and acceptance laws for a production behavioral intelligence
platform. This redefines production inference authority, maturity closure,
and the meaning of acceptable evidence.

Modified principles:
- Generic student monitoring engineering directives -> Production behavioral
  intelligence system doctrine and runtime authority.
- Optional/preferred Triton posture -> Triton-only production inference with
  fail-closed enforcement.
- General tracking/overlay expectations -> Identity- and temporal-truth
  preconditions for any behavioral claim.
- Broad test/documentation rules -> Evidence-based production and scientific
  acceptance gates.

Added sections:
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
- updated: docs/backend/architecture/triton-operations.md
- updated: docs/backend/architecture/data-flow.md
- updated: docs/backend/architecture/deployment-topology.md
- updated: docs/backend/architecture/observability-runbook.md
- updated: docs/ARCHITECTURE.md
- updated: README.md
- updated: docs/triton_inference_speed_stabilization_plan.md
- updated: docs/architecture/runtime-scenario-matrix.md
- reviewed/already aligned: AGENTS.md
- reviewed/already aligned: docs/linux_production_optimization_execution_phases.md

Follow-up TODOs:
- Pending alignment review: the in-progress user-owned
  specs/010-behavioral-maturity-closure/plan.md must be checked against the
  stricter production inactive-endpoint-unreachable rule before approval.
- Implementation gaps identified by this constitution must be planned and
  closed as acceptance work; they are not placeholder governance text.
-->

# Production Behavioral Intelligence Platform Constitution

## Core Principles

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

### 14. Final Architectural Positioning

#### 14.1 Current Maturity

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

#### 14.2 Transitional Architecture State

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

#### 14.3 Target Production Identity

The target platform is a native-Linux, Triton-only, single-active-mode,
GPU-backed behavioral intelligence system that deterministically transforms
governed live or offline inputs into identity-continuous temporal sequences,
versioned pose streams, interpretable behavioral features, traceable event
candidates, trustworthy metrics and reproducible evidence.

True temporal intelligence exists when an output can answer, with evidence:
what was observed, for whom, over which valid interval, using which source and
model authority, under which missingness and confidence constraints, according
to which behavioral definition, and with which reproducible validation.

#### 14.4 Future AI-Readiness Posture

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
- maturity claims without reproducible evidence.

Operational SLO numbers, model-quality thresholds, retention changes, access
policy changes, and behavioral ontology decisions MUST be recorded in the
feature plan and evidence artifacts when they are not fixed by this
constitution. Such values are engineering decisions subject to validation, not
license to weaken these laws.

**Version**: 2.0.0 | **Ratified**: 2026-02-27 | **Last Amended**: 2026-05-25
