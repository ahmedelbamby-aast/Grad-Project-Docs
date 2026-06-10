# Feature Specification: Explainable Behavioral Evidence And Anomaly Scoring

**Feature Branch**: `015-xai-anomaly-score`
**Created**: 2026-06-08
**Status**: Current active feature for implementation
**Input**: User request to create a deep, production-grade implementation plan
for XAI across all deployed models, an anomaly score layer, additional derived
signals, atomic implementation cycles, strict modularity, per-cycle
benchmarking, and WebGL-rendered analytical figures. The user additionally
clarified that no anomaly/cheating/normality dataset or ground-truth method
exists; behavioral assessment must depend on per-student temporal patterns
across all valid system signals.

## Mandatory No-Ground-Truth Constraint

[no-ground-truth-doctrine.md](no-ground-truth-doctrine.md) is binding for every
requirement, cycle, contract, task, benchmark, and user-facing claim.

- No anomaly, abnormal-behavior, cheating, non-cheating, or normality model may
  be trained or fine-tuned under this plan.
- Every pretrained model consumed is FROZEN — either a production signal source
  (Class A) or a `PROBE_ONLY` representation/one-class candidate (Class B) — and
  is catalogued with its role and promotion constraint in
  [pretrained-models-registry.md](pretrained-models-registry.md).
- The pretrained registry now has a reproducible acquisition manifest plus
  download helpers and a verifier so required model packs, digests, and helper
  commands are explicit rather than tribal knowledge.
- Reviewer feedback, heuristic rules, assumed-normal history, source-model
  agreement, and current BSIL outputs are not anomaly ground truth.
- The production layer compares bounded per-student multivariate temporal
  signal windows with versioned, contamination-aware observed-pattern
  profiles.
- Controlled fixtures validate algorithm semantics and invariants; they do not
  establish real-world behavioral truth.
- No cycle may claim anomaly accuracy, precision, recall, F1, AUROC, AUPRC, or
  cheating-detection quality without a separate future governed ground-truth
  authority.

## Clarified Safety And Product Semantics

- The system does not determine that a student is cheating. It produces
  evidence-backed, uncertainty-aware `review_priority_score` records and
  non-accusatory behavioral review candidates.
- A high score means that governed signals differ materially from an accepted
  baseline and that enough reliable evidence supports human review. It is not a
  probability of misconduct, intent, or guilt.
- `within_observed_pattern` means only that a valid signal window lies inside a
  governed observed-pattern envelope. It does not mean normal, innocent, or
  non-cheating.
- `pattern_deviation` means only that a valid signal window differs from its
  governed observed-pattern envelope. It does not mean abnormal intent,
  misconduct, or cheating.
- Every explanation must distinguish model confidence, calibration quality,
  evidence coverage, anomaly surprise, identity continuity, and missingness.
- Missing, degraded, contradictory, or low-quality evidence is never converted
  to zero and never silently increases certainty.
- Deep saliency and perturbation XAI is asynchronous and on-demand. It must not
  block the live critical path or the accepted offline inference path.
- A student's score remains student-scoped. If a reviewer later approves that
  one student's evidence as a meaningful deviation, that approval may raise the
  session/classroom aggregate only; it must not mutate any peer student's
  individual score or pattern state.
- Where operators informally say "normal until proven otherwise", the governed
  persisted state remains `within_observed_pattern` or `insufficient_context`
  until valid evidence supports `pattern_deviation`.
- Where operators use the shorthand `abnormal` or `cheating` for the same
  escalation bucket, the system normalizes that input to the single governed
  review-approved deviation path; it does not create separate persisted
  "abnormal" or "cheating" states, and it still keeps every student's score
  student-scoped.
- Student-interaction-graph edges (proximity, directed/mutual gaze-toward-peer,
  relative orientation, co-movement, shared-scene association) are identity-gated
  relational context derived from existing signals. An edge or an elevated
  interaction-graph signal raises only the involved student's own review
  priority; it is never proof of collusion or cheating and never accuses the
  peer.
- The general baseline is a **General Population Baseline** — a contamination-aware,
  assumed-normal aggregate computed across many students and sessions while
  ingesting the supported videos. It is **never derived from, or equal to, a
  single student's profile**. It provides **General Boundaries**, while each
  student's own time-windowed, cold-start-aware profile provides **Local
  Boundaries**. Scoring uses both; the general baseline is never known-normal
  ground truth, and a population pattern never becomes a per-student verdict.
- No operational value is hardcoded. Every threshold, weight, envelope bound,
  geometric constant, and gate is either learned from a governed baseline or read
  from a fingerprinted configuration/`.env` value, and each carries explicit
  provenance so it stays reconstructable for XAI.
- A research/probe model becomes a production requirement only through an
  evidence-gated promotion lifecycle (`PROBE_ONLY` → `SHADOW` → `CANARY` →
  `MANDATORY`) and only as a governed signal/representation — never as a
  behavioral judge. Promotions are benchmark- and metric-driven, recorded,
  approved by a governed approver, and reversible.

## User Scenarios & Testing

### User Story 1 - Explain A Model Output (Priority: P1)

As a reviewer, I need to inspect why each model produced a result for a
particular student and time so I can distinguish strong evidence, weak
evidence, contradictions, and unavailable evidence.

**Why this priority**: An anomaly score cannot be trusted if the underlying
model outputs are not independently traceable and calibrated.

