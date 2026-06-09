# Governed Signal Catalog And Derivation Plan

**Date**: 2026-06-08
**Purpose**: Define the raw, derived, contextual, quality, and runtime signals
available to XAI and anomaly scoring. Every enabled implementation must assign
a schema version, unit, validity rule, missingness behavior, retention policy,
and explanation strategy.

[no-ground-truth-doctrine.md](no-ground-truth-doctrine.md) is binding. Signals
support per-student observed-pattern comparison; they are not anomaly,
cheating, non-cheating, abnormality, or normality ground truth.

Pretrained-model lineage for every signal source is governed in
[pretrained-models-registry.md](pretrained-models-registry.md); all sources are
frozen Class A models captured via `XAIModelRouteSnapshot`.

## Signal Contract

Every signal definition must include:

| Field | Meaning |
|---|---|
| `signal_key` | Stable namespaced identifier |
| `schema_version` | Version of value and semantics |
| `owner` | Existing owning module/app |
| `source_refs` | Model, record, or artifact lineage |
| `scope` | Frame, observation, track, window, scene, session, or runtime |
| `unit` | Score, pixels, normalized coordinate, degrees, seconds, etc. |
| `validity_rule` | Minimum evidence and quality |
| `missingness_rule` | Explicit unavailable/degraded behavior |
| `calibration_ref` | Calibration snapshot when applicable |
| `explanation_strategy` | Fast and optional deep path |
| `streaming_compatibility` | Stream-safe, stream-safe-with-config, or offline-only |

Missing values are not zeros. Unknown, unavailable, degraded, suppressed, and
invalidated remain distinct.

## 1. Person Detector And Localization

**Source**: `person_detector`, detections, bounding boxes, frames, tracks.

### Raw Signals

- class ID/name, score, box coordinates, source/input shape, model route/version;
- detector call status, RTT, batch/input shape, frame cadence, and reuse state;
- assigned source-scoped local tracker ID and assignment confidence when present.

### Derived Signals And Tasks

- normalized centroid, width, height, area, aspect ratio, edge proximity, and
  seat/zone occupancy;
- per-track position, velocity, acceleration, trajectory curvature, dwell time,
  visible duration, entry/exit, and motion energy;
- box jitter, detector consistency, cadence/reuse age, occlusion support,
  overlap conflicts, duplicate candidates, and detection gaps;
- track fragmentation, merge risk, unresolved association, and identity
  continuity quality;
- scene-mask support/contradiction and person-count mismatch evidence.

### Explanation Strategy

- Fast: score/calibration, box geometry, temporal support, assignment quality,
  scene support, threshold delta, and contradictions.
- Deep on-demand: D-RISE or validated detector attribution, perturbation
  deletion/insertion fidelity, and optional Eigen-CAM where architecture support
  is proven.

## 2. Posture And Gaze Behavior Models

**Sources**: `posture_model`, `gaze_horizontal_model`,
`gaze_vertical_model`, `gaze_depth_model`, and `behavior_ensemble`.

### Current Semantic Outputs

- posture: `sitting`, `standing`;
- horizontal gaze: `left`, `right`;
- vertical gaze: `up`, `down`;
- depth gaze: `forward`, `backward`;
- persisted presentation names: `sitting_standing`, `attention_tracking`, and
  `hand_raising`.

### Derived Signals And Tasks

- raw/dense output summaries, top score, runner-up score, margin, entropy,
  selected class, NMS/filter thresholds, and crop/input metadata;
- calibrated class probability or confidence band;
- transition frequency, dwell duration, repeat count, change points, drift,
  instability, and sustained state;
- disagreement/support with pose-derived posture and head orientation;
- crop quality, truncation, blur/occlusion proxy, person-box support, and
  detector-to-behavior alignment;
- ensemble child agreement, child contribution, route variant, output-byte
  volume, and unavailable child output.

### Explanation Strategy

- Fast: selected class, margin, calibration, crop quality, temporal support,
  pose support/contradiction, threshold counterfactual, and ensemble-child
  decomposition.
- Deep on-demand: crop perturbation, integrated gradients or architecture-valid
  attribution, class-specific saliency, and sanity/fidelity evaluation.

### Required Audit

