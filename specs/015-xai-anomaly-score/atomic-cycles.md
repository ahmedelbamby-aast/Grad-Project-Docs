# Atomic Implementation Cycles

**Date**: 2026-06-08
**Rule**: Each cycle is an indivisible acceptance unit. Tasks inside a cycle may
run in parallel, but the cycle has one causal hypothesis, one rollback boundary,
and one production decision. No cycle may claim acceptance from partial tasks.

## Universal Cycle Evidence Packet

Every cycle must create:

- `docs/xai_anomaly/cycle_015_<id>_investigation.md`;
- `docs/xai_anomaly/cycle_015_<id>_results.md`;
- raw JSON/CSV/log artifacts under a cycle-specific production evidence root;
- a figure plan naming exactly one Figure Planner;
- a figure implementation claim naming exactly one separate Figure Implementer;
- generated figures and `figure_manifest.json` with input paths and digests;
- a row in `docs/BENCHMARK_RESULTS_LEDGER.md`;
- rollback command, rollback execution, and post-rollback reconciliation;
- explicit streaming compatibility;
- baseline/candidate/best-comparable metric deltas;
- unavailable metrics with reasons, never hidden or plotted as zero.
- explicit proof that no anomaly model training/fine-tuning target or
  behavioral-ground-truth claim was introduced.

## Cycle 015.0 - Runtime Truth, Route Snapshot, And BSIL Activation

**Indivisible capability**: Establish one trustworthy runtime/source baseline.
Route immutability, mapping authority, and BSIL population cannot be accepted
separately because all later explanations and scores depend on their combined
truth.

**Dependencies**: None. This blocks all later cycles.
**Streaming compatibility**: `stream-safe`.

**Implementation scope**:

- add immutable model-route/config/artifact snapshot and digest;
- remove hidden mutation of the shared default route table;
- reconcile behavior label mappings and persisted presentation names;
- replace accusatory rule-engine outputs with non-accusatory review-candidate
  vocabulary or explicitly deprecate the path;
- activate and reconcile BSIL semantic/temporal/lineage production rows behind
  flags;
- record the absence of anomaly behavioral ground truth and block any anomaly
  training/fine-tuning path;
- prove current route, queue, DB, telemetry, and frontend state convergence;
- retire or mark placeholder lifecycle explainability/visualization helpers.

**Benchmark hypothesis**: Runtime truth capture and BSIL activation fit within
declared latency/throughput/storage budgets without changing source model
outputs.

**Acceptance gates**:

- nonzero, reconciled BSIL/lineage rows on a valid production job;
- immutable route digest matches the exact active models/config;
- mapping parity and no accusation-language gate pass;
- source detection/track/pose/scene output parity;
- rollback returns BSIL/XAI additions to disabled with original path intact.

**Required figures**: route/config diff, BSIL row progression, phase overhead,
DB/Redis/resource delta, output parity, unavailable-state summary.

## Cycle 015.1 - Versioned Evidence Envelope And Signal Registry

**Indivisible capability**: One governed evidence contract and one registry.
Adapters cannot be accepted before all sources share lineage, missingness,
quality, and schema semantics.

**Dependencies**: 015.0.
**Streaming compatibility**: `stream-safe`.

**Implementation scope**:

- add `EvidenceEnvelope`, `SignalDefinition`, `SignalProvider`, and registry;
- register every current raw/derived signal from `signal-catalog.md`;
- register the bounded per-student signal-pattern feature schema and explicit
  non-ground-truth knowledge limits;
- enforce schema/version/unit/validity/missingness/retention definitions;
- add idempotent persistence and explicit serializers;
- add payload/vector size guards and route/schema compatibility checks.

**Benchmark hypothesis**: Envelope normalization/persistence is bounded and
does not cause unacceptable DB/Redis or critical-path overhead.

**Acceptance gates**: Complete active-signal registry, contract tests, explicit
missingness, idempotent replay, payload guards, production row/artifact
reconciliation, output parity, and rollback.

