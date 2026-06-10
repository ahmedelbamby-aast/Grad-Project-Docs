# Research: Explainable Behavioral Evidence And Anomaly Scoring

**Date**: 2026-06-08
**Scope**: Primary papers, official framework documentation, production
patterns, and current repository/runtime evidence relevant to explaining all
active models and building an anomaly score layer.

## Research Guardrails

- [no-ground-truth-doctrine.md](no-ground-truth-doctrine.md) is binding. No
  anomaly/cheating/normality dataset or accepted behavioral ground-truth method
  exists, so this plan does not train or fine-tune an anomaly model.
- Explainability is evidence about model behavior, not proof of human intent.
- Visually plausible saliency is not automatically faithful or causal.
- Raw classifier/detector confidence is not calibrated certainty.
- An anomaly score is not a probability of misconduct.
- New 2025 methods are treated as experimental probes until independent
  evaluation and production benchmarking prove their value.
- Production choices must fit the existing Triton/PostgreSQL/Redis/Celery and
  React/WebGL architecture.

## Primary And Official Sources Reviewed

| Area | Source | Production relevance |
|---|---|---|
| Detector attribution | D-RISE, https://arxiv.org/abs/2006.03204 | Perturbation attribution designed for object detectors |
| CNN attribution | Grad-CAM, https://arxiv.org/abs/1610.02391 | Architecture-dependent class localization |
| Architecture-light CAM | Eigen-CAM, https://arxiv.org/abs/2008.00299 | Useful candidate where class-gradient access is difficult |
| Attribution sanity | Sanity Checks for Saliency Maps, https://arxiv.org/abs/1810.03292 | Requires parameter/data randomization checks |
| Confidence calibration | On Calibration of Modern Neural Networks, https://arxiv.org/abs/1706.04599 | Establishes calibration as separate from accuracy |
| Integrated gradients | Captum official documentation, https://captum.ai/docs/extension/integrated_gradients | Supported implementation reference for differentiable models |
| Additive explanations | SHAP official documentation, https://shap.readthedocs.io/ | Candidate for supported tabular/fusion models and offline audit |
| One-class anomaly | Deep SVDD, https://proceedings.mlr.press/v80/ruff18a.html | Future probe only; not trainable/promotable under this plan |
| Visual anomaly | PatchCore, https://arxiv.org/abs/2106.08265 | Future probe only; not trainable/promotable under this plan |
| Visual anomaly | PaDiM, https://arxiv.org/abs/2011.08785 | Future probe only; not trainable/promotable under this plan |
| Adaptive conformal anomaly | Adaptive Conformal Predictions for Time Series, https://proceedings.mlr.press/v202/bashari23a.html | Candidate uncertainty control under nonstationarity |
| Detection uncertainty | Conformal Prediction Sets for Object Detection, https://arxiv.org/abs/2405.20227 | Candidate uncertainty sets for detection outputs |
| Online anomaly | River Half-Space Trees documentation, https://riverml.xyz/latest/api/anomaly/HalfSpaceTrees/ | Future bounded probe only under this plan |
| Drift/anomaly framework | Alibi Detect official documentation, https://docs.seldon.ai/alibi-detect | Industry reference for detector/drift interfaces |
| Serving metrics | Triton metrics documentation, https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/metrics.html | Runtime latency/queue/GPU evidence |
| Explainability governance | NIST Four Principles of Explainable AI, https://www.nist.gov/publications/four-principles-explainable-artificial-intelligence | Explanation, meaningfulness, accuracy, and knowledge limits |
| AI risk governance | NIST AI RMF, https://www.nist.gov/itl/ai-risk-management-framework | Human oversight, risk, measurement, and governance |
| Experimental 2025 method | PCBEAR, https://arxiv.org/abs/2506.17438 | Research probe only |
| Experimental 2025 method | DANCE, https://arxiv.org/abs/2508.18182 | Research probe only |
| Skeleton-based VAD | STG-NF, https://orhir.github.io/STG_NF/ | PROBE_ONLY; consumes existing `rtmpose` skeletons; no new labels |
| Skeleton-based VAD | DA-Flow, https://arxiv.org/abs/2406.02976 | PROBE_ONLY; recent SOTA skeleton VAD; few parameters |
| Pretrained model governance | [pretrained-models-registry.md](pretrained-models-registry.md) | Frozen Class A signal sources + Class B PROBE_ONLY representations |
| Skeleton spatio-temporal graph | ST-GCN, https://arxiv.org/abs/1801.07455 | PROBE_ONLY graph backbone over existing `rtmpose` skeletons; production graph stays deterministic |
| Group relation graph | Actor Relation Graph (ARG), https://arxiv.org/abs/1904.10117 | PROBE_ONLY reference for multi-person interaction modeling |
| Dynamic-graph anomaly | StrGNN structural temporal GNN, https://arxiv.org/abs/2005.07427 | PROBE_ONLY unsupervised dynamic-graph anomaly reference; no labels |
| Graph anomaly survey | Deep Graph Anomaly Detection survey (TKDE 2025), https://github.com/mala-lab/Awesome-Deep-Graph-Anomaly-Detection | Landscape reference; all candidates remain PROBE_ONLY |
| WebGL graph rendering | cosmos.gl (OpenJS), https://openjsf.org/blog/introducing-cosmos-gl and sigma.js, https://www.sigmajs.org/ | GPU graph-render references; only as a measured candidate behind the shared WebGL2 core |
| ML production readiness | ML Test Score (Breck et al., IEEE Big Data 2017), https://research.google/pubs/pub46555/ | Four-area production-readiness rubric for promotion gates |
| Model registry promotion | MLflow Model Registry, https://mlflow.org/docs/latest/ml/model-registry | Staged `dev → staging → production` promotion with role-based approval |
| Staged rollout / champion-challenger | DataRobot champion/challenger, https://www.datarobot.com/blog/introducing-mlops-champion-challenger-models/ | Shadow/canary promotion, governed approver, automated rollback |
| Model documentation | Model Cards, https://arxiv.org/abs/1810.03993 | Promotion documentation (purpose, metrics, limits) |
| Pseudo-label self-training | Noisy Student, https://arxiv.org/abs/1911.04252 | Filtered pseudo-label fine-tuning of probe COPIES |
| Pseudo-label filtering | Less is More: Pseudo-Label Filtering for Continual TTA, https://arxiv.org/abs/2406.02609 | Adaptive filtering prevents error accumulation in self-training |
| Source-free video adaptation | Source-free Video DA by Learning from Noisy Labels, https://arxiv.org/abs/2311.18572 | Adapting video models on unlabeled target video |
| Test-time adaptation | TENT, https://arxiv.org/abs/2006.10726 | Ephemeral entropy-minimization adaptation; never persisted weights |
| Consistency self-training | FixMatch, https://arxiv.org/abs/2001.07685 | Confidence-thresholded pseudo-labels with consistency |

