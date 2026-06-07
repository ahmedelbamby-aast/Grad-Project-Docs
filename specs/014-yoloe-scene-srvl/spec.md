# Feature Specification: YOLOE Scene Segmentation And SRVL

**Feature Branch**: `014-yoloe-scene-srvl`  
**Created**: 2026-06-07  
**Status**: Draft  
**Input**: User description: "Create a Spec Kit specification from `docs/yoloe_scene_segmentation_plan.md` for YOLOE scene segmentation, prompt-locked export, people mismatch recovery, non-ROI classroom guards, Spatial Relationship Vectorization Layer matrices/maps, frontend visualization, production evidence, and governance."

## Clarifications

### Session 2026-06-07

- Q: Should V1 automatically re-pass missing YOLOE people through the full inference path? → A: Candidate-only by default; re-pass remains disabled until a separate measured candidate proves correctness, latency, and timeline safety.
- Q: What canonical artifact formats should V1 use for masks and spatial matrices? → A: RLE-Zstd masks and compressed NPZ matrices, each referenced by a JSON manifest with digests.
- Q: Which renderer should planning treat as the first production visualization candidate? → A: PixiJS WebGL first, with deck.gl or Three.js considered only after benchmark evidence rejects the simpler path.
- Q: When should SRVL switch away from dense full matrices? → A: Full dense matrices up to 128 objects per frame; above that, top-k or thresholded output is required.
- Q: What is the evidence access boundary for scene artifacts containing classroom imagery? → A: Job-scoped authenticated artifacts; no public unauthenticated mask, snapshot, matrix, or scene-video access.
- Q: How should YOLOE outputs and operational parameters be governed? → A: Preserve every YOLOE output emitted by the runtime route, especially confidence scores and masks, for contradiction/recovery decisions; every tunable prompt, threshold, score weight, cadence, limit, codec, renderer, and mode must be declared through `.env`/settings with no hardcoded operational constants in feature code.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Segment Classroom Scene Evidence (Priority: P1)

As an instructor, analyst, or reviewer, I need each eligible offline video frame to produce traceable scene segmentation evidence for people and non-ROI classroom regions so person counts, hallucination checks, and classroom-map review are grounded in pixel-level evidence.

**Why this priority**: This is the minimum useful slice. Without prompt-locked scene segmentation, the system cannot independently count people, locate missed people, or identify regions where downstream models should be treated with caution.

**Independent Test**: Can be tested by replaying a classroom video with scene segmentation enabled and verifying that selected frames produce person masks, non-ROI masks, scene summaries, mask artifacts, and contradiction evidence while the existing pipeline remains unchanged when disabled.

**Acceptance Scenarios**:

1. **Given** an offline classroom video job has scene segmentation enabled, **When** the job reaches a selected scene frame, **Then** the output includes prompt-profile identity, scene object summaries, YOLOE confidence scores, all available YOLOE result fields, recoverable person masks, recoverable non-ROI masks, and artifact lineage.
2. **Given** scene segmentation is disabled, **When** the same offline video is processed, **Then** the existing video-analysis outputs and terminal job state are preserved with no scene-only side effects.
3. **Given** a person detector box overlaps a non-ROI region such as a table, chair, wall, floor, mirror, board, screen, door, window, or ceiling, **When** the scene guard evaluates the frame, **Then** an append-only contradiction event is recorded without deleting or mutating the original detection.

---

### User Story 2 - Reconcile Missing People (Priority: P2)

As a system reviewer, I need the system to investigate cases where YOLOE sees a different number of people than the current student-plus-teacher count with recorded lookup latency so missed people can be located and false alarms can be separated from real detector misses.

**Why this priority**: People count mismatches directly affect downstream classroom reasoning. The feature must distinguish missed detections, duplicates, and false alarms before recovery evidence is trusted.

**Independent Test**: Can be tested by feeding frames with known count mismatches and verifying that the system reads existing appearance evidence first, produces one-to-one match decisions, records unresolved ambiguity, and sends only eligible missing-person candidates through the recovery path.

**Acceptance Scenarios**:

1. **Given** YOLOE detects more people than the current student-plus-teacher count, **When** mismatch recovery runs, **Then** each YOLOE person is classified as matched existing person, false alarm or duplicate, missing candidate, or unresolved with traceable scores and reasons.
2. **Given** a YOLOE person remains unmatched after Redis-first appearance and geometry checks, **When** the candidate passes configured confidence, area, non-ROI, and temporal gates, **Then** the system creates append-only recovery evidence and may re-pass that candidate through the existing inference timeline when recovery re-pass is enabled.
3. **Given** the mismatch evidence is ambiguous, **When** recovery scoring cannot make a confident one-to-one decision, **Then** the candidate remains unresolved and is not force-merged into an existing identity.

---

### User Story 3 - Visualize Spatial Relationships (Priority: P3)

As a frontend user or analyst, I need a smooth 2D scene map, distance matrix, vector matrix, and heatmap/correlation view so object relationships in the classroom can be inspected without freezing playback or blocking inference.

**Why this priority**: Spatial relationships are useful only if visualization stays non-blocking and traceable. The visualization must support human review while analytics artifacts remain complete and digest-addressed.

**Independent Test**: Can be tested by running the spatial layer on representative object counts and verifying matrix correctness, map artifacts, frontend frame rate, dropped visualization counters, and unchanged backend progress when rendering exceeds the configured late-frame threshold.

**Acceptance Scenarios**:

1. **Given** a scene frame has ordered objects with coordinates, **When** the spatial layer evaluates the frame, **Then** it produces distance, angle, vector, heatmap, direction, and correlation outputs with frame, object, coordinate, timing, and artifact references.
2. **Given** the frontend cannot keep up with scene-map updates, **When** new visualization states arrive, **Then** stale visualization work is dropped while analytics artifacts continue to be written reliably.
3. **Given** a frame does not run scene segmentation, **When** the scene-map video is rendered, **Then** the renderer reuses the latest valid scene state so the output video preserves source frame count and timing.

---

### User Story 4 - Prove Production Readiness (Priority: P4)

As a maintainer, I need every implementation phase to produce real tests, real production benchmark evidence, helper scripts, manifests, figures, and rollback proof so the feature cannot be accepted from mocks, templates, or unsupported local runs.

**Why this priority**: The feature touches inference, identity, artifacts, frontend visualization, and benchmark claims. Acceptance must be based on real production evidence and must obey the repository constitution and `AGENTS.md`.

**Independent Test**: Can be tested by reviewing the feature evidence package and verifying that each accepted claim has a real source file, test output, benchmark artifact, generated figure, manifest digest, production job lineage, and rollback proof.

**Acceptance Scenarios**:

1. **Given** an implementation phase claims readiness, **When** reviewers inspect the evidence package, **Then** every claim resolves to real tracked files or linked production artifacts and missing metrics are marked unavailable with reasons.
2. **Given** the production benchmark is run, **When** results are recorded, **Then** the decision table includes baseline/candidate lineage, exact video, deployed revision, configuration delta, throughput, latency, GPU, memory, database, Redis, correctness, frontend, figure, and rollback evidence.
3. **Given** rollback is requested, **When** scene segmentation and spatial relation features are disabled, **Then** the existing offline pipeline behavior is restored and verified by tests and production evidence.

### Edge Cases