**Required figures**: evidence coverage, envelope size distribution,
normalization latency, DB write cost, missingness distribution, signal registry
coverage.

## Cycle 015.2 - Per-Model Calibration And Reliability

**Indivisible capability**: Versioned calibration snapshots plus reliability
evaluation. Calibrated outputs are meaningless without the paired evaluation
and route compatibility.

**Dependencies**: 015.1.
**Streaming compatibility**: `stream-safe` for lookup; fitting is `offline-only`.

**Implementation scope**:

- add calibration snapshot contracts and artifact lineage;
- use task-appropriate held-out evidence only for existing source-model
  calibration where such evidence exists;
- fit/evaluate model-appropriate source-model calibrators or explicitly mark
  calibration unavailable;
- expose calibrated confidence, reliability, age, route compatibility, and
  unavailable reasons;
- block stale/incompatible/underpowered calibration.

**Benchmark hypothesis**: Calibration lookup adds negligible bounded overhead
and materially improves calibration measures without output-quality regression.

**Acceptance gates**: Where valid source-model held-out evidence exists:
reliability diagrams, ECE/Brier/task metrics, sample counts/confidence
intervals, subgroup limitations, and route compatibility. Where it does not:
explicit unavailable state. No anomaly ground-truth inference, bounded
production lookup overhead, and rollback.

**Required figures**: reliability diagrams, confidence histograms, calibration
delta, subgroup calibration, lookup latency/resource delta.

## Cycle 015.3 - Deterministic Fast Explainers For Every Active Model

**Indivisible capability**: Complete fast-explainer coverage across all active
routes. Partial coverage would make aggregate explanations selectively blind.

**Dependencies**: 015.2.
**Streaming compatibility**: `stream-safe`.

**Implementation scope**:

- implement registered fast adapters for detector, behavior children/ensemble,
  RTMPose, OSNet ReID, YOLOE, tracking, pose kinematics, SRVL, and BSIL rules;
- expose score/margin/entropy, geometry, quality, temporal support,
  contradictions, thresholds, and counterfactual deltas;
- reject active routes without registered compatible adapters;
- replace placeholder explanation notes with real artifact/record contracts.

**Benchmark hypothesis**: Fast explanation coverage can be complete within
declared per-observation and per-frame overhead budgets.

**Acceptance gates**: Every ready route covered, deterministic replay,
model-output parity, bounded overhead/storage, no hidden fallback, and rollback.

**Required figures**: adapter coverage, per-adapter latency, contribution
examples, explanation completeness, output parity, storage/resource delta.

## Cycle 015.4 - Bounded Temporal Evidence And Episode Explanations

**Indivisible capability**: Bounded temporal windows plus deterministic episode
explanations. A temporal score without reconstructable windows is not valid.

**Dependencies**: 015.3.
**Streaming compatibility**: `stream-safe-with-config` using bounded time/count
windows and stale eviction.

**Implementation scope**:

- build bounded per-camera/per-identity evidence windows;
- derive bounded per-student multivariate signal-pattern windows from all valid
  registered signals;
- explain change points, drift, repeated patterns, instability, persistence,
  decay, hysteresis, cooldown, and invalid gaps;
- generate episode-level contribution summaries and counterfactuals;
- prove reconnect, out-of-order, eviction, and indefinite-stream behavior.

**Benchmark hypothesis**: Temporal evidence improves transient suppression and
diagnosis without unbounded state or unacceptable latency.

**Acceptance gates**: Deterministic replay, controlled transient/sustained
pattern-fixture behavior, invalid-gap/cold-start behavior, bounded soak state,
resource/latency budgets, and rollback. No behavioral recall claim.

**Required figures**: temporal state transitions, window coverage, spike versus
sustained behavior, state-size soak, latency/resource delta.

## Cycle 015.5 - Transparent Hierarchical Anomaly Score

**Indivisible capability**: The first reconstructable review-priority score,
its contribution records, and score-withholding rules.

**Dependencies**: 015.4.
**Streaming compatibility**: `stream-safe`.

**Implementation scope**:

- implement per-signal observed-pattern deviation and reliability-weighted
  components;