`backend/apps/pipeline/multi_model.py` maps horizontal classes as
`{4: right, 5: left}`. Layer-specific mappings must be audited and reconciled
before explanation labels can be considered authoritative.

## 3. RTMPose And Pose Kinematics

**Sources**: `rtmpose_model`, `PoseKinematicsRecord`, and
`backend/apps/pipeline/services/pose_kinematics.py`.

### Raw And Existing Derived Signals

- 17 COCO keypoint coordinates and confidence scores;
- missing/low-confidence keypoints, estimated points, anchors, and pose quality;
- joint angles, limb vectors, torso/head yaw, pitch, roll, and alignment;
- posture, gesture, attention support, and physical-constraint violations;
- head/wrist velocity, pose stability, sudden jumps, spike/transition state;
- model-versus-kinematics support, contradiction, fusion, and override evidence.

### Additional Derived Signals And Tasks

- per-joint sensitivity/contribution to each semantic output;
- left/right symmetry, range-of-motion trends, occlusion topology, and
  keypoint-quality coverage;
- bounded temporal derivatives and motion phase segmentation;
- pose confidence calibration and posture/orientation confusion analysis;
- counterfactual joint movement required to change a semantic state.

### Explanation Strategy

- Fast: geometric derivation trace, joint support, quality/missingness,
  temporal continuity, contradiction, and counterfactual thresholds.
- Deep on-demand: keypoint/heatmap attribution when available, joint occlusion
  perturbation, and stability/sanity checks.

## 4. Tracking And OSNet ReID

**Sources**: `StudentTrack`, `FrameEmbedding`, OSNet ReID client, lifecycle
events, tracking metrics.

### Raw And Derived Signals

- embedding vector and provenance, cosine similarity, threshold, margin,
  identity confidence, match decision, and candidate count;
- lifecycle state, visible/lost/reidentified intervals, aliases, track age,
  fragmentation, merge collision, and unresolved association;
- motion consistency, temporal gap, appearance consistency, and boundary
  tracklet quality.

### Explanation Strategy

- Fast: nearest accepted/rejected candidate distances, decision threshold,
  ambiguity margin, lifecycle/temporal support, and unresolved reasons.
- Deep on-demand: governed nearest prototypes/exemplars and counterfactual
  candidate comparison. Individual embedding dimensions are not presented as
  human-semantic causes unless separately validated.

### Identity Rule

Local tracker labels are source-scoped opaque values. Raw numeric/string label
equality is never cross-run identity proof.

## 5. YOLOE Scene Segmentation

**Sources**: `yoloe_scene_seg`, `SceneFrameSummary`,
`SceneObjectObservation`, and scene artifacts.

### Raw And Existing Derived Signals

- prompt profile/class order, class ID/name, confidence, box, centroid, mask,
  mask area, source shape, raw output references, and unavailable fields;
- people count, detector contradiction, non-ROI overlap, recovery eligibility,
  duplicate/false-alarm/missing/unresolved states, and assignment evidence.

### Additional Derived Signals And Tasks

- mask compactness, boundary stability, temporal mask persistence, object
  appearance/disappearance, and scene-layout change;
- person-to-object proximity, desk/seat association, occlusion support, and
  contradiction persistence;
- prompt/profile sensitivity and route artifact mismatch diagnostics.

### Explanation Strategy

- Fast: prompt/artifact lineage, score, mask/box support, overlap and count
  contradictions, temporal support, and threshold delta.
- Deep on-demand: D-RISE or architecture-valid segmentation attribution,
  mask deletion/insertion fidelity, and prompt-sensitivity probes.

## 6. Spatial Relationship Vectorization Layer

**Sources**: `SpatialRelationFrame`, `scene/srvl.py`, and `scene/srvl_maps.py`.

### Existing Signals

- ordered object set, normalized pairwise distance, angle, magnitude-angle
  vector, optional Cartesian vector, object count, heatmap, direction map, and
  correlation map.

### Additional Derived Signals And Tasks

- per-student nearest-object/nearest-peer distance, rank, and change rate;
- pairwise approach/withdrawal velocity, relation persistence, topology change,
  and local density;
- spatial graph degree, centrality proxy, cluster change, and scene-relative
  position;
- matrix-change magnitude and localized contribution to a contextual score.

### Explanation Strategy