- YOLOE returns zero people, one person, duplicate people, or low-confidence person masks.
- YOLOE person count is higher than, lower than, or equal to student-plus-teacher count.
- A YOLOE person mask overlaps a non-ROI region.
- A downstream person detector fires on a table, chair, wall, mirror, floor, ceiling, board, screen, door, or window.
- Redis appearance evidence is missing, expired, unavailable, malformed, or stale.
- A YOLOE recovery candidate has no appearance vector meeting the configured validity checks.
- Multiple candidates compete for one existing track or one candidate competes for multiple tracks.
- A recovery candidate is plausible in one frame but disappears in adjacent frames.
- A frame has more objects than the dense matrix threshold.
- A spatial relation matrix has zero objects, one object, invalid coordinates, duplicate object identifiers, or out-of-frame coordinates.
- Frontend rendering is slower than backend scene production.
- More than 128 scene objects appear in one frame.
- GPU metrics tools are unavailable on the production server.
- A prompt profile changes after a model export already exists.
- A previous TensorRT or ONNX artifact has the wrong prompt digest.
- A YOLOE runtime/export route omits a field that the PyTorch result normally exposes.
- A confidence, overlap, recovery, cadence, matrix, codec, or renderer parameter is missing from `.env`/settings.
- A developer accidentally hardcodes an operational prompt class, threshold, score weight, stride, dense limit, render mode, or artifact codec in source code.
- The feature is enabled for a live source before a live-specific validation plan exists.
- A user without job access attempts to fetch scene masks, snapshots, matrices, or scene-map video.

### Mandatory Runtime Scenarios *(video/inference features)*