- build versioned contamination-aware per-student pattern profiles;
- add context/persistence/uncertainty modifiers;
- persist score/contribution/reconstruction records;
- enforce minimum coverage, baseline, identity, route, calibration, and
  uncertainty gates;
- expose exact counterfactual threshold/recovery deltas;
- deprecate score language implying cheating probability.

**Benchmark hypothesis**: Transparent pattern-deviation scoring provides
reconstructable review prioritization while preserving latency/throughput and
avoiding unsupported behavioral-truth claims.

**Acceptance gates**: Exact reconstruction, deterministic replay, controlled
pattern fixtures, metamorphic/invariant and sensitivity checks, cold-start/
contamination/drift/quarantine behavior, withholding correctness, runtime
budgets, required observed-pattern vocabulary, no anomaly training or
ground-truth claim, and rollback.

**Required figures**: score decomposition, per-student pattern envelope versus
window, coverage versus score, threshold/counterfactual plots, fixture and
invariant outcomes, contamination/quarantine behavior, performance delta.

## Cycle 015.6 - Missingness, Uncertainty, And Conformal Calibration

**Indivisible capability**: Uncertainty and coverage semantics plus conformal
outputs where valid. A conformal value without assumption/coverage evaluation
cannot be accepted.

**Dependencies**: 015.5.
**Streaming compatibility**: `stream-safe-with-config`; adaptive calibration
state is bounded and drift-gated.

**Implementation scope**:

- propagate missingness and uncertainty into scoring/explanations;
- add conformal calibration snapshots and validity assumptions;
- evaluate coverage under time/session/camera drift;
- withhold conformal outputs when assumptions fail;
- label all conformal coverage as distributional only, never behavioral
  correctness;
- compare adaptive versus fixed calibration as separate candidates inside the
  same cycle only if the cycle benchmark matrix keeps each candidate isolated.

**Benchmark hypothesis**: Explicit uncertainty improves coverage/error control
without creating false precision or unacceptable overhead.

**Acceptance gates**: Coverage/error metrics, drift behavior, withholding,
reconstruction, bounded state, latency/resource budgets, and rollback.

**Required figures**: empirical versus target coverage, interval/set size,
coverage under drift, missingness impact, performance delta.

## Cycle 015.7 - Cross-Model Fusion And Explanation Graph

**Indivisible capability**: Identity-gated evidence fusion plus a reconstructable
explanation graph. Fusion without source-preserving graph evidence is unsafe.

**Dependencies**: 015.6.
**Streaming compatibility**: `stream-safe-with-config` using bounded graph
windows.

**Implementation scope**:

- implement source-preserving fusion and contradiction policies;
- add identity-gated peer/scene/SRVL context;
- create explanation graph nodes/edges with contribution and lineage;
- keep unresolved identity/context unresolved;
- prevent graph contamination across invalid windows.

**Benchmark hypothesis**: Context/fusion improves explanation usefulness
without association, source-output, or latency regressions.

**Acceptance gates**: Identity/association metrics, contradiction visibility,
graph reconstruction, non-ground-truth reviewer usability/disagreement,
bounded graph state, performance, and rollback.

**Required figures**: explanation graphs, contribution flow, contradiction
matrix, association/fragmentation metrics, ranking/performance delta.

## Cycle 015.8 - Offline And On-Demand Deep Vision XAI

**Indivisible capability**: Isolated deep-XAI execution plus its fidelity,
sanity, security, and lifecycle evidence.

**Dependencies**: 015.3 and 015.7.
**Streaming compatibility**: `offline-only` for automatic execution; authorized
bounded snapshots may be requested from live evidence after capture.

**Implementation scope**:

- add deep-XAI request/task/artifact lifecycle;
- implement eligible D-RISE, CAM/IG, perturbation, joint occlusion, and exemplar
  adapters;
- enforce method allowlist and route compatibility;
- evaluate sanity, fidelity, stability, latency, memory, artifact size, access,
  retention, idempotency, deadline, reconciler, and rollback.