- Fast: exact pairwise values, deltas, graph edges, context contribution, and
  object/identity lineage.
- Deep path is not required; SRVL is deterministic and should be explained by
  exact computation and counterfactual geometry.

## 7. BSIL Semantic, Temporal, Episode, And Interaction Signals

**Sources**: `apps.behavior` models and services.

### Existing Signals

- feature windows, semantic pose states, confidence breakdowns, behavioral
  states, episodes, interaction edges, anomaly primitives, and decision lineage;
- primitives: change point, drift, repeated pattern, instability, and attention
  deviation;
- confidence bands, truth states, uncertainty, temporal decay, hysteresis,
  cooldown, and invalid-window evidence.

### Additional Derived Signals And Tasks

- per-feature coverage, support duration, contradiction ratio, evidence age,
  and confidence trend;
- episode-level contribution summary and threshold counterfactual;
- interaction evidence strength separated from individual behavior evidence;
- explanation completeness and evidence freshness.

### Explanation Strategy

Deterministic derivation and exact feature/rule contribution are the primary
explanation. Heuristic-only outputs remain labeled heuristic and cannot be
presented as model truth.

## 8. Observed-Pattern, Anomaly, Calibration, And Review Signals

**Sources**: `apps.anomalies`, valid evidence envelopes, signal-pattern windows,
observed-pattern profiles, optional source-model calibration artifacts, and
non-ground-truth reviewer assessments.

### Existing Signals

- baseline means/stddev/medians/ranges/sample counts, lineage, drift state;
- threshold context, threshold shift, hysteresis state, z-score-like surprise;
- baseline quarantine, governed reviewer assessments, and rollback target.

### Required New Signals And Tasks

- per-student bounded multivariate signal-pattern window with feature coverage,
  transition rates, recurrence, persistence, volatility, cross-signal
  correlation, contradiction, and identity-continuity features;
- observed-pattern profile robust medians/MADs/quantiles, bounded rolling/EWMA
  statistics, sample age/count, cold-start, contamination, drift, quarantine,
  compatibility, and expiry states;
- per-signal pattern-deviation magnitude and threshold/counterfactual delta;
- optional source-model expected calibration error, Brier/task score,
  calibration sample coverage, and calibration age only where task-appropriate
  held-out evidence exists;
- conformal p-value/set/interval only where calibration assumptions are valid;
- evidence coverage, reliability, uncertainty penalty, context support,
  contradiction penalty, and final review-priority decomposition;
- score withholding reason, explanation completeness, and reviewer disagreement;
- controlled-fixture, metamorphic/invariant, sensitivity, cold-start,
  contamination, drift, quarantine, and threshold impact analysis.

### Knowledge Limits

- `within_observed_pattern` does not mean normal, innocent, or non-cheating.
- `pattern_deviation` does not mean abnormal intent, misconduct, or cheating.
- Reviewer assessments, heuristic outputs, model agreement, and assumed-normal
  history are not ground truth or training/fine-tuning targets.

## 9. Runtime, Quality, And Operational Signals

**Sources**: telemetry, queues, runtime policies, frames, jobs, artifacts.

### Signals

- source/ingest/queue/inference/persistence timestamps and latency;
- dropped frames, temporal gaps, stale windows, reconnect state, queue depth,
  backpressure, retries, deadlines, and reconciler outcomes;
- model RTT/status/input shape/output bytes, phase timing, GPU/VRAM, CPU/RSS,
  PostgreSQL/Redis latency, artifact sizes, and renderer status;
- route snapshot, environment fingerprint, model/artifact digest, feature flags,
  calibration snapshot, and data-quality state.

### Explanation Strategy

Operational factors are first-class explanation context. A high anomaly score
that coincides with degraded runtime, poor input quality, route drift, or stale
evidence must be degraded or withheld.

## 10. Student Interaction Graph Signals

**Sources**: `SpatialRelationFrame` and `scene/srvl.py` (pairwise distance/angle),
`InteractionEdge` (`apps.behavior`), `person_detector` tracks, `rtmpose_model`
head/torso orientation, gaze models, the COMBINED `yolo11m` head (gaze
direction), and `yoloe_scene_seg` (shared seat/zone/object). The production graph
is a **deterministic** consolidation of these existing relational signals; it is
not a trained model. Learned graph methods are governed `PROBE_ONLY` per
[pretrained-models-registry.md](pretrained-models-registry.md) and never drive a
production score.