**Independent Test**: Select one observation from every active production model
route and verify that the explanation identifies the exact model route,
artifact/version, input scope, source timestamp, local/canonical identity
scope, output, calibrated confidence, reliability, evidence contributions,
contradictions, unavailable fields, and lineage digest.

**Acceptance Scenarios**:

1. **Given** a person-detector output, **When** a reviewer requests an
   explanation, **Then** the system exposes the detection score, localization,
   tracking assignment, temporal support, quality, and optional on-demand
   attribution without claiming identity or intent from the detector alone.
2. **Given** posture or gaze outputs disagree with pose kinematics, **When** the
   reviewer opens the explanation, **Then** the contradiction and both sources
   remain visible rather than one source being silently discarded.
3. **Given** a required output or attribution is unavailable, **When** the
   explanation is rendered, **Then** the system shows the unavailable reason
   and lowers or withholds explanation confidence.

---

### User Story 2 - Diagnose A Behavioral Review Candidate (Priority: P1)

As a reviewer, I need to understand why a student's behavior received a high
review priority over time, including the baseline, temporal pattern,
uncertainty, and exact score contributions.

**Why this priority**: The user requested a diagnosis of abnormal behavior
using all current model signals. This must be temporal, calibrated, and
non-accusatory.

**Independent Test**: Replay deterministic controlled signal-pattern fixtures
and real-media signal sequences containing stable, transient, sustained,
contradictory, cold-start, contaminated-profile, and missing-evidence
intervals. Verify that the score is reconstructable, deterministic, and
withheld where a valid comparison cannot be made. The fixtures validate
algorithm behavior and are not behavioral ground truth.

**Acceptance Scenarios**:

1. **Given** a transient one-frame change, **When** anomaly scoring evaluates
   the interval, **Then** temporal persistence and hysteresis prevent an
   unsupported high review priority.
2. **Given** repeated reliable deviations from a valid baseline, **When** the
   score activates, **Then** the explanation lists pattern-deviation magnitude,
   persistence, context, reliability, uncertainty, profile compatibility, and
   threshold provenance.
3. **Given** an observed-pattern profile is missing, contaminated, drifting, or
   quarantined,
   **When** scoring is requested, **Then** the score is withheld or degraded
   and the reason is visible.
4. **Given** a new student or incompatible route has insufficient valid
   history, **When** scoring is requested, **Then** the result is
   `insufficient_context` rather than a low score or normal label.
5. **Given** a valid window with both a compatible general baseline and the
   student's own profile, **When** scoring runs, **Then** the explanation lists
   `deviation_vs_self` and `deviation_vs_population` separately, references both
   snapshot versions, and the student's own profile remains the primary
   comparison.

---

### User Story 3 - Analyze Context And Cross-Model Evidence (Priority: P2)

As a reviewer, I need to see how pose, behavior models, tracking, scene objects,
spatial relations, and peer context support or contradict one another without
forcing identity or causal conclusions.

**Why this priority**: A single model can be confidently wrong. Context and
cross-model contradiction are necessary to diagnose model behavior safely.

**Independent Test**: Replay crowded, crossing, occluded, scene-contradiction,
and synchronized-behavior sequences and verify that evidence fusion remains
source-scoped, identity-gated, and reconstructable.

**Acceptance Scenarios**:

1. **Given** an unstable or unresolved identity, **When** contextual evidence is
   fused, **Then** cross-person contributions are suppressed or degraded.
2. **Given** a scene mask contradicts a person box, **When** the explanation is
   assembled, **Then** the original box and append-only contradiction evidence
   both remain visible.
3. **Given** multiple signals support a temporal review candidate, **When** the
   reviewer inspects it, **Then** the explanation graph shows each source,
   contribution, reliability, timestamp, and contradiction.

---

### User Story 4 - Request Deep XAI And Review Evidence (Priority: P2)

As an authorized reviewer, I need to request deeper model interpretation for a
selected observation and compare it with governed prototypes or similar cases
without slowing normal inference.

**Why this priority**: Saliency, perturbation, and exemplar methods can aid
diagnosis, but they are expensive and require faithfulness checks.

**Independent Test**: Request deep XAI for representative detector, behavior,
pose, ReID, and scene observations and verify asynchronous completion,
bounded resource use, artifact access control, faithfulness evaluation, and
non-blocking inference.

**Acceptance Scenarios**:

1. **Given** an eligible completed observation, **When** deep XAI is requested,
   **Then** it runs on an isolated bounded queue and returns a digest-addressed
   artifact or an explicit unavailable reason.
2. **Given** a saliency method fails sanity or perturbation-fidelity checks,
   **When** the result is reviewed, **Then** it is labeled unreliable and cannot
   support a production-valid explanation.
3. **Given** a reviewer submits feedback, **When** it is accepted, **Then** it
   becomes governed evaluation evidence and does not directly mutate production
   thresholds, profiles, baselines, or model weights and is never treated as
   anomaly ground truth or a training target.

---

### User Story 5 - Explore Explanations In A Responsive WebGL Workbench (Priority: P2)

As a frontend user, I need large time series, contribution plots, heatmaps,
distance matrices, attribution overlays, and explanation graphs to remain
responsive while a job is still processing.

**Why this priority**: The user requires WebGL rendering, real-time updates,
high throughput, low latency, and stable interaction for analytical figures.

