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
- whether 2025 PCBEAR/DANCE results reproduce on the project's signals.

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