### Graph Construction (deterministic)

- nodes are tracked students/teacher with source-scoped, identity-gated labels;
- candidate edges are built only between resolved identities; an edge touching an
  ambiguous/unresolved identity is emitted as `unresolved` and excluded from
  valid per-student features;
- edge types and their evidence:
  - `proximity` — normalized pairwise distance and approach/withdrawal velocity
    from SRVL/detector centroids;
  - `gaze_toward_peer` (directed) — peer falls inside the gaze/head-orientation
    cone using gaze yaw/pitch + `head_yaw`; `mutual_gaze` when both directions hold;
  - `orientation_toward_peer` — relative torso/body orientation and lean;
  - `co_movement` — bounded temporal correlation of motion/transition events;
  - `shared_scene` — shared desk/seat/zone/object from scene segmentation
    (context only; not asserted as interaction).
- each edge carries `edge_type`, `weight`, `persistence`, `identity_confidence`,
  `truth_state`, and source/evidence lineage; node/edge counts are
  configuration-bounded.

### Derived Per-Student Graph Signals And Tasks

- interaction degree, weighted degree, and per-edge-type degree;
- mutual-gaze count/dwell and directed gaze-toward-peer duration;
- proximity dwell and nearest-peer approach/withdrawal rate;
- edge persistence, neighbor churn, and dyad-specific interaction strength;
- local clustering coefficient and bounded centrality/betweenness proxy;
- graph-topology change magnitude and synchronized-transition support;
- per-feature coverage, identity-continuity gating, and contradiction with
  individual-behavior evidence.

These per-student features are registered governed signals (FR-035) that feed the
bounded multivariate observed-pattern profiles in §8; a graph-feature deviation
raises only that student's own review priority and never the peer's.

### Explanation Strategy

- Fast: exact edge list with type/weight/persistence/identity-confidence, the
  per-student feature derivation trace, source lineage, and counterfactual graph
  deltas. SRVL/graph geometry is deterministic and requires no deep method.
- Deep path is not required for the production deterministic graph; any learned
  graph embedding is `PROBE_ONLY` research only.

### Knowledge Limits And Streaming

- An edge or graph feature is relational context, never proof of collusion,
  cheating, or intent.
- `streaming_compatibility`: `stream-safe-with-config` using bounded graph
  windows, node/edge caps, and stale-edge eviction.

## 11. Candidate Additional Models And Strategies

The production anomaly path is deterministic robust observed-pattern analysis.
The following are research candidates, not pre-approved production
dependencies:

- robust statistics plus bounded hysteresis as the first score baseline;
- conformal calibration for valid exchangeable or adaptively calibrated paths;
- Half-Space Trees, Deep SVDD, PatchCore, and PaDiM only as future
  `HYPOTHESIS_ONLY` or `PROBE_ONLY` candidates under a separate governed plan;
- prototype/case-based explanations for reviewer diagnosis;
- D-RISE/Eigen-CAM/Integrated Gradients only where model architecture,
  fidelity, and sanity checks support them;
- SHAP only for supported tabular/fusion models or offline audit, not as a
  universal explanation method.

Every candidate requires its own atomic cycle or explicit subtask, baseline,
quality evaluation, production benchmark, rollback, and decision.
No learned anomaly candidate may be trained/fine-tuned or promoted under the
current no-ground-truth plan.

## Source References

- `backend/apps/pipeline/services/model_route_service.py`
- `backend/apps/pipeline/multi_model.py`
- `backend/apps/pipeline/services/pose_kinematics.py`
- `backend/apps/video_analysis/models.py`
- `backend/apps/video_analysis/scene/normalizer.py`
- `backend/apps/video_analysis/scene/srvl.py`
- `backend/apps/video_analysis/scene/srvl_maps.py`
- `backend/apps/behavior/models.py`
- `backend/apps/behavior/anomaly_primitives.py`
- `backend/apps/anomalies/bsil_baselines.py`
- `backend/apps/anomalies/bsil_drift.py`
- `backend/apps/anomalies/bsil_thresholds.py`
- `backend/apps/telemetry/models.py`