**Independent Test**: Stream representative small and large explanation
payloads into the workbench while toggling series, panning, zooming,
auto-following, resizing, downloading, and switching jobs; verify that no
section crashes, no WebGL context is leaked, and rendering remains within the
declared frame-time and memory budgets.

**Acceptance Scenarios**:

1. **Given** a running job, **When** new score and contribution samples arrive,
   **Then** visible WebGL plots append incrementally and auto-follow without
   rebuilding the complete dataset.
2. **Given** a large matrix or long series, **When** the user pans or zooms,
   **Then** viewport-aware downsampling/tiling keeps labels, cells, and plots
   aligned without overflow.
3. **Given** WebGL is unavailable or context-lost, **When** the workbench
   degrades, **Then** numeric/tabular evidence remains available and the figure
   is marked unavailable; a Canvas2D figure is not presented as accepted WebGL
   rendering.

---

### User Story 6 - Prove Production Performance And Scientific Integrity (Priority: P3)

As a maintainer or scientific reviewer, I need every atomic cycle to include
tests, production benchmarks, generated figures, manifests, rollback proof,
and explicit limits so no XAI or anomaly claim ships from placeholders or
local-only evidence.

**Why this priority**: Explainability can create false confidence. Production
and scientific acceptance require independent evidence.

**Independent Test**: Audit each cycle package and verify that implementation,
quality, performance, stability, scientific evaluation, WebGL rendering,
security, lineage, and rollback gates are all satisfied from authoritative
evidence.

**Acceptance Scenarios**:

1. **Given** an atomic cycle claims acceptance, **When** its evidence is
   audited, **Then** the benchmark ledger, raw artifacts, generated figures,
   manifest digests, Figure Planner output, Figure Implementer output, and
   rollback proof all resolve.
2. **Given** an optimization improves one metric but violates correctness,
   calibration, identity, stability, or latency gates, **When** the cycle is
   decided, **Then** it is not accepted.
3. **Given** a component probe or local test passes, **When** production
   acceptance is considered, **Then** the result remains probe-only until a
   completed stride-1 native Linux RTX 5090 end-to-end benchmark exists.
4. **Given** an operational value drives a gate, weight, or geometric constant,
   **When** acceptance is audited, **Then** the value resolves to a learned
   baseline reference or a fingerprinted configuration/`.env` key with a
   `learned` or `configured` provenance tag, and no hardcoded operational
   constant is present.

---

### User Story 7 - Inspect The Student-Interaction Graph (Priority: P2)

As a reviewer, I need to see how students relate to one another over time — who
is near whom, who is oriented or looking toward whom, and how those relations
change — as both a live WebGL graph and as per-student signals that feed the
review-priority score, without any edge implying collusion or cheating.

**Why this priority**: Many review-worthy moments are relational (sustained
directed attention toward a neighbor, abnormal proximity dynamics). These must
be derived from existing signals deterministically, identity-gated, rendered
responsively, and kept non-accusatory and student-scoped.

**Independent Test**: Replay crowded, crossing, occluded, and
synchronized-behavior sequences and verify that the interaction graph is
reconstructable from the referenced relational signals, that ambiguous-identity
edges stay unresolved, that per-student graph features are bounded and feed the
observed-pattern profile, and that the node-link graph and adjacency matrix
render in WebGL2 within budget with a numeric/tabular fallback.

**Acceptance Scenarios**:

1. **Given** two students with sustained directed gaze-toward-peer and close
   proximity, **When** the interaction graph is built, **Then** a weighted edge
   with persistence and identity-confidence appears and the per-student
   directed-attention/proximity features are exposed, without labeling either
   student as cheating or colluding.
2. **Given** an unresolved or ambiguous identity, **When** a candidate edge
   would connect it, **Then** the edge is marked unresolved and excluded from
   the student's valid graph features rather than fabricated.
3. **Given** a graph with more nodes/edges than the configured bound or a lost
   WebGL context, **When** the workbench renders, **Then** bounded LOD/tiling
   and context-loss recovery keep it interactive, or a numeric/tabular fallback
   is shown and the figure is marked unavailable.
4. **Given** a running live or offline job, **When** new interaction-graph frames
   arrive, **Then** the dedicated real-time graph plot updates incrementally and
   continuously alongside the other plots without rebuilding the full graph,
   stays within the live frame-time/update-latency budgets, and drops only stale
   renderer updates under backpressure.

## Edge Cases

- A model is confident but miscalibrated.
- A score is high because one unreliable signal dominates.
- All contributing signals are missing, degraded, or contradictory.
- A student's local tracker ID changes after occlusion or reconnect.
- Independent runs reuse the same numeric local tracker ID for different people.
- An observed-pattern profile contains high-deviation or reviewer-disputed
  periods.
- No anomaly-behavior dataset or accepted ground-truth method exists.
- A new identity has too little valid history to form a pattern profile.
- A consistently unusual session attempts to redefine itself as normal.
- A contaminated, disputed, or high-deviation window attempts to update a
  pattern profile.
- A new model route or artifact version appears without an explainer adapter.
- A route changes between scoring and explanation generation.
- A model emits dense outputs too large to persist on every frame.
- A saliency map is visually plausible but fails model-parameter randomization.
- An on-demand XAI task retries after partial artifact creation.
- A WebGL context is lost while multiple charts are visible.
- A time series contains millions of samples or out-of-order timestamps.
- A matrix contains zero, one, more than 128, or thousands of objects.
- A user lacks access to the source job or classroom imagery.
- The production anomaly/BSIL tables exist but remain empty.
- Runtime settings differ from the accepted benchmark route.
- A reviewer assessment conflicts with the current pattern profile or score.
- A live stream runs indefinitely and must not accumulate unbounded state.
- A score is requested across a reconnect gap or invalid temporal window.
- A pairwise interaction edge depends on an ambiguous or unresolved identity.
- An interaction graph contains more students or edges than the configured
  node/edge bound.
