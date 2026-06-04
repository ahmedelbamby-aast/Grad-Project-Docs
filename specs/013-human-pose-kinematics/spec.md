# Feature Specification: Human Pose Kinematics Layer

**Feature Branch**: `013-human-pose-kinematics`  
**Created**: 2026-06-04  
**Status**: Draft  
**Input**: User description: "Create a Pose Human Forward & Inverse Kinematics Layer after RTMPose and before higher-level fusion / behavior reasoning. Convert raw skeleton keypoints into human-understandable body mechanics: quality, normalization, body anchors, joint angles, limb vectors, torso and head orientation, sitting/standing kinematics, hand raising, attention support, missing-joint estimation, physical validity, skeleton correction, temporal smoothing, and fusion-ready pose evidence."

## Clarifications

### Session 2026-06-04

- Q: Evidence Persistence Boundary → A: Compact summary in DB; full keypoint evidence in artifacts/exports.
- Q: Temporal History Bound → A: Same bound everywhere: ≤5 seconds, measurable and trackable.
- Q: Reviewer-Labeled Acceptance Scope → A: Validate posture, hand raising, torso/head orientation, and attention support.
- Q: Correction Authority In Fusion → A: Corrected pose may override gaze/behavior only when pose confidence is higher than the overridden model, same-student bounded history shows the signal is not a spike, and full traceability/monitoring records contradiction root causes.
- Q: Override Gate Threshold → A: Default gate is pose confidence at least `0.15` higher than the overridden model and at least `3` supporting same-student frames inside the `5` second history window; these values must be configurable through `.env`/configuration files and must never be hardcoded.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Explain Pose Mechanics Per Student (Priority: P1)

As an instructor, analyst, or reviewer, I need each pose-enabled student frame to produce understandable body-mechanics evidence so attention, posture, hand raising, and behavior conclusions can be explained from skeleton evidence rather than opaque keypoints alone.

**Why this priority**: This is the minimum useful slice. Raw keypoints do not tell an operator whether a student is seated, turned away, leaning, raising a hand, or physically unreliable.

**Independent Test**: Can be fully tested by replaying a representative classroom video with pose keypoints and verifying that each pose-eligible student frame receives quality, anchor, orientation, posture, gesture, and validity evidence without changing terminal job state.

**Acceptance Scenarios**:

1. **Given** a tracked student has sufficient pose keypoints in a frame, **When** the pose kinematics layer evaluates the student, **Then** the output includes pose quality, body anchors, normalized geometry, joint-angle evidence, torso/head orientation, posture class, hand-raise class, attention-support evidence, and physical-validity status.
2. **Given** a tracked student has weak or partial pose keypoints, **When** the pose kinematics layer evaluates the student, **Then** the output is marked weak, partial, occluded, invalid, or unknown with clear missing-keypoint and confidence reasons instead of fabricating a confident state.

---

### User Story 2 - Stabilize Pose Signals Across Time (Priority: P2)

As a behavior-reasoning user, I need pose signals to remain stable across short frame-to-frame jitter so hand raising, head direction, sitting/standing, and attention support do not flicker from noisy keypoints.

**Why this priority**: Classroom students are frequently seated, occluded, or partly hidden by desks. Temporal stability is required before pose evidence can safely support gaze or behavior reasoning.

**Independent Test**: Can be tested by comparing raw frame-level pose states against stabilized pose states over a labeled sequence and measuring reduced flicker without hiding true state changes.

**Acceptance Scenarios**:

1. **Given** a tracked student has consistent pose over adjacent frames, **When** the layer evaluates temporal pose state, **Then** smoothed keypoints, stability score, motion velocity, and sudden-jump flags are produced for that track.
2. **Given** a tracked student genuinely changes pose, **When** the layer evaluates the transition, **Then** the change is preserved and not flattened into the previous state.

---

### User Story 3 - Provide Fusion-Ready Pose Evidence (Priority: P3)

As a higher-level analytics system, I need pose-derived evidence to support gaze, posture, hand-raising, attention, and behavior fusion while preserving clear provenance and validity boundaries.

**Why this priority**: Pose should validate and enrich behavior reasoning. Corrected pose may override gaze or behavior only through an explicit high-confidence override path that preserves the original prediction, the corrected evidence, and the reason for the override.