**Benchmark hypothesis**: Selected methods provide faithful diagnostic value
without affecting critical inference and within bounded deep-task resources.

**Acceptance gates**: Method-specific fidelity/sanity/stability, task isolation,
critical-path parity, artifact access/lineage, lifecycle reconciliation, and
rollback.

**Required figures**: attribution examples, deletion/insertion curves, sanity
comparisons, stability distribution, task latency/resource and critical-path
delta.

## Cycle 015.9 - Prototype, Case Comparison, And Governed Review Feedback

**Indivisible capability**: Human-review diagnosis loop with governed prototypes
and feedback that cannot self-modify production.

**Dependencies**: 015.7 and 015.8.
**Streaming compatibility**: `stream-safe` for read-only review; prototype
building/promotion is `offline-only`.

**Implementation scope**:

- add governed prototypes/exemplars and similar-case comparison;
- expose reviewer explanation, evidence, disagreement, and feedback workflow;
- audit access and feedback lineage;
- prove feedback cannot directly alter thresholds/baselines/models;
- prove feedback cannot become behavioral ground truth, a training target, or
  a direct pattern-profile update;
- add promotion workflow requiring a later governed cycle.

**Benchmark hypothesis**: Prototypes and explanations improve reviewer
consistency/time without unsafe automatic adaptation.

**Acceptance gates**: Reviewer agreement/time study explicitly labeled
non-ground-truth, privacy/access, feedback immutability, no-training-target and
no-direct-mutation proof, performance, and rollback.

**Required figures**: reviewer time/agreement, prototype similarity, disagreement
analysis, access/audit summary, performance delta.

## Cycle 015.10 - Shared WebGL2 Explanation Workbench

**Indivisible capability**: One shared renderer core and migrated analytical
surfaces. Context management, data stores, and migrated figures must be
accepted together because partial migration leaves the original context and
Canvas2D failures.

**Dependencies**: 015.3, 015.5, and 015.7.
**Streaming compatibility**: `stream-safe`.

**Implementation scope**:

- implement shared WebGL2 renderer/context/buffer/data-store core;
- migrate time series, matrices, heatmaps, metrics, scene map, telemetry lanes,
  contribution charts, attribution overlays, and explanation graph;
- add typed-array/binary incremental transport, Workers, LOD/tiling, ring
  buffers, stable colors, interactions, downloads, accessibility, and recovery;
- remove accepted Canvas2D analytical rendering.

**Benchmark hypothesis**: Shared WebGL2 improves update latency, frame time,
memory, context stability, and large-data responsiveness without regressions.

**Acceptance gates**: All accepted analytical figures report WebGL2, context
budget holds, no crash on toggles, representative/stress frame-time budgets,
large matrix/series correctness, context-loss recovery, accessibility,
downloads, and rollback.

**Required figures**: frame-time distributions, update latency, context count,
buffer/upload bytes, memory proxy, stress scaling, dropped-update/recovery
summary.

## Cycle 015.13 - Student Interaction Graph Signals And Plots

**Indivisible capability**: A deterministic, identity-gated student-interaction
graph plus its per-student graph signals and shared-WebGL2 plots. The builder,
the registered graph signals, and the render layer must be accepted together
because graph features without a reconstructable graph, or a graph without
identity gating, are unsafe.

**Dependencies**: 015.5, 015.7, and 015.10.
**Streaming compatibility**: `stream-safe-with-config` using bounded graph
windows, node/edge caps, and stale-edge eviction.

**Implementation scope**:

- build a deterministic identity-gated graph (nodes = resolved tracks; typed
  edges = proximity, directed/mutual gaze cone, orientation, co-movement,
  shared-scene) reusing SRVL/gaze/pose/scene signals;
- register bounded per-student graph features as governed signals feeding the
  observed-pattern profiles;
- keep ambiguous-identity edges `unresolved` and out of valid features;
- render node-link, adjacency-matrix, and dyad-timeline views on the shared
  WebGL2 core with LOD/tiling, context-loss recovery, and numeric fallback;