- A mutual-gaze or directed-attention edge is implied while the peer is occluded,
  missing, or has low pose/gaze quality.
- Two students share a desk or scene object but never actually interact.
- A learned graph model is proposed as production scoring authority.
- The live interaction graph receives updates faster than its render budget.
- A WebGL context is lost while the real-time interaction graph is streaming.
- The live interaction graph runs for an indefinite stream and must stay bounded.
- The supported-video corpus is too small to form a valid general baseline
  (population cold-start).
- The supported-video corpus is contaminated or drifts at population scale.
- The general baseline and the student's own profile disagree for the same
  window.
- A general-baseline tier (age-band, scene type, camera) has no compatible match.
- A required configuration/`.env` value is missing, unfingerprinted, or changed
  between scoring and reconciliation.
- A hardcoded operational constant is introduced into the code path.
- A probe is proposed for production-mandatory status without a benchmark,
  computed metrics, model card, governed approver, or rollback proof.
- A shadow model meets latency but degrades serving-output distribution quality.
- A promoted model later drifts and must automatically roll back to the prior
  stage.
- Someone proposes promoting a learned model into a behavioral decision authority
  or claiming anomaly accuracy/AUROC for it.

## Mandatory Runtime Scenarios

- **Live Stream Scenario**: Only the bounded fast-XAI and anomaly-score path may
  run inline. State is per-camera/per-identity bounded by time and count, queue
  overflow drops oldest analytical updates without blocking decode, reconnect
  gaps invalidate continuity, and no offline-only deep XAI runs automatically.
  Deep XAI may operate only on explicitly captured bounded observations after
  authorization and may not mutate prior live evidence.
- **Offline Video Processing Scenario**: The fast path may emit evidence and
  scores incrementally while a job runs. Deep XAI and full-video analytical
  reports remain asynchronous and optional. Acceptance benchmarks use
  frame-stride `1`, `new-attempt` replay policy, real production media, and the
  active Triton-only route.
- **Frontend-Backend Wiring**: Versioned REST/WebSocket contracts expose
  incremental evidence envelopes, anomaly-score records, explanation status,
  renderer status, and truth states `unknown`, `unavailable`, `degraded`,
  `valid`, `suppressed`, and `invalidated`. Payloads support bounded JSON
  fallback and a binary typed-array path for high-volume WebGL data.
- **Backend-Inference Wiring**: Production model interpretation references the
  immutable active route snapshot and exact artifact/version. Governed model
  inference remains Triton-only. Explainers must not silently invoke a local
  model fallback and label it production-valid.
- **Temporal/Identity/Pose Authority**: Every record carries source,
  ingest/queue/inference/persistence timestamps; source-scoped local identity;
  canonical identity when governed; ReID/continuity provenance; pose stream and
  quality; feature/ontology version; and invalid-window reasons.
- **Independent-Run/Sharded Identity Evaluation**: Local labels are opaque.
  Comparisons use observation matching and deterministic globally one-to-one
  assignment. Detection/localization, association, fragmentation, merges,
  unresolved cases, and proxy-ground-truth limitations are separate.
- **Deployment Boundary**: Local Windows checks are contract evidence only.
  Production decisions require native Linux RTX 5090, PostgreSQL, Redis/Celery,
  one active Triton profile, no Docker, no sudo, committed SHA, environment
  fingerprint, model-route snapshot, and durable evidence.
- **Concurrency/Worker Scaling Boundary**: No worker/thread/concurrency change
  is accepted implicitly. Any proposed isolation queue or concurrency change
  requires a separate baseline/candidate topology, resource budget,
  duplicate-worker check, production benchmark, and rollback.
- **Replay Policy**: Acceptance uses `new-attempt`. Failed, partial, reused,
  sampled, or reduced-media runs do not establish acceptance.
- **Benchmark Decision Evidence**: Each cycle records stride-1 baseline and
  candidate jobs, exact media, SHA, configuration delta, latency percentiles,
  throughput, FPS, per-model RTT/call rate/status/input shape, phase timing,
  GPU/VRAM, CPU/RSS, PostgreSQL/Redis, correctness, calibration, explanation
  fidelity/stability, identity, frontend frame time, resource budgets,
  unavailable reasons, causal interpretation, remaining bottleneck, generated
  figures, manifests, and rollback proof.
- **No-Ground-Truth Evaluation Boundary**: Anomaly-layer acceptance uses exact
  reconstruction, deterministic replay, controlled pattern fixtures,
  metamorphic/invariant checks, contamination/cold-start/drift behavior,
  real-media stability, performance, and rollback. It does not claim
  behavioral classification accuracy. Reviewer feedback is explicitly
  non-ground-truth evaluation evidence.
- **Runtime Reconciliation**: Route snapshot, task, queue, database, artifact,
  telemetry, frontend, and review states must converge. Mismatch blocks
  production-valid explanations or scores.
- **Evidence Lineage**: Evidence is immutable after publication,
  digest-addressed, access-controlled, and distinguishes mock/real, local/prod,
  CPU/GPU, synthetic/real-media, fallback/canonical, and fast/deep-XAI paths.