- **Live Stream Scenario**: V1 live use is disabled. Live streams must not enable YOLOE scene segmentation, mismatch recovery re-pass, or SRVL-driven suppression until a separate live-specific plan proves latency, bounded per-camera queues, append-only evidence, no seekability dependency, frame-drop accounting, reconnect gap handling, and frontend responsiveness. If accidentally requested for live mode, the feature must report `unavailable` or `disabled_for_live` and must not block the live camera flow.
- **Offline Video Processing Scenario**: The offline upload/raw-video flow is the V1 target. The system processes decoded frames, runs scene segmentation only on configured scene frames, stores compact summaries and recoverable artifacts, runs mismatch and non-ROI guards when enabled, produces spatial relation artifacts when enabled, and renders scene-map media without changing existing outputs when the feature is disabled. Acceptance requires a real production run on `/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4`.
- **Frontend-Backend Wiring**: API, WebSocket, and artifact contracts must expose optional scene summary, object summary, contradiction events, recovery candidate state, matrix/map artifact references, renderer status, and user-visible truth states including `unknown`, `unavailable`, `degraded`, `valid`, `contradiction`, `recovery_candidate`, and `unresolved`. Existing playback must remain usable when scene fields are absent. Scene artifacts containing classroom imagery must be job-scoped and authenticated.
- **Backend-Inference Wiring**: The scene model route must be prompt-locked before export and must reject stale prompt digests. Production inference authority remains the active native model-serving route; local, mock, or prompt-free exports cannot be accepted as production evidence. Model failures emit unavailable/degraded scene evidence and must not fabricate successful masks. The route must normalize and preserve every YOLOE output it receives, including boxes, class IDs/names, confidence scores, segmentation masks, shape metadata, raw-output references when exposed, and unavailable reasons for outputs that a deployed export cannot provide.
- **Temporal/Identity/Pose Authority**: Every scene object, mismatch event, recovery candidate, and spatial relation output must carry job identity, video identity, frame number, timestamp, prompt profile, model artifact digest, object identity or frame-local relation index, coordinate source, and track/ReID provenance where available. Recovery candidates use source-scoped provisional IDs until governed assignment resolves them.
- **Independent-Run/Sharded Identity Evaluation**: Raw local tracker labels are not identity proof. Mismatch recovery must use documented observation matching and deterministic one-to-one assignment when comparing YOLOE persons to existing student/teacher tracks. Detection/localization quality, association quality, false alarms, missing candidates, unresolved cases, duplicate matches, and proxy-ground-truth limitations must be reported separately. Sharded identity transport is out of scope for V1.
- **Deployment Boundary**: Local and Windows checks are contract evidence only. Production acceptance requires native Linux RTX 5090 execution, PostgreSQL, Redis when recovery is enabled, active worker state, active model route, no Docker, no `sudo`, environment fingerprint, and durable evidence root. User-space tooling is allowed when recorded and reproducible.
- **Concurrency/Worker Scaling Boundary**: V1 does not request worker, pool, or Celery topology changes. The feature may use bounded internal queues for scene compute and visualization. Any future worker/thread/concurrency change must define baseline/candidate topology, resource budgets, duplicate-worker detection, rollback, and production benchmark proof before acceptance.
- **Heterogeneous Authority Boundary**: Local tests may validate schemas, prompt-digest checks, object normalization, matrix math, recovery scoring, API compatibility, and frontend behavior. Claims about throughput, latency, GPU utilization, model readiness, renderer smoothness, or production acceptance require native Linux RTX 5090 evidence tied to Git SHA, `backend/.env`, PostgreSQL, Redis, worker state, model route, and artifacts.
- **Replay Policy**: Acceptance runs use `new-attempt` replay policy on the canonical production video. Failed runs, partial runs, smoke tests, reduced clips, dry-runs, or component probes cannot count as acceptance evidence, although they may be recorded as hypothesis or debugging evidence.
- **Benchmark Decision Evidence**: Any accepted candidate must include a completed native Linux RTX 5090 benchmark on `combined.mp4` with baseline and candidate replay/job identifiers, deployed SHA, environment/configuration delta, target gates, live/cumulative/database FPS, latency percentiles, model round-trip times, per-model call rate/status/input shape, per-step and per-phase timing, GPU utilization/memory/power/clocks, CPU/RSS, PostgreSQL/Redis timing, frontend FPS/frame time, artifact size, correctness/model-agreement or reviewer-label agreement, mismatch recovery counts, non-ROI contradiction distribution, causal interpretation, remaining bottleneck, rollback proof, and durable evidence paths. Component probes are hypothesis-only. The implementation cycle must designate exactly one Figure Planner and exactly one Figure Implementer before any benchmark decision; the figure package must include generated plots, source artifacts, digests, unavailable-metric policy, and Markdown embed targets.
- **Runtime Reconciliation**: Task state, queue state, model-route state, database state, Redis state, artifact manifests, telemetry, frontend status, and rendered media must converge. If YOLOE, SRVL, recovery, or rendering fails, the reconciled output must preserve frame/object lineage, unavailable reason, and drop/degrade counters without mutating prior evidence. Recovery re-pass is not part of the V1 default path and must remain disabled unless a separate measured candidate enables it.
- **Evidence Lineage**: Evidence snapshots must identify source video, frame/time range, prompt profile, checkpoint identity, export manifest digest, active `.env` configuration fingerprint, model route, feature version, runtime profile, GPU/runtime fingerprint, artifact digests, test command, benchmark command, replay/job lineage, and whether results are local, smoke, synthetic, production, live, or offline.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST provide a disabled-by-default scene segmentation feature for offline video jobs that preserves the existing pipeline when disabled.
- **FR-002**: System MUST use the YOLOE-26 segmentation small checkpoint as the default model for the first benchmark path unless a later governed benchmark cycle changes the model.
- **FR-003**: System MUST resolve the active prompt profile or ordered prompt class list before export and MUST apply the resolved classes before any ONNX or TensorRT export.
- **FR-004**: System MUST record a model export manifest containing checkpoint identity, source URL, prompt profile, ordered prompt classes, class count, prompt digest, export format, artifact digests, exporter version, and active environment values.
- **FR-005**: System MUST reject runtime use when the active prompt configuration does not match the exported artifact manifest.
- **FR-006**: System MUST treat YOLOE boxes, labels/class IDs, class names, confidence scores, masks, shape metadata, and any raw-output references exposed by the runtime route as one unified scene output and MUST preserve them or record explicit unavailable reasons for scene frames where YOLOE runs.
- **FR-007**: System MUST store compact scene summaries in durable indexed records and store large masks, matrices, heatmaps, snapshots, videos, and traces as artifact files with digests; V1 mask artifacts use RLE-Zstd and V1 dense matrix artifacts use compressed NPZ.
- **FR-008**: System MUST segment people and the configured non-ROI classroom classes, including tables/desks, chairs, walls, floors, ceilings, doors, windows, mirrors, boards, and screens.
- **FR-009**: System MUST build a non-ROI union mask and record append-only contradiction events when downstream detections or recovery candidates overlap non-ROI regions at or above the configured overlap threshold.
- **FR-010**: System MUST compare YOLOE person count with the current student-plus-teacher count and record match, mismatch, unavailable, and unresolved states.
- **FR-011**: System MUST use existing Redis-first appearance evidence first for mismatch recovery and MUST fall back to configured DB or geometry lookup only when configured and recorded.
- **FR-012**: System MUST classify each YOLOE person during mismatch recovery as matched existing person, false alarm or duplicate, missing candidate, or unresolved.
- **FR-013**: System MUST use deterministic one-to-one assignment for recovery matching and MUST keep ambiguous candidates unresolved rather than forcing identity merges.
- **FR-014**: System MUST create append-only missing-person recovery candidates with frame identity, timestamp, source mask, source bbox, prompt profile, export digest, score details, and reason codes.
- **FR-015**: System MUST keep recovery re-pass disabled by default; eligible missing-person candidates are candidate-only evidence unless a separate measured candidate enables downstream re-pass.
- **FR-016**: System MUST synchronize recovery candidates to the original frame time and track lineage without retroactively mutating prior persisted detections.
- **FR-017**: System MUST provide detector-assist evidence for missed regions in following frames bounded by configured TTL/lookahead frame settings, and MUST record whether recurrence is reduced or unresolved.
- **FR-018**: System MUST compute pairwise distance, angle, magnitude-angle vector, optional Cartesian vector, heatmap, direction map, and correlation map outputs for ordered scene objects.
- **FR-019**: System MUST produce correct zero-object, one-object, invalid-coordinate, duplicate-identifier, large-object-count, top-k, thresholded, and heatmap-only spatial outputs; dense full matrices are allowed only up to 128 objects per frame.
- **FR-020**: System MUST persist analytics artifacts independently of visualization and keep visualization non-blocking, with bounded queues, latest-frame-wins rendering, and recorded dropped/delayed/rendered counters.
- **FR-021**: System MUST render a 2D classroom scene map, configured-resolution snapshots, and MP4 review artifact while preserving source video frame count.
- **FR-022**: System MUST benchmark renderer candidates using representative scene payloads and MUST evaluate PixiJS WebGL as the first production candidate before adding more complex renderers.
- **FR-023**: System MUST expose backward-compatible API, WebSocket, artifact, and frontend contracts so older clients/jobs remain valid when scene fields are absent.
- **FR-024**: System MUST emit trace data for frame, object, coordinate, prompt, backend, timing, matrix mode, downsampling, recovery state, non-ROI state, visualization state, artifact references, and unavailable reasons.
- **FR-025**: System MUST provide production helper scripts for prompt-locked export, export validation, benchmark execution, metric collection, renderer benchmarking, figure generation, and rollback.
- **FR-026**: System MUST record all required production metrics, including throughput, latency, model calls, GPU, memory, CPU, database, Redis, frontend, artifact size, correctness, missing metrics, and drop/degrade counters.
- **FR-027**: System MUST create or update workflows, checklists, tasks, tests, helper scripts, and documentation when required for CI visibility, evidence validation, and reproducibility.
- **FR-028**: System MUST obey PostgreSQL-only relational persistence, source-control visibility, documentation, figure-evidence, streaming-compatibility, identity-association, and no-hallucination governance before acceptance.
- **FR-029**: System MUST provide rollback proof showing that disabling scene segmentation and spatial relation features restores the existing offline pipeline behavior.
- **FR-030**: System MUST require job-scoped authenticated access for scene masks, snapshots, matrices, rendered scene videos, and other artifacts containing classroom imagery.
- **FR-031**: System MUST use YOLOE confidence scores and configured confidence/overlap thresholds as traceable inputs to contradiction evidence, people mismatch classification, missing-candidate eligibility, false-alarm handling, and visualization truth states.
- **FR-032**: System MUST declare every tunable YOLOE/SRVL/recovery/render/artifact parameter in `.env` and parse it through the governed settings layer; hardcoded operational prompt classes, thresholds, score weights, cadences, limits, codecs, renderer choices, matrix modes, and recovery gates are forbidden in feature implementation code.
- **FR-033**: System MUST provide tracked environment documentation or an `.env.example` contract listing every required scene parameter, its type, default benchmark value, allowed values, and whether changing it requires YOLOE re-export, migration, rollback, or benchmark rerun.

