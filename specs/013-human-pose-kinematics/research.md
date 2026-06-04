# Research: Human Pose Kinematics Layer

**Last updated:** 2026-06-04

## Decision 1: Implement a deterministic post-RTMPose layer

**Decision**: The Human Pose Kinematics Layer is deterministic Python logic
that consumes existing RTMPose keypoints after
`backend/apps/pipeline/services/pose_runtime.py` and before fusion/export.

**Rationale**: The feature asks for human-understandable body mechanics from
existing skeleton keypoints. Adding a new model would increase inference
traffic and would create a second pose authority, which violates the spec's
backend-inference boundary.

**Alternatives considered**:

- Add a new learned pose-intelligence model: rejected for this feature because
  it adds inference scope and requires separate training/validation authority.
- Fold the logic into RTMPose decode: rejected because kinematics evidence has
  downstream fusion and persistence concerns that should stay separate from
  model output decoding.

## Decision 2: Compact DB summary plus artifact-backed full arrays

**Decision**: Persist compact fusion-ready summaries and override events in
PostgreSQL. Store full raw, normalized, corrected, and smoothed keypoint arrays
in versioned artifacts/exports with digests.

**Rationale**: The user explicitly chose compact DB summaries with full
keypoint evidence in artifacts/exports. This also avoids large JSON write
amplification for every pose-eligible tracked student frame.

**Alternatives considered**:

- Store all keypoint arrays in PostgreSQL: rejected because it expands durable
  row size for high-frame-count offline and live sessions.
- Artifact-only evidence: rejected because fusion, monitoring, and review need
  compact queryable states and counters.

## Decision 3: Bound temporal history by configuration

**Decision**: Use a per-track history window with default
`POSE_KINEMATICS_HISTORY_SECONDS=5`. The implementation must also use a sample
cap so sparse or high-FPS streams cannot grow unbounded.

**Rationale**: The clarified requirement fixed a `5` second history bound. The
streaming constitution forbids unbounded per-job state and whole-video
assumptions on live streams.

**Alternatives considered**:

- Whole-video smoothing: rejected as offline-only and not live-compatible.
- Single-frame only: rejected because spike detection, smoothing, and
  same-student contradiction checks require bounded temporal evidence.

## Decision 4: Override gates are configuration-driven

**Decision**: The default override gate is pose confidence at least
`POSE_KINEMATICS_OVERRIDE_MARGIN=0.15` higher than the overridden model and at
least `POSE_KINEMATICS_OVERRIDE_MIN_FRAMES=3` supporting same-student frames
inside the configured history window. These values must be read from settings,
not hardcoded in service logic.

**Rationale**: The user explicitly required no hardcoded values. Configuration
also makes production rollback and calibration possible without changing code.

**Alternatives considered**:

- Fixed constants in the kinematics service: rejected by requirement.
- Allow pose to always override gaze/behavior: rejected because corrected pose
  must only override with higher confidence, bounded history support, and full
  traceability.

## Decision 5: Trace contradictions instead of hiding them

**Decision**: Fusion receives `support`, `contradiction`, `unavailable`, and
`override_applied` states with root-cause fields and original/corrected
evidence preserved.

**Rationale**: The user selected high traceability and monitoring to understand
model conflicts. This makes one-frame spikes distinguishable from persistent
cross-model disagreement.

**Alternatives considered**:

- Suppress contradicted behavior rows silently: rejected because live evidence
  must not be retroactively mutated and because the project needs conflict
  root-cause monitoring.
- Only log contradictions: rejected because downstream consumers and reviewers
  need structured states, not ad hoc log messages.

## Decision 6: Stream-safe-with-config compatibility

**Decision**: The feature is `stream-safe-with-config`: enabled logic may use
only current frame plus bounded per-track history; live outputs are append-only;
corrections never rewrite previously emitted live evidence.

**Rationale**: Offline and live pipelines share the inference plane. The
feature must support RTSP/RTSPS/WHEP-style streams without buffering collapse
or backward seek.

**Alternatives considered**:

- Offline-only kinematics artifact pass: rejected because the spec requires
  live state exposure and streaming compatibility.
- Post-job correction pass for all frames: rejected for live because it relies
  on whole-job availability and retroactive mutation.

## Decision 7: Benchmarks are evidence, probes are not decisions

**Decision**: Local tests, direct probes, and sample labeled checks are
necessary but cannot accept/reject the feature. A completed production Linux
RTX 5090 benchmark on `combined.mp4` is required before maturity or
performance claims.

**Rationale**: The constitution makes production benchmark decision authority
binding. The kinematics layer changes evidence and may affect fusion/exports,
so correctness and performance must be measured end to end.

**Alternatives considered**:

- Accept from unit/integration tests only: rejected because they do not measure
  production runtime cost or pipeline interaction.
- Accept from component probes: rejected because probes are hypothesis-only.