**Independent Test**: Can be tested by feeding pose evidence into an isolated fusion review and confirming that every fusion decision can trace whether pose supported, contradicted, or was unavailable for the behavior claim.

**Acceptance Scenarios**:

1. **Given** gaze or behavior evidence disagrees with pose orientation, **When** fusion reviews the frame, **Then** the pose layer supplies support, contradiction, unavailable, or high-confidence override evidence with confidence and reason fields.
2. **Given** pose evidence is unavailable or invalid, **When** fusion reviews the frame, **Then** the fusion state records pose unavailable and continues using the other available evidence sources without treating pose as negative evidence.

---

### Edge Cases

- Student is seated behind a desk with knees or ankles hidden.
- Student is partially outside the frame.
- Multiple students overlap and a skeleton may swap left/right limbs.
- A wrist, elbow, hip, knee, or face point is missing or low confidence.
- The head is visible but shoulders or hips are missing.
- The lower body is hidden but the torso/head/arm evidence is strong.
- Sudden keypoint jumps occur across adjacent frames for the same track.
- Corrected pose conflicts with gaze or behavior for one frame but the same student's bounded history suggests a spike rather than a true model contradiction.
- A track ID changes or is missing for a short interval.
- A pose is physically impossible or has inconsistent limb lengths.
- A live stream drops frames or reconnects with a temporal gap.
- Offline replay is re-run for the same video and evidence must remain comparable.

### Mandatory Runtime Scenarios *(video/inference features)*