### Key Entities *(include if feature involves data)*

- **Scene Prompt Profile**: The named set of prompted classes used for a benchmark or runtime artifact, including ordered classes and digest.
- **YOLOE Export Manifest**: The export evidence record tying checkpoint, prompt profile, exporter environment, model artifact digests, and runtime compatibility.
- **Scene Frame Summary**: A compact per-scene-frame record containing frame identity, prompt profile, object count, person count, non-ROI coverage, artifact references, and trace status.
- **Scene Object Observation**: A per-object record containing every available normalized YOLOE output for that object, including class ID/name, confidence score, bbox, mask reference, area, centroid, anchor point, shape metadata, raw-output reference when exposed, unavailable output reasons, and source profile.
- **Non-ROI Region Mask**: The union of scene regions that are not targets of person or behavior reasoning and are used to flag hallucination risk.
- **Scene Contradiction Event**: Append-only evidence that a downstream prediction overlaps a non-ROI region or lacks support from a person mask.
- **People Mismatch Event**: The per-frame comparison between YOLOE person count and student-plus-teacher count, including match state and recovery summary.
- **Scene Recovery Candidate**: An append-only missing-person candidate with source YOLOE object, matching scores, timeline identity, provisional ID, and re-pass state.
- **Spatial Relation Frame**: The ordered object set and coordinate basis used to generate distance, angle, vector, heatmap, and correlation artifacts.
- **Scene Visualization Artifact**: Human-facing map, snapshot, heatmap, or video output with render backend, payload, timing, and digest lineage.
- **Scene Artifact Manifest**: The index of masks, matrices, traces, maps, rendered media, and generated figures with paths, digests, schema version, and size.
- **Scene Artifact Access Grant**: The job-scoped authorization boundary that determines which authenticated users can retrieve scene artifacts containing classroom imagery.
- **Scene Runtime Configuration**: The `.env`/settings-backed configuration snapshot for prompt classes, thresholds, score weights, cadences, matrix modes, dense limits, artifact codecs, renderer choices, recovery gates, and benchmark toggles.