## Requirements

### Functional Requirements

- **FR-001**: The system MUST represent all explanation inputs and outputs
  through one versioned evidence envelope with explicit lineage, truth state,
  quality, missingness, and unavailable reasons.
- **FR-002**: The system MUST capture an immutable model-route snapshot and
  fingerprint before producing a production-valid explanation or score.
- **FR-003**: The system MUST provide a registered explainer adapter for every
  active production model route and MUST reject unregistered routes from
  production-valid explanations.
- **FR-004**: The system MUST distinguish fast inline explanations from
  asynchronous deep explanations and record the path used.
- **FR-005**: The fast path MUST provide deterministic score/output
  decomposition, calibrated confidence, input/output quality, temporal support,
  contradictions, counterfactual threshold deltas, and missingness.
- **FR-006**: The deep path MUST support model-appropriate attribution,
  perturbation, exemplar, or prototype methods and MUST record method,
  parameters, cost, fidelity, stability, and sanity-check results.
- **FR-007**: The system MUST provide model-specific explanation strategies for
  person detection, posture, horizontal/vertical/depth gaze, behavior ensemble,
  RTMPose, OSNet ReID, YOLOE scene segmentation, tracking, pose kinematics,
  SRVL, BSIL rules, and anomaly scoring.
- **FR-008**: The system MUST preserve raw model scores or compact governed
  summaries required for calibration and explanation, subject to configured
  sampling, retention, privacy, and payload budgets.
- **FR-009**: The system MUST derive the governed signal catalog in
  `signal-catalog.md` and expose every enabled signal's definition, unit,
  source, validity rule, and explanation strategy.
- **FR-010**: The system MUST compute anomaly evidence from valid temporal
  windows rather than isolated frames unless a frame-level primitive is
  explicitly labeled advisory.
- **FR-011**: The system MUST compute a non-accusatory
  `review_priority_score`, not a cheating probability or misconduct verdict.
- **FR-012**: Every review priority MUST include reconstructable contributions,
  coverage, reliability, uncertainty, pattern-profile/threshold provenance,
  contradictions, and score-withholding reasons.
- **FR-013**: The system MUST support person-, session-, camera-, scene-, and
  population-context observed-pattern profiles while preventing contaminated
  or drifting profiles from silently changing decisions.
- **FR-014**: The system MUST calibrate existing source-model confidence only
  where task-appropriate governed held-out evidence exists and MUST otherwise
  report calibration unavailable; raw confidence is not accepted as certainty.
- **FR-015**: The system MUST support uncertainty-aware or conformal outputs
  where assumptions and calibration evidence are valid and MUST label them
  unavailable otherwise; distributional coverage MUST NOT be described as
  behavioral correctness.
- **FR-016**: Cross-model fusion MUST preserve each source, contribution,
  reliability, contradiction, and unavailable reason; it MUST NOT hide
  disagreement behind one aggregate score.
- **FR-017**: Cross-person or peer-context evidence MUST be identity-gated and
  must remain unresolved when identity evidence is ambiguous.
- **FR-018**: Review feedback MUST be governed evaluation evidence and MUST NOT
  be called ground truth, become a training/fine-tuning target, update a
  pattern profile, or directly mutate production thresholds, baselines, or
  model weights.
- **FR-019**: All XAI artifacts containing classroom imagery or identity
  evidence MUST be job-scoped, authenticated, audited, and retention-governed.
- **FR-020**: Deep XAI tasks MUST be idempotent, bounded, deadline-controlled,
  reconciled, and isolated from critical inference queues.
- **FR-021**: Live fast-XAI state MUST be bounded per camera/identity and must
  tolerate indefinite streams without unbounded growth.
- **FR-022**: Every analytical figure, plot, time-series plot, matrix, heatmap,
  contribution chart, and explanation graph MUST use the shared WebGL2
  renderer for accepted production rendering.
- **FR-023**: WebGL rendering MUST use persistent shared contexts, typed arrays,
  incremental append/update, viewport-aware downsampling/tiling, context-loss
  recovery, and visibility-paused rendering.
- **FR-024**: WebGL-unavailable states MUST retain accessible numeric/tabular
  evidence and explicitly mark the figure unavailable; Canvas2D is not an
  accepted figure-rendering substitute.
- **FR-025**: Every figure MUST support a download action with lineage metadata
  and must not require rebuilding the complete live data stream on the main
  thread.
- **FR-026**: The implementation MUST use modular interfaces and registries,
  reuse existing behavior/anomaly/pipeline/telemetry boundaries, and avoid a
  parallel anomaly service or duplicated model-specific orchestration.
- **FR-027**: Operational thresholds, method choices, sampling, retention,
  queue bounds, score weights, rendering budgets, and rollout gates MUST be
  configuration-governed and fingerprinted.
- **FR-028**: Every atomic cycle MUST include implementation, tests, local
  validation, production benchmark, Figure Planner output, Figure Implementer
  output, manifest/digests, documentation, rollback, and ledger entry.
- **FR-029**: A cycle MUST be independently reversible and MUST NOT combine two
  causal optimization variables in one acceptance decision.
- **FR-030**: Production acceptance MUST fail on placeholder artifacts,
  empty/unpopulated required layers, hidden fallbacks, stale route snapshots,
  missing metrics, unresolved reconciliation, or unsupported accusation terms.