- **Live Stream Scenario**: The live classroom flow must evaluate pose kinematics only from the current frame and at most `5` seconds of per-track history. It must not require whole-video access, backward seeking, unbounded frame buffering, or retroactive mutation of previously emitted live evidence. Live outputs must include timestamp, track identity, pose validity, unavailable/degraded reasons, history-window size, history sample count, bound-violation count, and bounded latency/backpressure evidence. If pose keypoints or track continuity are unavailable, the live state must report `unknown`, `unavailable`, or `degraded` rather than blocking the camera flow.
- **Offline Video Processing Scenario**: The offline video flow must produce deterministic pose kinematics evidence for every pose-eligible tracked student frame in the replay. Offline processing may use at most `5` seconds of per-track history for smoothing and correction, but acceptance evidence must distinguish raw, corrected, smoothed, and unavailable states. Replays must produce comparable evidence for the same input video, runtime profile, and configuration. Compact fusion-ready summaries are the durable relational evidence; full raw/normalized/corrected/smoothed keypoint arrays are preserved in versioned artifacts or exports.
- **Frontend-Backend Wiring**: User-visible and exported states must expose pose validity, pose quality, posture class, hand-raise class, attention-support class, orientation, physical-validity status, override state, contradiction state, and degraded/unavailable reasons. Consumers must be able to distinguish `unknown`, `unavailable`, `degraded`, `valid`, `contradiction`, and `override_applied` pose states without reading raw keypoint internals.
- **Backend-Inference Wiring**: The layer consumes the project-authoritative pose keypoint output and must not introduce an alternate pose authority. The selected production inference profile, model artifact identity, inactive-profile rejection, timeout behavior, and failure semantics must be recorded when the feature is validated. If pose keypoint production fails, the layer must emit unavailable evidence and must not fabricate successful pose mechanics.
- **Temporal/Identity/Pose Authority**: Every output must carry frame time, source time where available, track identity, pose-source provenance, keypoint visibility, feature-version identity, invalid-window rules, and the measured temporal-history window used. Track-level smoothing and correction may use at most `5` seconds of per-track history and must record when estimated or smoothed keypoints affect the final evidence.
- **Deployment Boundary**: Development and local checks are contract-only. Production acceptance requires the native Linux GPU production path, the active inference profile, no hidden fallback route, and recorded environment/runtime evidence. The feature must preserve single-active-mode assumptions and must not depend on elevated host privileges.
- **Concurrency/Worker Scaling Boundary**: N/A for this specification. The feature does not request worker, thread, pool, queue, or concurrency topology changes. If a future plan proposes such changes, that proposal must define baseline and candidate topology, resource budgets, duplicate-worker detection, rollback, and production benchmark proof before acceptance.
- **Heterogeneous Authority Boundary**: Local tests may validate schema, state contracts, and edge-case behavior. Production claims about throughput, latency, GPU impact, replay correctness, or operational readiness require native production evidence with Git SHA, environment fingerprint, PostgreSQL state, Redis state if touched, worker state, inference profile, and durable evidence root.
- **Replay Policy**: Acceptance runs use `new-attempt` replay policy. Failed replay lineage, partial runs, local dry-runs, or component probes cannot count as acceptance evidence.
- **Benchmark Decision Evidence**: Any accepted runtime implementation must include a production end-to-end benchmark on the canonical classroom video or an explicitly documented equivalent. The decision table must include baseline replay/job, candidate replay/job, exact video, deployed SHA, environment/configuration delta, FPS, latency percentiles, per-model call rate/status/input shape, per-step and per-phase timings, GPU, memory, database/model correctness, model-agreement or reviewer-label agreement, causal interpretation, remaining bottleneck, rollback proof, and durable evidence paths. Component probes are hypothesis-only and cannot accept, reject, skip, close, or deprioritize the feature.
- **Runtime Reconciliation**: Task state, queue state, database state, artifacts, telemetry, and frontend/export state must converge. If the pose kinematics layer cannot produce valid evidence for a frame or track, the reconciled output must preserve the frame/track lineage and the unavailable reason. If corrected pose overrides a gaze or behavior prediction, reconciliation must preserve the original prediction, corrected pose evidence, override reason, confidence values, same-student history summary, spike-versus-contradiction classification, and contradiction/root-cause monitoring fields.
- **Evidence Lineage**: Evidence snapshots must identify input video or stream, frame/time range, source keypoint provenance, feature version, configuration, runtime profile, validation dataset, reviewer-label source if used, artifact digests where available, and whether results came from local, synthetic, production, live, or offline execution.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST produce a pose kinematics evidence record for every pose-eligible tracked student frame. Acceptance: replay evidence shows one valid, degraded, unavailable, or unknown pose kinematics state for each pose-eligible track/frame pair.
- **FR-002**: System MUST preserve raw keypoints and clearly distinguish raw, normalized, corrected, and smoothed keypoint views. Acceptance: compact relational evidence exposes which views exist and whether correction or smoothing was applied, while full keypoint arrays are available in versioned artifacts or exports.
- **FR-003**: System MUST compute keypoint quality features including mean confidence, minimum confidence, visible keypoint count, missing keypoints, low-confidence keypoints, and a quality label. Acceptance: partial and occluded examples produce non-confident labels with explicit reasons.
- **FR-004**: System MUST compute skeleton normalization and body geometry features including person height, torso height, shoulder width, hip width, torso length, body scale, and body anchor points. Acceptance: each valid record includes body center, head center, neck or shoulder center, hip center, torso center, and geometry values or a reason they are unavailable.
- **FR-005**: System MUST compute forward-kinematic features including joint angles, limb vectors, torso lean, head tilt, shoulder-line angle, and hip-line angle where source points are reliable enough. Acceptance: unavailable angles identify the missing or low-confidence source joints.
- **FR-006**: System MUST estimate torso orientation and head pose as categorical states with confidence and supporting feature values. Acceptance: outputs include torso yaw, torso pitch, torso roll, head yaw, head pitch, head roll, and head-torso alignment or unknown reasons.
- **FR-007**: System MUST produce posture evidence for sitting, standing, leaning, crouching, partial-body, and unknown states. Acceptance: each posture output includes confidence and supporting features such as torso verticality, hip/knee evidence, lower-body visibility, and bounding-box shape where available.
- **FR-008**: System MUST produce hand-raising evidence for none, left hand, right hand, both hands, and uncertain states. Acceptance: hand-raise outputs include wrist-above-shoulder/head checks, elbow evidence, side-specific confidence, and missing-joint reasons.
- **FR-009**: System MUST produce attention-support evidence that states whether pose supports attention, possible distraction, looking down, turned-away posture, peer-facing posture, or unknown. Acceptance: pose evidence is recorded as support, contradiction, unavailable, or override-applied evidence for higher-level reasoning; any override of gaze or behavior preserves the original prediction, corrected-pose basis, and same-student bounded-history evidence.
- **FR-010**: System MUST detect physical-constraint violations including limb-length inconsistency, implausible angle, sudden skeleton flip, impossible joint crossing, and unstable shoulder/hip geometry. Acceptance: violation lists include type, affected joint or segment, severity, and impact on pose validity.
- **FR-011**: System MUST estimate missing joints only when confidence and body-geometry constraints support a bounded estimate. Acceptance: estimated joints have lower confidence than observed joints and record the estimation reason.
- **FR-012**: System MUST apply temporal smoothing and motion/stability evidence per track using no more than `5` seconds of history. Acceptance: outputs include smoothed keypoints where available, motion velocity for key body parts, pose stability score, sudden-jump detection, measured history-window duration, history sample count, and any bound-violation count.
- **FR-013**: System MUST expose a compact fusion-ready pose evidence summary containing the most important project signals: corrected/smoothed keypoint availability, pose quality, torso yaw, head yaw, head pitch, head-torso alignment, posture class, hand-raise class, attention-support score, physical-validity score, and constraint violations. Acceptance: downstream consumers can read these fields from durable relational evidence without deriving them from raw keypoints.
- **FR-014**: System MUST preserve streaming compatibility. Acceptance: live-mode evaluation uses only current frame data and bounded per-track state; offline-only assumptions are rejected for live scenarios.
- **FR-015**: System MUST record telemetry and evidence lineage for accepted, degraded, unavailable, and failed pose kinematics outcomes. Acceptance: benchmark/export evidence can explain count, rate, reason, compact summary location, and full artifact/export location for each outcome class.
- **FR-016**: System MUST provide rollback and non-blocking failure behavior. Acceptance: if the layer is disabled or unavailable, existing pose and behavior flows continue with explicit `unavailable` pose kinematics state rather than false success.
- **FR-017**: System MUST monitor pose-vs-gaze and pose-vs-behavior contradictions and overrides. Acceptance: every contradiction or override records source predictions, corrected pose evidence, confidence values, selected outcome, reason code, frame/track lineage, same-student bounded-history summary, spike-versus-contradiction classification, and aggregate monitoring counts for root-cause analysis.
- **FR-018**: System MUST block corrected-pose overrides when pose confidence does not exceed the active configured margin over the overridden model or when same-student bounded history has fewer than the active configured number of supporting frames. Acceptance: default values are margin `0.15` and minimum supporting frames `3` inside the `5` second history window, but the active values come from `.env`/configuration files, are recorded in evidence, and are never hardcoded.
- **FR-019**: System MUST expose override-gate configuration and monitoring evidence. Acceptance: every benchmark/export identifies the active confidence margin, supporting-frame threshold, history-window bound, override count, blocked-spike count, and blocked-low-confidence count.