## Decision 1: Use A Two-Lane XAI Architecture

**Decision**: Separate XAI into:

1. a deterministic, bounded fast path that produces score decomposition,
   calibration, quality, missingness, temporal support, contradictions, and
   threshold counterfactuals inline; and
2. an asynchronous deep path for attribution, perturbation, prototypes, and
   case comparison.

**Rationale**: Detector perturbation and gradient attribution are too expensive
and too architecture-specific for every frame. The fast path is needed for
continuous diagnosis; deep XAI is valuable for selected observations and model
audit.

**Alternatives considered**:

- Run saliency for every frame: rejected for latency, GPU contention, artifact
  volume, and weak default faithfulness.
- Use only post-hoc text explanations: rejected because text without
  reconstructable evidence creates false confidence.

## Decision 2: Use Model-Appropriate Explainers, Not One Universal Method

**Decision**: Register model-specific adapters:

| Model family | Fast explanation | Deep/on-demand candidate |
|---|---|---|
| YOLO person detector | score, box, calibration, temporal/scene support | D-RISE; validated Eigen-CAM |
| Behavior YOLO models | class/margin/entropy, crop quality, pose agreement | crop perturbation; IG/CAM when supported |
| RTMPose | keypoint quality, geometry, joint contributions | joint occlusion; heatmap attribution when exposed |
| OSNet ReID | distance, margin, prototypes, lifecycle evidence | governed exemplar comparison |
| YOLOE segmentation | prompt/score/mask/overlap/contradiction | D-RISE or mask perturbation |
| Tracking/SRVL/BSIL rules | exact deterministic derivation | no deep method required |
| Anomaly/fusion layer | exact additive decomposition/counterfactual | SHAP only for supported learned fusion models |