- prove no learned graph model (`PROBE_ONLY`) drives a production score and no
  edge/feature asserts collusion or cheating.

**Benchmark hypothesis**: Deterministic interaction-graph signals and plots add
reconstructable relational evidence within bounded latency/state and render
budgets, without introducing a trainable collusion target.

**Acceptance gates**: Deterministic reconstruction, identity-gating and
unresolved-edge correctness, node/edge bound/overflow behavior, per-student
feed-into-profile and no-peer-mutation proof, WebGL render/context budgets and
numeric fallback, `PROBE_ONLY` learned-graph isolation, non-accusatory
vocabulary, performance, and rollback.

**Required figures**: interaction graph snapshots, adjacency/interaction matrix,
per-student graph-feature envelopes, edge-persistence/identity-confidence
distribution, graph render frame-time/context budget, performance delta.

## Cycle 015.14 - General Baseline Ingestion, Dual Comparison, And Parameter Provenance

**Indivisible capability**: A corpus-ingested general baseline, dual comparison
(self + population), and parameter provenance. These ship together because dual
comparison needs the general baseline, and the no-hardcoding provenance must
cover both the new baseline-derived values and the existing scoring values.

**Dependencies**: 015.4 and 015.5.
**Streaming compatibility**: corpus ingestion is `offline-only`; dual-comparison
lookup and provenance resolution are `stream-safe`.

**Implementation scope**:

- ingest the supported-video corpus into a versioned, contamination-aware general
  baseline (population tier plus context tiers) reusing the per-student
  robust-statistics, cold-start, quarantine, and drift machinery;
- compare each valid window against both the student's own profile (primary) and
  the compatible general baseline (contextual), exposing `deviation_vs_self` and
  `deviation_vs_population` and keeping tier disagreement visible;
- route every operational value through one parameter resolver and bind it to a
  `learned`/`configured` `ParameterProvenanceRecord`;
- prove the general baseline is never ground truth and no value is hardcoded.

**Benchmark hypothesis**: A general baseline plus dual comparison improves
cold-start/context anchoring and tunability within bounded overhead, without any
trained anomaly target or hardcoded constant.

**Acceptance gates**: General-baseline cold-start/contamination/quarantine/drift
behavior, dual-delta reconstruction from both snapshots, tier-disagreement
visibility, complete parameter-provenance coverage (no magic numbers),
never-ground-truth proof, runtime budgets, and rollback.

**Required figures**: population-versus-self envelopes, dual-delta decomposition,
baseline contamination/quarantine behavior, parameter-provenance coverage
(`learned` vs `configured`), performance delta.

## Cycle 015.15 - Model Promotion Governance (Probe -> Mandatory)

**Indivisible capability**: An evidence-gated promotion lifecycle plus its record,
gates, governed approval, and rollback. The lifecycle, the gate evaluation, and
the rollback path must ship together because a stage transition without recorded
evidence and a revert path is unsafe.

**Dependencies**: 015.0 and 015.8.
**Streaming compatibility**: shadow/canary serving is `stream-safe`; promotion
fitting/decision is `offline-only`.

**Implementation scope**:

- implement stages `PROBE_ONLY` -> `SHADOW` -> `CANARY` -> `MANDATORY`
  (+ `ARCHIVED`/`ROLLED_BACK`) and the `ModelPromotionRecord`;
- run shadow/canary with a champion/challenger serving-metrics collector
  (latency/throughput/resource, serving distribution, minimum shadow duration);
- evaluate gates G1-G6 (doctrine cap; determinism; serving SLO benchmark; signal
  quality; monitoring + rollback; model card + ledger + governed approver);
- enforce signal/representation-only target and reject behavioral-decision or
  accuracy/AUROC promotions;
- prove governed-approver sign-off and automated rollback to the prior stage.

**Benchmark hypothesis**: A probe can be promoted to a governed production signal
within bounded serving overhead once it meets the recorded evidence bar, without
any behavioral-truth claim.