### Key Entities *(include if feature involves data)*

- **Pose Observation**: A per-student, per-frame pose input containing track identity, frame identity, time, bounding box, keypoints, and source confidence.
- **Kinematics Evidence Record**: The final per-student, per-frame output that summarizes validity, quality, normalized geometry, orientation, posture, gesture, attention support, motion, and inverse-kinematic status.
- **Pose Evidence Artifact**: A versioned artifact or export containing full raw, normalized, corrected, and smoothed keypoint arrays for reproducibility without requiring every array to be stored in relational evidence.
- **Keypoint Quality Assessment**: Visibility, confidence, missingness, low-confidence points, and quality labels used to decide whether a pose is reliable.
- **Body Geometry Profile**: Normalized skeleton scale, body anchors, limb lengths, and reference distances for comparing students across camera distance and scale.
- **Orientation Assessment**: Torso and head yaw/pitch/roll, lean direction, and head-torso alignment.
- **Posture And Gesture Evidence**: Sitting/standing/leaning evidence, hand-raise evidence, and their supporting measurements.
- **Inverse-Kinematic Correction Evidence**: Estimated keypoints, physical-constraint violations, correction confidence, and correction notes.
- **Temporal Pose State**: Per-track smoothing, velocity, stability, and sudden-jump evidence computed from a measurable history window of at most `5` seconds.
- **Fusion Pose Signal**: A compact, versioned subset of pose evidence intended for higher-level attention, gaze validation, behavior reasoning, and analytics.
- **Fusion Override Event**: A traceable decision where corrected pose evidence changes the selected gaze or behavior outcome while preserving the original model output, confidence values, active override-gate values, same-student history summary, reason code, spike-versus-contradiction classification, and contradiction context.

### Assumptions