**Rationale**: Methods answer different questions and require different model
interfaces. ReID embedding dimensions and deterministic geometry should not be
misrepresented as human-semantic saliency.

**Alternatives considered**:

- Apply Grad-CAM everywhere: rejected because it is architecture-dependent and
  does not directly explain detector assignment, tracking, geometry, or rules.
- Apply SHAP everywhere: rejected because computational cost and semantics vary
  substantially by model and input.

## Decision 3: Require Faithfulness And Sanity Evaluation

**Decision**: Deep-XAI artifacts are production-valid only if the configured
method passes method-appropriate checks:

- model-parameter and data-randomization sanity checks where applicable;
- deletion/insertion or perturbation-fidelity measurement;
- repeated-run stability and sensitivity;
- localization overlap where governed labels exist;
- latency, memory, and artifact-size budgets;
- explicit unavailable reasons when checks cannot run.

**Rationale**: The saliency sanity-check literature demonstrates that visually
appealing maps may be insensitive to learned parameters. A picture alone is not
evidence.

**Alternatives considered**:

- Trust qualitative reviewer preference alone: rejected as insufficient for
  scientific or production acceptance.

## Decision 4: Calibrate Before Calling Confidence Certainty

**Decision**: Create versioned calibration snapshots per existing source-model
route, artifact, output, evidence cohort, and relevant subgroup only where
task-appropriate held-out evidence exists. Report calibration quality and use
calibrated confidence bands in explanations. If such evidence is unavailable,
calibration remains unavailable rather than inferred from anomaly patterns.

**Rationale**: Modern neural network confidence is frequently miscalibrated.
Calibration must be measured separately from classification or localization
quality.

**Required measures**:

- reliability diagrams and sample counts;
- expected calibration error with declared binning;
- Brier score or task-appropriate proper score;
- confidence intervals and subgroup limitations;
- age/drift/route compatibility of the calibration snapshot.

**Alternatives considered**:

- Treat raw YOLO scores as certainty: rejected.
- Fit one global calibrator across all models/routes: rejected because output
  semantics and route artifacts differ.

## Decision 5: Build The First Anomaly Score From Observed Signal Patterns

**Decision**: Start with a transparent hierarchical score:

```text
valid_component_i =
    pattern_deviation_magnitude_i
    * reliability_i
    * temporal_support_i

evidence_strength =
    weighted_mean(valid_component_i over valid evidence only)

review_priority_score =
    100
    * clamp(evidence_strength
            * context_support
            * persistence_support
            * (1 - uncertainty_penalty))
```

The score is withheld when valid evidence coverage, observed-pattern profile
validity, identity continuity, route compatibility, or required source-model
calibration quality is below its governed gate. Contradiction remains visible
as evidence; it does not disappear inside the aggregate.

**Rationale**: There is no valid anomaly-behavior ground truth or training
dataset. A transparent, per-student, contamination-aware observed-pattern
profile is reconstructable and testable without pretending that historical
behavior is known-normal truth.

**Alternatives considered**:

- Train an opaque end-to-end misconduct classifier: prohibited because there
  are no valid targets, no accepted ground truth, and no legitimate promotion
  metric.
- Average raw model confidence: rejected because confidence scales and
  reliability differ.

## Decision 6: Do Not Train Or Fine-Tune An Anomaly Model In This Plan

**Decision**:

- production scoring uses deterministic robust statistics, bounded temporal
  patterns, contamination controls, threshold/hysteresis, and explicit
  missingness;
- reviewer feedback, assumed-normal history, heuristic output, BSIL output, and
  model agreement are never anomaly training targets or ground truth;
- Half-Space Trees, Deep SVDD, PatchCore, PaDiM, PCBEAR, DANCE, and other
  learned anomaly candidates remain future `HYPOTHESIS_ONLY` or `PROBE_ONLY`
  research; and
- a trainable anomaly candidate requires a separate future governed plan that
  establishes legitimate target semantics, dataset authority, evaluation, and
  promotion criteria.

**Rationale**: The production database has no BSIL/anomaly rows and the project
has no anomaly dataset or accepted behavioral ground truth. Training on
assumed-normal or reviewer-labeled history would manufacture authority the
system does not possess.

**Alternatives considered**:

- Select the newest research method immediately: rejected because recency is
  not production evidence.
- Treat all previous behavior as normal training data: rejected because
  contamination and unobserved behavior make that claim invalid.

## Decision 7: Use Conformal Outputs Only When Assumptions Are Demonstrated

**Decision**: Add conformal p-values, prediction sets, or intervals only behind
an explicit calibration snapshot, validity assumptions, drift monitoring, and
coverage evaluation. Mark them unavailable when assumptions fail. Any
distributional coverage statement is not a claim of behavioral correctness.

**Rationale**: Conformal methods can give useful uncertainty guarantees, but
coverage claims are invalid when calibration data or nonstationarity handling
is inappropriate.

**Alternatives considered**:

- Always show a conformal-looking interval: rejected as false precision.

## Decision 8: Preserve Missingness, Contradiction, And Knowledge Limits

**Decision**: Every evidence envelope and score contains explicit missingness,
quality, contradiction, knowledge-limit, and unavailable-reason fields.

**Rationale**: NIST explainability principles require explanations to reflect
their limits. Missing evidence is often informative and cannot be treated as a
neutral zero.

**Alternatives considered**:

- Impute missing signals to zero in scoring: rejected.
- Hide contradictory models behind the winner: rejected.

## Decision 9: Extend Existing Ownership Boundaries

**Decision**:

- `apps.behavior` owns evidence envelopes, temporal evidence, explanation
  composition, and lineage;
- `apps.anomalies` owns source-model calibration where evidence exists,
  observed-pattern profiles, scoring, thresholds, drift, conformal
  calibration, and non-ground-truth review feedback governance;
- `apps.pipeline` owns model-specific extraction at the inference boundary and
  immutable route snapshots;
- `apps.telemetry` owns metrics;
- `apps.video_analysis` remains source evidence authority;
- the frontend adds one shared WebGL analytical renderer.

**Rationale**: The repository already has these boundaries. A new parallel XAI
or anomaly service would duplicate persistence, queue, telemetry, and access
logic.

**Alternatives considered**:

- New standalone XAI microservice: rejected until measurement proves process
  isolation is required.
- Put all scoring inside pipeline tasks: rejected because anomaly governance
  must not depend on pipeline internals.

## Decision 10: Capture An Immutable Runtime Route Snapshot

**Decision**: Before scoring or explanation, capture and fingerprint:

- logical task-to-model route;
- model/version/artifact digests;
- active Triton profile and endpoint;
- relevant environment/config values;
- calibration and feature schema versions.

**Rationale**: Production evidence showed route drift between documented
accepted variants and the active runtime. The current route service also shares
a mutable default route table. Explanations are not reconstructable without an
immutable point-in-time route.

**Alternatives considered**:

- Resolve the route again later: rejected because it may have changed.

## Decision 11: Use A Shared WebGL2 Analytical Renderer

**Decision**: Converge time series, matrices, heatmaps, metric charts, scene
maps, contribution charts, attribution overlays, and explanation graphs onto a
shared WebGL2 core with:

- a context-budget manager and persistent contexts;
- typed-array data stores and buffer reuse;
- incremental append and bounded ring buffers;
- Web Worker transforms/downsampling;
- viewport-aware min/max aggregation and matrix tiling;
- visibility-paused rendering and context-loss recovery;
- stable student colors and semantic behavior palettes;
- binary transport for high-volume data with JSON fallback;
- accessible numeric/table fallback when WebGL is unavailable.

**Rationale**: Existing components independently create WebGL contexts and
comments document context exhaustion on toggles. `SceneMapRenderer` and
`TelemetryCanvas` still use Canvas2D. A shared renderer reduces repeated code,
main-thread cost, and context churn.

**Alternatives considered**:

- Continue separate per-chart renderers: rejected for duplication and context
  pressure.
- Adopt WebGPU immediately: deferred as a measured future candidate after the
  WebGL2 baseline; browser support and operational complexity must be measured.

## Decision 12: Treat Deep-XAI Artifacts As Sensitive Evidence

**Decision**: Deep-XAI artifacts are job-scoped, authenticated, audited,
digest-addressed, retention-governed, and idempotently generated.

**Rationale**: Attribution maps, crops, prototypes, and exemplars may contain
classroom imagery and identity evidence.

**Alternatives considered**:

- Public static artifact URLs: rejected.
- Embed large artifacts in relational JSON: rejected for size and access
  control.

## Decision 13: Benchmark One Causal Variable Per Atomic Cycle

**Decision**: Each atomic cycle introduces one coherent causal change and must
include the complete evidence packet: baseline, implementation, tests,
production benchmark, figures, manifest/digests, decision, and rollback.

**Rationale**: Combining calibration, scoring, deep XAI, queue changes, and
renderer changes in one benchmark makes performance and correctness deltas
unattributable.

**Alternatives considered**:

- One large XAI release: rejected for causal ambiguity and rollback risk.

## Decision 14: Evaluate Pattern Semantics Without Behavioral Ground Truth

**Decision**: Anomaly-layer acceptance uses exact reconstruction,
deterministic replay, controlled signal-pattern fixtures, metamorphic and
invariant tests, sensitivity/counterfactual checks, cold-start/contamination/
drift/quarantine behavior, real-media stability, reviewer usability, and
production performance. It does not report label-based anomaly accuracy.

**Rationale**: Controlled fixtures prove implementation semantics, not whether
real human behavior is cheating or abnormal. Reviewer feedback can expose
operational concerns but cannot become ground truth or a hidden training
target.

**Alternatives considered**:

- Report AUROC/F1 against heuristic or reviewer labels: rejected because those
  labels are not accepted behavioral ground truth.
- Use model agreement as truth: rejected because correlated models can agree
  while being wrong.

## Decision 15: Build The Student-Interaction Graph As A Deterministic Signal And Plot

**Decision**: Add a student-interaction graph whose **production** form is a
deterministic, identity-gated consolidation of existing relational signals
(SRVL pairwise geometry, behavior `InteractionEdge`, gaze/`head_yaw`
orientation, the COMBINED head, and scene segmentation):

- nodes are tracked students/teacher; edges are typed relational evidence
  (`proximity`, directed/`mutual_gaze`, `orientation_toward_peer`, `co_movement`,
  `shared_scene`);
- per-student graph features (degree, mutual-gaze dwell, directed-attention
  duration, persistence, clustering/centrality proxy, dyad strength) become
  governed signals that feed the per-student observed-pattern profiles;
- the graph renders as a **live, real-time, continuously-updating** node-link plot
  through the shared WebGL2 core (Decision 11) — additional to, not a replacement
  for, the existing plots — updating incrementally on every
  `xai.interaction_graph.appended` event, with its adjacency matrix and dyad
  timelines reusing the matrix-tile and series stores;
- learned graph models (ST-GCN-family, GAT/GraphSAGE, ARG/GroupFormer,
  StrGNN/TADDY-style dynamic-graph anomaly) are `PROBE_ONLY` per the registry and
  cannot drive a production score.

**Rationale**: Many review-worthy moments are relational, yet the no-ground-truth
doctrine forbids training a collusion/cheating model. A deterministic graph turns
relations into reconstructable per-student signals without manufacturing
ground truth, and reuses the SRVL graph-degree/centrality work already noted in
`signal-catalog.md`. Because the strongest learned graph methods are
skeleton-based and consume the `rtmpose` skeletons already produced, a learned
graph probe needs no new labels — but stays a research hypothesis.

**Alternatives considered**:

- Train a supervised collusion/group-cheating classifier on the interaction
  graph: prohibited — no valid targets or ground truth.
- Promote an unsupervised dynamic-graph anomaly model to production scoring:
  rejected — remains `PROBE_ONLY` with no accuracy/AUROC claim.
- Introduce a standalone GPU graph library (cosmos.gl/sigma.js) outside the
  shared renderer: deferred to a measured candidate so context budget and
  determinism are preserved.

## Decision 16: Tiered Assumed-Normal Baselines (Student + General) Via Corpus Ingestion

**Decision**: Learn a **general baseline** (population tier plus context tiers
such as age-band, scene type, camera, and session) by ingesting all supported
videos with the **same** robust-statistics, contamination, cold-start,
quarantine, and drift machinery used for per-student profiles. Score by **dual
comparison**: each window is compared against both the student's own profile
(primary) and the compatible general baseline (contextual), exposing
`deviation_vs_self` and `deviation_vs_population` separately. The general baseline
is a **General Population Baseline** aggregated across many students/sessions —
never a single student — yielding **General Boundaries**, while each student's
time-windowed, cold-start-aware profile yields **Local Boundaries**; the General
Boundaries also support a general classroom-level deviation distinct from any
individual.