### Assumptions

- V1 is offline-first and remains disabled for live streams until a live-specific plan and evidence package exist.
- Image-space distances are pixel-normalized and do not represent real-world meters.
- Contradiction handling is flag-only in V1; it does not suppress or mutate existing detections.
- The benchmark/test default prompt profile is `classroom_roi_guard_v1`, but users and developers may change prompt classes through environment configuration if the model is re-exported and revalidated.
- Full masks and large matrices are artifact-backed rather than stored as raw relational JSON.
- V1 mask artifacts use RLE-Zstd and V1 dense matrix artifacts use compressed NPZ unless a later benchmarked plan changes the artifact contract.
- Existing embedding and tracking evidence is the first source for recovery matching when available.
- Recovery re-pass is candidate-only by default and requires a separate measured candidate before enablement.
- PixiJS WebGL is the first renderer candidate; renderer choice still requires benchmark evidence before acceptance.
- Dense full SRVL matrices are limited to at most 128 objects per frame before top-k or thresholded output is required.
- Scene artifacts containing classroom imagery are accessible only through job-scoped authenticated paths.
- Production acceptance requires the real `combined.mp4` benchmark on the production Linux server.
- YOLOE confidence scores are decision evidence, not display-only metadata; every contradiction or recovery decision using them must record the threshold, score, and source `.env` key.
- Operational values are configuration, not code constants. If a value affects detection, filtering, matching, artifact generation, rendering, benchmarking, or rollback, it belongs in `.env`/settings and in the evidence fingerprint.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: With scene features disabled, existing offline video tests and at least one production rollback proof show no scene-specific behavior change in job terminal state, existing artifacts, or existing user-visible playback.
- **SC-002**: The prompt-locked model export validation rejects 100% of stale prompt-digest, missing manifest, wrong checkpoint, and prompt-after-export cases in focused tests.
- **SC-003**: On the production `combined.mp4` benchmark, 100% of YOLOE-executed scene frames have a scene summary, prompt profile, object count, and artifact manifest entry.
- **SC-004**: At least 99% of YOLOE object observations on executed scene frames have recoverable mask artifacts or explicit unavailable reasons with artifact lineage.
- **SC-005**: People mismatch recovery reports YOLOE person count, student-plus-teacher count, matched existing count, false alarm count, missing candidate count, re-pass eligibility, unresolved count, Redis lookup latency, and candidate-only default status for 100% of mismatch frames where recovery is enabled.
- **SC-006**: In focused recovery tests, 100% of ambiguous one-to-one matching cases remain unresolved rather than force-merged.
- **SC-007**: Non-ROI guard tests produce append-only contradiction events without mutating existing detection rows in 100% of overlap-threshold cases.
- **SC-008**: SRVL distance, angle, magnitude-angle vector, optional Cartesian vector, top-k, thresholded, and heatmap-only outputs match a trusted reference on zero-object, one-object, normal, invalid-coordinate, 128-object, and above-128-object tests.
- **SC-009**: In production SRVL benchmarks, executed-frame spatial compute p95 is at or below 10 ms for the accepted representative object-count range, or the feature remains disabled with the measured bottleneck documented.
- **SC-010**: PixiJS WebGL renderer benchmark demonstrates at least 30 FPS for representative scene-map payloads with no playback freeze longer than 500 ms, or PixiJS is rejected with evidence before testing a more complex renderer.
- **SC-011**: Visualization overload tests prove stale visualization frames are dropped while analytics artifacts remain complete.
- **SC-012**: Production benchmark evidence reports live/cumulative/database FPS, latency percentiles, per-model call rates/status/input shapes, per-step/per-phase timings, GPU utilization, GPU memory, GPU power or unavailable reason, CPU/RSS, PostgreSQL timing, Redis timing, frontend metrics, artifact sizes, correctness/model agreement, mismatch recovery counts, non-ROI contradiction counts, and unavailable metric reasons.
- **SC-013**: Generated benchmark figures and figure manifests are produced from the same raw artifacts as the benchmark decision table and include source paths and digests.
- **SC-014**: API and frontend compatibility tests show existing clients/jobs still work when scene fields are absent and new clients can read scene summaries without loading full masks by default.
- **SC-015**: Artifact access tests prove unauthenticated users and authenticated users without job access cannot retrieve scene masks, snapshots, matrices, rendered scene videos, or other classroom imagery artifacts.
- **SC-016**: Runtime reconciliation proves task, queue, database, Redis, artifact, telemetry, frontend, and rendered media states converge or expose a blocking fault.
- **SC-017**: Evidence artifacts are real, immutable or digest-addressed where applicable, PostgreSQL-backed where relational state is involved, and reproducible from recorded lineage.
- **SC-018**: No hidden xfails, unsupported live enablement, SQLite fallback, untracked CI-required files, mock acceptance artifacts, or production claims based only on local runs remain for the accepted scope.
- **SC-019**: If implementation touches async jobs, vector persistence, or orchestration, the accepted scope declares per-stage deadlines, reconciler behavior, idempotency keys, error-ratio fail-closed thresholds, vector payload guards, and re-run tests before acceptance.
- **SC-020**: If future worker/thread/concurrency changes are proposed, the candidate remains hypothesis-only until a governed production benchmark proves improvement without GPU, memory, database, Redis, lifecycle, duplicate-worker, or correctness regression.
- **SC-021**: Focused YOLOE normalization tests prove that all runtime-emitted YOLOE fields, including confidence scores and masks, are either preserved in summaries/artifacts/traces or recorded with explicit unavailable reasons.
- **SC-022**: Configuration tests prove every operational prompt, threshold, score weight, cadence, limit, codec, renderer, matrix mode, and recovery gate is declared in `.env`/settings and that hardcoded feature constants cannot silently control production behavior.