- **FR-031**: The current plan MUST NOT train or fine-tune an anomaly,
  abnormal-behavior, cheating, non-cheating, or normality model and MUST NOT
  use reviewer feedback, heuristic outputs, assumed-normal history, source
  model agreement, or BSIL output as anomaly ground truth.
- **FR-032**: The anomaly layer MUST derive bounded per-student multivariate
  temporal pattern windows from all enabled valid signals and compare them
  against versioned, contamination-aware observed-pattern profiles with
  explicit cold-start, drift, quarantine, and incompatibility states.
- **FR-033**: Persisted and user-facing behavioral states MUST use
  `within_observed_pattern`, `pattern_deviation`, `insufficient_context`, or
  `withheld` semantics and MUST NOT equate those states with cheating,
  non-cheating, normality, abnormal intent, innocence, or guilt.
- **FR-034**: Until a given student's own valid evidence supports a governed
  deviation, that student's persisted and previewed state MUST remain
  `within_observed_pattern`, `insufficient_context`, or `withheld`. A
  reviewer-approved deviation for one student MAY lift only a separate
  session/classroom aggregate term and MUST NOT mutate any peer student's
  individual score, contribution set, or pattern state.
- **FR-035**: The system MUST derive a bounded, identity-gated
  student-interaction graph (nodes = tracked students/teacher; edges =
  deterministic relational evidence such as proximity, directed/mutual
  gaze-toward-peer, relative body orientation, co-movement, and
  shared-scene-object association) from existing valid relational signals
  (person detection, pose kinematics, gaze, the COMBINED head, YOLOE scene, and
  SRVL), and MUST expose per-student graph-derived features (interaction degree,
  weighted degree, mutual-gaze dwell, directed-attention duration, edge
  persistence, local clustering/centrality proxy, neighbor churn, and dyad
  strength) as governed signals that feed the per-student observed-pattern
  profiles. Any learned graph model (GNN, ST-GCN-family, or dynamic-graph
  anomaly method) remains `PROBE_ONLY` per the pretrained-models registry and
  MUST NOT drive a production score.
- **FR-036**: Interaction-graph node-link views, adjacency/interaction matrices,
  and dyad timelines MUST render through the shared WebGL2 renderer under the
  same persistent-context, incremental-append, viewport-LOD/tiling,
  context-loss-recovery, and numeric/tabular-fallback rules; edges whose
  identity evidence is ambiguous MUST be shown unresolved rather than asserted,
  and no graph edge or graph signal may be presented as proof of cheating,
  intent, or collusion.
- **FR-037**: The system MUST learn a versioned, contamination-aware **general
  baseline** (population tier plus context tiers such as age-band, scene type,
  camera, and session) by ingesting all supported videos, using the same
  deterministic robust-statistics, cold-start, quarantine, drift, and
  append-only machinery as per-student profiles. The general baseline is
  assumed-normal/semi-supervised aggregation; it MUST NOT be called known-normal
  ground truth and MUST NOT be a trained anomaly/cheating/normality model.
- **FR-038**: Scoring MUST perform **dual comparison** — each valid window is
  compared against BOTH the compatible general baseline AND the student's own
  observed-pattern profile. Contributions MUST expose `deviation_vs_self` and
  `deviation_vs_population` separately; the student's own profile is the primary
  comparison and the general baseline is contextual support; the score is
  withheld when a required tier is missing, incompatible, cold-start, or
  quarantined, and tier disagreement remains visible rather than hidden.
- **FR-039**: No operational value may be hardcoded. Every threshold, weight,
  envelope/quantile bound, geometric constant (e.g., gaze-cone angle),
  persistence/hysteresis window, score weight, and gate MUST be either **learned**
  from a governed baseline snapshot or sourced from a **fingerprinted
  configuration/`.env`** value, and MUST carry an explicit provenance tag
  (`learned` or `configured`) with its source reference. Static and runtime
  checks MUST fail on any magic-number operational constant, and because XAI is
  the top priority, every such value MUST be reconstructable to its provenance.
- **FR-040**: A `PROBE_ONLY` model MUST advance to a production-mandatory role
  only through the governed promotion lifecycle (`PROBE_ONLY` → `SHADOW` →
  `CANARY` → `MANDATORY`), where each transition requires a recorded evidence
  packet: a native RTX 5090 stride-1 benchmark; computed serving metrics
  (latency/throughput/resource SLOs, equal-or-better serving distribution versus
  the incumbent, minimum shadow duration); reproducibility/determinism; drift
  monitoring; XAI fidelity/sanity where applicable; a model card; security and
  retention compliance; a benchmark-ledger entry; governed-approver sign-off; and
  proven automated rollback. The gates are a bounded minimum bar — once the
  recorded evidence meets them, promotion is permitted.
- **FR-041**: Promotion MUST NOT convert a learned model into a behavioral,
  cheating, or normality decision authority, and MUST NOT report anomaly
  accuracy, precision, recall, F1, AUROC, or AUPRC. A probe MAY be promoted only
  into a governed signal/representation role; any behavioral-decision promotion
  requires a separate governed ground-truth program. `PROBE_ONLY`/`SHADOW`/`CANARY`
  outputs MUST NOT enter a production `review_priority_score` or `pattern_state`.
  Because no ground-truth labels exist, every promotable candidate MUST be
  unsupervised or self-supervised (e.g. skeleton normalizing-flow VAD, one-class
  boundaries, dynamic-graph anomaly), and its promotion certifies serving quality
  as a signal, never a labeled accuracy.