**Rationale**: A per-student profile alone cannot anchor a brand-new student or a
globally unusual session; a population/context baseline supplies that anchor
without any labels. Reusing one machinery means the general tier inherits the
"contamination-aware, never ground truth" guarantees. The intuition mirrors
hierarchical/partial-pooling estimation: sparse per-student estimates are
interpreted against the population while staying distinct.

**Alternatives considered**:

- Population baseline only: rejected — erases legitimate per-student differences
  and risks penalizing individuals for population norms.
- Per-student only: rejected — no anchor for cold-start students/sessions.
- Train a population "normal/abnormal" classifier on the corpus: prohibited —
  that is assumed-normal-as-truth.

## Decision 17: No Hardcoded Operational Constants — Learned Or Configured, With Provenance

**Decision**: Every operational value (threshold, weight, envelope/quantile
bound, geometric constant, persistence/hysteresis window, gate) is either
**learned** from a governed baseline (data-derived) or read from a **fingerprinted
`.env`/config** key, and is bound through a `ParameterProvenanceRecord`
(`learned`/`configured`). A static verifier plus runtime reconciliation reject
magic numbers. Clean code: all values flow through a single parameter resolver,
never inline literals.

**Rationale**: XAI is the top priority — a reviewer must be able to ask "where did
this threshold come from?". Hardcoded constants are unexplainable, untunable, and
brittle across age-bands and cameras. Deriving bounds from data (quantile
envelopes, robust scale) where possible removes guesswork; the remainder live in
`.env` for safe, fingerprinted tuning.

**Alternatives considered**:

- Inline literals/defaults scattered in code: rejected — unexplainable, untunable,
  and drift-prone.
- One global config blob without provenance: rejected — cannot distinguish learned
  from configured values or reconstruct learned ones.

## Decision 18: Evidence-Gated Model Promotion (Probe -> Shadow -> Canary -> Mandatory)

**Decision**: A research/probe model is promoted to a production-mandatory role
only through a staged, evidence-gated lifecycle modeled on industry practice and
adapted to our doctrine:

- stages `PROBE_ONLY` -> `SHADOW` (prod inputs copied, outputs discarded) ->
  `CANARY` (limited, reviewer-visible, advisory) -> `MANDATORY` (production
  signal/representation), with `ARCHIVED`/`ROLLED_BACK`;
- gates G1-G6 (doctrine cap; reproducibility/determinism; native RTX 5090
  stride-1 serving-SLO benchmark with minimum shadow duration and
  equal-or-better serving distribution; signal quality/stability/calibration/
  XAI fidelity; monitoring + proven automated rollback; model card + ledger +
  governed-approver sign-off);
- every transition recorded in a `ModelPromotionRecord`, reversible, ledgered;
- every promotable candidate is **unsupervised or self-supervised** (no
  ground-truth labels exist) and is certified on serving quality, not accuracy.

**Rationale**: Industry promotes models with gated offline checks, shadow/canary
rollout, champion/challenger comparison, governed approval, and automated
rollback (ML Test Score rubric; MLflow/Databricks registry stages; DataRobot
champion/challenger; model cards; NIST AI RMF Measure/Manage). We keep those
**serving-quality** gates — which are doctrine-safe — and drop the
behavioral-accuracy criterion that our no-ground-truth constraint forbids. The
result is evidence-driven but **bounded**: a clear minimum bar, not infinite
gates.

**Alternatives considered**:

- Promote on benchmark pass alone with no governance/rollback: rejected — no
  approver, model card, or revert path.
- Allow a learned probe to become a behavioral decision authority once it "looks
  good": prohibited by the doctrine; requires a separate ground-truth program.
- Keep everything `PROBE_ONLY` forever: rejected — a useful representation should
  be promotable to a governed signal once it earns it with evidence.

## Decision 19: Probe Fine-Tuning Lane — Adapt Copies On The Ingested Corpus

**Decision**: During corpus ingestion, probe models may be adapted **only as
copies** (parent frozen, new lineage row, `PROBE_ONLY`) through five governed
options:

- **A. Filtered pseudo-label self-training**: the copy infers on the corpus;
  every inference is screened by the accepted standard deterministic methods
  (robust observed-pattern statistics, cross-model agreement, identity gating,
  temporal consistency, confidence/entropy gates) and recorded as
  accepted/refused/edited; the copy fine-tunes on accepted/edited signals only.
- **B. Frozen vs fine-tuned champion/challenger**: parent and Option-A copy run
  side-by-side on held-out corpus videos; compared on serving metrics, signal
  stability, drift, and agreement with the deterministic baseline.
- **C. Self-supervised continued pretraining (DAPT)**: continue the probe's own
  label-free objective on the corpus (flow likelihood for STG-NF/DA-Flow, masked
  reconstruction for VideoMAE-style) — no pseudo-labels at all.
- **D. Ephemeral test-time adaptation**: TENT-style entropy/BN adaptation with
  pseudo-label filtering at inference; never persisted as new weights.
- **E. Distillation from governed deterministic signals**: the copy mimics
  fast-path/ensemble outputs (machine-generated targets with lineage).

**Rationale**: This is exactly the self-training / source-free-adaptation /
test-time-adaptation playbook the literature uses on unlabeled video (Noisy
Student; pseudo-label filtering for continual TTA; source-free video DA from
noisy labels; TENT; FixMatch), constrained to our doctrine: filtered inferences
are model-derived training signals — never ground truth — and the known failure
mode (pseudo-label error accumulation / confirmation bias) is countered by the
adaptive filter policy, held-out comparison against the frozen parent, and the
deterministic-baseline agreement check. Reviewer feedback may only exclude
corpus windows, never label them.

**Alternatives considered**:

- Fine-tune the registered model in place: rejected — destroys the frozen
  baseline, breaks lineage and rollback.
- Treat filter-passing pseudo-labels as ground truth and report accuracy:
  prohibited by the doctrine.
- Unfiltered self-training on raw inferences: rejected — documented error
  accumulation; the accept/refuse/edit filter is mandatory.

## Experimental Research Backlog

The following questions remain research probes and cannot create production
claims without dedicated evidence:

- whether D-RISE fidelity is sufficient for the deployed YOLO/TensorRT routes;
- whether exported models expose architecture points needed by Eigen-CAM/IG;
- whether a future governed ground-truth program can establish legitimate
  behavioral target semantics;
- whether learned unsupervised methods add operational value without being
  misrepresented as behavior-validity improvements;
- whether conformal coverage remains valid under session/camera drift;
- whether WebGPU improves renderer throughput enough to justify a second
  backend;
- whether 2025 PCBEAR/DANCE results reproduce on the project's signals;
- whether a learned graph probe (ST-GCN-family or dynamic-graph anomaly) over the
  deterministic interaction graph adds operational value without being
  misrepresented as collusion-detection validity;
- whether a GPU graph library (cosmos.gl/sigma.js) beats the shared WebGL2 core
  for large interaction graphs enough to justify a second render path;
- how much of the general baseline should be partial-pooled into sparse
  per-student profiles without erasing legitimate individual differences;
- which operational bounds are robustly data-derivable from the corpus versus
  which must remain `.env`-configured, and how to keep both fully auditable;
- which probe (skeleton-VAD, graph, or representation) first earns promotion to a
  governed signal under the gates, and what minimum shadow duration and
  serving-distribution criterion fit our streams.

## Repository Evidence Used

- `production-inventory-20260608.md`
- `signal-catalog.md`
- `backend/apps/pipeline/services/model_route_service.py`
- `backend/apps/pipeline/model_lifecycle/explainability.py`
- `backend/apps/pipeline/model_lifecycle/visualizations.py`
- `backend/apps/pipeline/rule_engine.py`
- `backend/apps/behavior/anomaly_primitives.py`
- `backend/apps/anomalies/bsil_baselines.py`
- `backend/apps/anomalies/bsil_drift.py`
- `backend/apps/anomalies/bsil_thresholds.py`
- `frontend/src/components/telemetry/WebGLTimeseriesPlot.tsx`
- `frontend/src/components/telemetry/WebGLMatrixHeatmap.tsx`
- `frontend/src/components/telemetry/WebGLMetricChart.tsx`
- `frontend/src/components/telemetry/TelemetryCanvas.tsx`
- `frontend/src/components/scene/SceneMapRenderer.tsx`