- The layer is positioned after the existing pose keypoint stage and before higher-level fusion or behavior reasoning.
- 2D keypoints are the initial required input; 3D keypoints, depth, calibration, and additional gaze outputs are optional future enrichments.
- Pose estimates must support both offline video processing and live streaming; any whole-video or history window beyond `5` seconds is out of scope for this feature and cannot be enabled for live.
- Pose validates and supports gaze/behavior reasoning by default. Corrected pose may replace the selected gaze or behavior outcome only through an explicit high-confidence override event where pose confidence is higher than the overridden model, same-student bounded history does not indicate a spike, and original evidence plus monitoring are preserved.
- Override-gate defaults are a `0.15` pose-confidence margin and `3` supporting same-student frames inside the `5` second history window, but these are deployment configuration values. Hardcoded threshold values are outside the accepted scope.
- A human-labeled validation subset will be available before acceptance for posture, hand-raise, torso/head orientation, and attention-support agreement checks.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: At least 98% of pose-eligible tracked student frames produce a kinematics evidence record with one of `valid`, `degraded`, `unavailable`, or `unknown`.
- **SC-002**: At least 95% of invalid, partial, or occluded pose examples include an explicit missingness or confidence reason visible in exported evidence.
- **SC-003**: On a reviewer-labeled classroom validation set of at least 500 person-frame samples, posture class agreement reaches at least 90%.
- **SC-004**: On a reviewer-labeled classroom validation set of at least 500 person-frame samples, hand-raise class agreement reaches at least 92%.
- **SC-005**: On a reviewer-labeled classroom validation set of at least 500 person-frame samples, torso/head orientation agreement reaches at least 85%.
- **SC-006**: On a reviewer-labeled classroom validation set of at least 500 person-frame samples, attention-support agreement reaches at least 80%.
- **SC-007**: Live stream scenario emits pose kinematics evidence or explicit unavailable/degraded state within the live session latency budget for at least 95% of pose-eligible frames, verified through the active production inference path with real media and evidence artifacts.
- **SC-008**: Offline video processing completes with pose kinematics enabled without more than a 10% throughput regression against the latest accepted baseline and with no regression in database/model correctness, verified through the active production inference path with real media and reproducible evidence artifacts.
- **SC-009**: Temporal/identity/pose validity is verified by showing that invalid windows, track gaps, missing keypoints, corrected keypoints, measured history-window duration, history sample count, and history bound violations are represented explicitly and are not counted as confident behavior evidence.
- **SC-010**: Telemetry and benchmark reports distinguish valid, degraded, unavailable, failed, corrected, smoothed, estimated, contradiction, and override-applied outcomes without synthetic success, and report the maximum observed history window with zero accepted records exceeding `5` seconds.
- **SC-011**: Runtime reconciliation proves task, database, queue, artifact, telemetry, and frontend/export states converge or expose a blocking fault.
- **SC-012**: Evidence artifacts are non-placeholder, immutable, digest-addressed where applicable, linked from compact relational summaries where relational state is involved, and reproducible from recorded lineage.
- **SC-013**: No hidden fallbacks, unsupported live assumptions, unbounded history, environment drift, or production claims based only on local runs remain for the accepted scope.
- **SC-014**: N/A for production runtime maturity final packaging unless this feature is promoted into a runtime-maturity acceptance wave.
- **SC-015**: If implementation adds async jobs, vector persistence, or orchestration changes, the accepted scope declares per-stage deadlines, reconciler behavior, idempotency keys, error-ratio fail-closed thresholds, and re-run tests before acceptance.
- **SC-016**: N/A for worker/thread/concurrency changes because this feature does not request concurrency topology changes; any future topology candidate remains hypothesis-only until a governed production benchmark proves improvement without resource or correctness regression.
- **SC-017**: For all override-applied outcomes, 100% of sampled evidence includes original prediction, corrected pose basis, confidence values, active override-gate values, same-student bounded-history summary, spike-versus-contradiction classification, reason code, selected outcome, and root-cause monitoring category.
- **SC-018**: In reviewer audit samples, 100% of cases where pose confidence does not exceed the active configured margin or same-student history has fewer than the active configured number of supporting frames are not counted as override-applied outcomes.
- **SC-019**: Benchmark and export evidence proves that override confidence margin and supporting-frame thresholds are read from `.env`/configuration files by showing active values, default values, and zero hardcoded-threshold violations.