- **FR-042**: The general baseline MUST be a population/context aggregate computed
  across many students and sessions and MUST NOT be derived from, or equated with,
  any single student's profile. It provides **General Boundaries** for comparison,
  while each student's own time-windowed, cold-start-aware profile provides **Local
  Boundaries**. Scoring MUST reference both, and the General Boundaries MUST
  additionally support a **general classroom-level deviation** signal that is
  distinct from, and never substituted for, any individual student's deviation.
- **FR-043**: In addition to the already-required analytical plots (time series,
  contribution charts, matrices/heatmaps, scene maps, and explanation graphs), the
  system MUST provide a dedicated **live, real-time, continuously-updating
  student-interaction graph plot** that visualizes students as nodes and their
  relations (proximity, directed/mutual gaze-toward-peer, orientation, co-movement,
  shared-scene) as edges. It MUST update **incrementally as each new frame/window
  arrives** — driven by the `xai.interaction_graph.appended` WebSocket event and
  shared-WebGL2 incremental append — during both live streaming and offline
  processing, within the declared live frame-time and update-latency budgets, with
  bounded state, latest-frame-wins under backpressure, context-loss recovery, and a
  numeric/tabular fallback. This live graph **complements, and does not replace,**
  the existing plots, and remains identity-gated and non-accusatory (it never
  asserts collusion or cheating).

### Key Entities

- **XAI Model Route Snapshot**: Immutable active route/artifact/config truth.
- **XAI Signal Definition**: Governed catalog entry for one raw or derived
  signal.
- **XAI Evidence Envelope**: Versioned source evidence plus lineage,
  missingness, quality, and explanation payload.
- **XAI Attribution Artifact**: Digest-addressed deep-XAI output and evaluation.
- **XAI Explanation Record**: Human-readable and machine-readable explanation.
- **Anomaly Score Record**: Non-accusatory review-priority result.
- **Anomaly Score Contribution**: Per-signal or per-context contribution.
- **Observed Pattern Profile Snapshot**: Versioned contamination-aware summary
  of valid per-student multivariate signal history; never known-normal truth.
- **Signal Pattern Window**: Bounded temporal feature vector and validity
  evidence compared with an observed-pattern profile.
- **Calibration Snapshot**: Versioned confidence calibration evidence.
- **Conformal Calibration Snapshot**: Versioned uncertainty calibration and
  assumptions.
- **Explanation Evaluation Record**: Fidelity, stability, sanity, and latency
  evaluation.
- **XAI Reviewer Assessment**: Governed non-ground-truth reviewer assessment and
  evidence reference.
- **Student Interaction Graph Frame**: Bounded, identity-gated per-frame/per-window
  graph of student relational evidence plus per-student graph-derived features;
  relational context only, never collusion proof.
- **Student Interaction Edge**: One directed or undirected pairwise relational
  evidence item (edge type, weight, persistence, identity confidence) between two
  tracked students, with explicit unresolved state when identity is ambiguous.
- **General Baseline Snapshot**: Versioned, contamination-aware population/context-tier
  observed-pattern aggregate learned by corpus ingestion; assumed-normal, never
  known-normal ground truth.
- **Parameter Provenance Record**: Binds each operational value to its source — a
  `learned` baseline reference or a `configured` config/`.env` key plus fingerprint
  — proving no hardcoded operational constant is used.
- **Model Promotion Record**: Immutable record of one model stage transition
  (`PROBE_ONLY`/`SHADOW`/`CANARY`/`MANDATORY`), the benchmark, computed serving
  metrics, model card, governed approver, decision, and rollback proof that
  justified it.

## Success Criteria

### Measurable Outcomes

- **SC-001**: Every active production model route has a registered fast
  explanation adapter and a documented deep-XAI eligibility decision.
- **SC-002**: A reviewer can reconstruct every displayed review-priority score
  from persisted contributions and the referenced observed-pattern
  profile/threshold snapshot.
- **SC-003**: No production-valid score is emitted when required evidence
  coverage, route fingerprint, temporal authority, identity continuity, or
  observed-pattern profile validity fails its configured gate.
- **SC-004**: Where task-appropriate source-model held-out evidence exists,
  calibration reports include reliability diagrams, expected calibration
  error, Brier score or task-appropriate alternatives, sample counts,
  confidence intervals, and subgroup limitations; otherwise calibration is
  explicitly unavailable.
- **SC-005**: Explanation evaluation reports fidelity, stability, sanity-check,
  and unavailable reasons; unfaithful methods cannot support accepted
  explanations.
- **SC-006**: Fast-XAI plus scoring stays within the per-cycle accepted latency,
  throughput, memory, database, Redis, and correctness budgets established by
  a completed production benchmark.
- **SC-007**: Deep XAI does not reduce critical inference throughput or violate
  latency/stability gates because it is isolated, bounded, and optional.
- **SC-008**: Live state remains bounded during a documented soak test and
  reconnect gaps do not fabricate continuity.
- **SC-009**: Offline stride-1 end-to-end production benchmarks complete with
  required raw metrics, figures, manifests, and rollback proof for every
  accepted cycle.
- **SC-010**: WebGL workbench interaction meets declared frame-time, memory,
  context-count, update-latency, and context-loss-recovery budgets for small,
  representative, and stress payloads.