**Acceptance gates**: Complete promotion record on every transition, serving-SLO
benchmark and equal-or-better distribution, determinism, drift + auto-rollback,
governed-approver RBAC, model card, doctrine-cap proof (no decision authority, no
behavioral accuracy/AUROC), and rollback restoring the prior stage.

**Required figures**: serving-distribution champion-vs-challenger, latency/SLO
versus budget, gate-coverage matrix, shadow/canary timeline, rollback
verification.

## Cycle 015.11 - Observability, Performance, Stability, Security, And Fairness

**Indivisible capability**: Cross-cutting operational/scientific acceptance
suite. These gates must be evaluated together because one green dimension
cannot compensate for a critical failure in another.

**Dependencies**: 015.0 through 015.10.
**Streaming compatibility**: `stream-safe`.

**Implementation scope**:

- add end-to-end XAI/anomaly/renderer telemetry and dashboards;
- run load, soak, queue-failure, restart, context-loss, DB/Redis, retention,
  access-control, privacy, calibration, subgroup, identity, and explanation
  stability tests;
- verify the no-ground-truth doctrine, required observed-pattern vocabulary,
  and absence of anomaly-model training/fine-tuning paths;
- report subgroup signal-distribution and operational differences without
  claiming behavioral-accuracy or fairness parity that ground truth cannot
  support;
- verify instrumentation readiness and evidence completeness;
- quantify remaining bottlenecks and debt.

**Benchmark hypothesis**: The integrated disabled/default and enabled candidate
profiles are stable, measurable, secure, and scientifically auditable.

**Acceptance gates**: All required metrics or unavailable reasons, no hidden
xfails/fallbacks/placeholders, soak and failure recovery, access/privacy,
subgroup limitations, no label-based behavioral accuracy claim, no anomaly
training/fine-tuning path, no unsupported fairness-performance claim,
performance budgets, and rollback.

**Required figures**: latency/throughput/resource overview, soak trends, queue
and failure recovery, calibration/subgroup/error analysis, evidence
completeness.

## Cycle 015.12 - Production Canary, Rollback, And Final Acceptance

**Indivisible capability**: One governed canary/promotion/rollback decision.
Canary evidence and final rollback proof cannot be accepted independently.

**Dependencies**: 015.11.
**Streaming compatibility**: `stream-safe` only for capabilities previously
accepted as stream-safe.

**Implementation scope**:

- define disabled, shadow, reviewer-visible, and promoted profiles;
- execute production canary with explicit exposure and stop conditions;
- prove score/explanation/frontend reconciliation and human-review semantics;
- prove the no-ground-truth knowledge limit and observed-pattern vocabulary are
  present in every canary surface;
- execute rollback and verify original behavior;
- package final immutable evidence and open remaining research debt.

**Benchmark hypothesis**: The accepted capability set can operate in production
without violating latency, throughput, source-output correctness, pattern
invariants, stability, safety, or rollback gates.

**Acceptance gates**: Canary stop/promote criteria, reviewer and runtime
evidence, full benchmark/figure/ledger package, rollback proof, branch/SHA/env/
route/calibration fingerprint, and no unresolved blocker.
The final fingerprint also includes the observed-pattern profile/feature schema
and no-ground-truth policy version.

**Required figures**: canary versus baseline, review-priority distribution,
latency/throughput/resource/correctness, rollback verification, final evidence
completeness.

## Dependency Order

```text
015.0 -> 015.1 -> 015.2 -> 015.3 -> 015.4 -> 015.5 -> 015.6 -> 015.7
                                      |                         |
                                      +-------> 015.8 <---------+
                                                  |
                                               015.9

015.3 + 015.5 + 015.7 -> 015.10
015.3 + 015.5 + 015.7 + 015.10 -> 015.13
015.4 + 015.5 -> 015.14
015.0 + 015.8 -> 015.15
015.0 through 015.10, plus 015.13 + 015.14 + 015.15 -> 015.11 -> 015.12
```

Parallel implementation work is allowed only where dependencies and file
ownership do not overlap. Production acceptance decisions remain sequential in
the order above unless a later plan amendment proves an alternate dependency
graph.