- **SC-011**: All figures use WebGL2 in accepted production rendering, and
  unavailable WebGL is surfaced without a mislabeled Canvas2D substitute.
- **SC-012**: All persisted XAI/anomaly records are versioned, idempotent,
  PostgreSQL-backed where relational, digest-addressed where artifact-backed,
  access-controlled, and reconstructable.
- **SC-013**: Review feedback changes no production threshold, baseline, or
  model without a separate governed promotion cycle and benchmark.
- **SC-014**: No user-facing or persisted production-valid output states that a
  student cheated, intended misconduct, or is guilty.
- **SC-015**: Runtime reconciliation proves route, task, queue, database,
  artifact, telemetry, frontend, and review state convergence or exposes a
  blocking fault.
- **SC-016**: Every atomic cycle has exactly one declared Figure Planner and one
  declared Figure Implementer before a benchmark decision.
- **SC-017**: Every benchmark is recorded in
  `docs/BENCHMARK_RESULTS_LEDGER.md`, including refused and probe-only runs.
- **SC-018**: Final production canary acceptance is tied to one committed SHA,
  environment fingerprint, model-route snapshot, source-model calibration
  state/snapshot, observed-pattern profile snapshot, benchmark jobs, rollback
  proof, and immutable evidence manifest.
- **SC-019**: Static checks, runtime reconciliation, and final acceptance prove
  that no anomaly model training/fine-tuning target exists and that reviewer
  feedback and observed history cannot be consumed as anomaly ground truth.
- **SC-020**: Controlled pattern fixtures and metamorphic tests prove exact,
  deterministic behavior for persistence, missingness, cold start,
  contamination, drift, quarantine, identity gaps, contradictions, and
  counterfactual changes without claiming real-world behavioral accuracy.
- **SC-021**: Every persisted and displayed anomaly state uses the required
  observed-pattern vocabulary and exposes the no-ground-truth knowledge limit.
- **SC-022**: Aggregate previews and persisted summaries prove that
  reviewer-approved deviation lifts only the separate session/classroom
  aggregate while each peer student's individual score and pattern state remain
  unchanged.
- **SC-023**: Per-student interaction-graph features are deterministic,
  reconstructable from the referenced source relational signals, identity-gated,
  and bounded; ambiguous-identity edges remain unresolved and never fabricate
  peer interactions or peer scores.
- **SC-024**: Interaction-graph node-link and adjacency renders use WebGL2
  within the declared frame-time, context-count, and update-latency budgets,
  expose a numeric/tabular fallback when WebGL is unavailable, and any learned
  graph model used for research comparison is recorded `PROBE_ONLY` with no
  anomaly accuracy, AUROC, or collusion-detection claim.
- **SC-025**: A static no-hardcoding verifier plus runtime reconciliation prove
  that every operational value resolves to a learned baseline reference or a
  fingerprinted configuration/`.env` key with a `learned` or `configured`
  provenance tag; any magic-number operational constant fails acceptance.
- **SC-026**: Every score exposes `deviation_vs_self` and `deviation_vs_population`
  (or an explicit unavailable reason), reconstructable from the referenced
  student-profile and general-baseline snapshots; the general baseline is
  versioned, contamination-aware, and never labeled ground truth.
- **SC-027**: Every model promotion records its from/to stage, the computed
  benchmark and serving metrics and evidence references that justified it, the
  governed approver, and the rollback proof; a promotion missing that packet
  fails acceptance, and a rollback restores the prior stage.
- **SC-028**: No promotion produces or claims anomaly/behavioral accuracy or
  AUROC; a promoted learned model serves only as a governed signal/representation,
  and `PROBE_ONLY`/`SHADOW`/`CANARY` outputs never enter a production score or
  pattern state.
- **SC-029**: Evidence proves the general baseline is population-derived (built
  from at least the configured minimum distinct students/sessions, never a single
  student), that General Boundaries and Local Boundaries are separately referenced
  in each score, and that a classroom-level deviation derived from General
  Boundaries never becomes a per-student verdict.
- **SC-030**: A documented live and offline soak proves the real-time
  student-interaction graph plot updates incrementally within the declared
  frame-time and update-latency budgets alongside the other live plots, recovers
  from context loss, keeps bounded state across an indefinite stream, drops only
  stale renderer updates under backpressure, and never leaks WebGL contexts.

## Assumptions And Dependencies

- Existing `apps.behavior` remains the owner of semantic, temporal, episode,
  interaction, evidence, and lineage behavior.
- Existing `apps.anomalies` remains the owner of review-priority scoring,
  baselines, calibration, thresholds, drift, and review-label governance.
- No anomaly/cheating/normality dataset or accepted behavioral ground-truth
  method exists. Cycle 015 is restricted to deterministic, reconstructable
  observed-signal pattern analysis as defined in
  `no-ground-truth-doctrine.md`.
- Existing `apps.pipeline` remains the owner of model-route and inference
  adapter integration, but anomaly code does not import pipeline internals.
- Existing `apps.telemetry` remains the metrics authority.
- Existing `apps.video_analysis` remains the owner of frames, detections,
  tracks, pose kinematics, scene observations, and SRVL records.
- Existing PostgreSQL, Redis, Celery, Triton, React/Vite, PixiJS, and WebGL2
  stack is retained unless a later measured cycle proves a replacement.
- The point-in-time production inventory is recorded in
  `production-inventory-20260608.md`; it is not a timeless claim.
