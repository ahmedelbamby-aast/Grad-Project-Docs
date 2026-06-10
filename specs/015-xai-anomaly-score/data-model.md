# Data Model: Explainable Behavioral Evidence And Anomaly Scoring

**Date**: 2026-06-08
**Storage authority**: PostgreSQL for relational state; digest-addressed
job-scoped artifacts for large arrays/images/traces.

## Design Rules

- Reuse existing `apps.behavior`, `apps.anomalies`, `apps.video_analysis`, and
  `apps.telemetry` records instead of duplicating source evidence.
- New XAI records reference source rows/artifacts through stable lineage
  references and digests.
- Every record has an explicit schema version, truth state, creation time,
  route snapshot, idempotency key, and reconstruction lineage where applicable.
- Large saliency maps, raw dense outputs, calibration arrays, and generated
  figures are artifact-backed rather than embedded in unbounded relational JSON.
- Missingness and unavailable reasons are explicit.
- Production records are append-only after publication; supersession creates a
  new record and preserves the prior record.
- Reviewer feedback never directly mutates a baseline, threshold, score, or
  model.
- No record represents anomaly, cheating, non-cheating, abnormality, or
  normality ground truth. No reviewer assessment or observed history may be a
  hidden anomaly-model training/fine-tuning target.
- All pretrained models are FROZEN and catalogued in
  [pretrained-models-registry.md](pretrained-models-registry.md); each consumed
  model is captured via `XAIModelRouteSnapshot` lineage. Class B (`PROBE_ONLY`)
  model outputs are never persisted as a production `review_priority_score` or
  `pattern_state`.
- `within_observed_pattern` and `pattern_deviation` describe comparison with a
  versioned signal-pattern profile only; they are not behavioral verdicts.
- Student review-priority results remain student-scoped. Any session/classroom
  escalation derived from reviewer-approved deviation is a separate aggregate
  output and never mutates peer students' scores, contributions, or pattern
  states.
- The student-interaction graph is deterministic, identity-gated relational
  evidence derived from existing source signals; an edge or graph feature is
  context only, never a collusion/cheating record, and a learned graph model
  output is never persisted as a production `review_priority_score` or
  `pattern_state` (it stays `PROBE_ONLY` per
  [pretrained-models-registry.md](pretrained-models-registry.md)).
- The general baseline is the same `ObservedPatternProfileSnapshot` machinery at a
  population/context tier (`baseline_tier=general`), **computed across many
  students/sessions and never from or equal to a single student's profile**. It is
  assumed-normal and contamination-aware, never known-normal ground truth. It
  supplies **General Boundaries**; the student-tier profile supplies **Local
  Boundaries** (time-window + cold-start dependent). A score compares against both
  and may also emit a general classroom-level deviation from the General
  Boundaries, distinct from any individual.
- No operational value is hardcoded. Every threshold, weight, envelope bound,
  geometric constant, and gate is bound through a `ParameterProvenanceRecord`
  tagged `learned` (with the baseline snapshot it was derived from) or
  `configured` (with a fingerprinted `.env`/config key); records reference the
  provenance of every value they used so XAI can reconstruct it.
- A model used in production must hold a current `MANDATORY` `ModelPromotionRecord`;
  `PROBE_ONLY`/`SHADOW`/`CANARY` outputs are never persisted as a production
  `review_priority_score` or `pattern_state`, and no promotion record stores a
  behavioral accuracy/AUROC metric.
- Probe fine-tuning operates only on copies: a `ProbeFineTuneRun` references the
  frozen parent and the new child artifact digests; filtered inferences used for
  fine-tuning are persisted as an accept/refuse/edit ledger and are never labeled
  ground truth.

## Existing Entities Reused

| Existing entity | Owner | Role |
|---|---|---|
| `VideoAnalysisJob`, `Frame`, `Detection`, `BoundingBox` | `apps.video_analysis` | Source job/frame/model evidence |
| `StudentTrack`, `FrameEmbedding` | `apps.video_analysis` | Identity/ReID source evidence |
| `PoseKinematicsRecord` | `apps.video_analysis` | Pose geometry and quality |
| `SceneFrameSummary`, `SceneObjectObservation`, `SpatialRelationFrame` | `apps.video_analysis` | Scene and spatial evidence |
| `BehaviorFeatureWindow`, `AnomalyPrimitiveEvent`, `SemanticPoseState` | `apps.behavior` | Semantic/temporal source evidence |
| `BehavioralState`, `BehavioralEpisode`, `InteractionEdge` | `apps.behavior` | Higher-level behavioral evidence |
| `DecisionLineageRecord`, `BehavioralReviewLabel` | `apps.behavior` | Existing lineage/review evidence; never anomaly ground truth |
| `AnomalyEvent`, `BSILBaselineSnapshot`, `BSILThresholdShift`, `BSILReviewLabelRecord` | `apps.anomalies` | Existing anomaly/baseline governance; never anomaly ground truth |
| `TelemetryModelCall` and telemetry records | `apps.telemetry` | Runtime evidence |

## New Entity: XAIModelRouteSnapshot

**Owner**: `apps.pipeline` for capture; persisted/read through a neutral
versioned contract.

**Purpose**: Immutable point-in-time truth for all model routes and relevant
runtime configuration used by an explanation/score.

| Field | Type | Rule |
|---|---|---|
| `snapshot_id` | UUID | Primary key |
| `schema_version` | string | Required |
| `environment` | string | `dev`, `staging`, or `prod` |
| `runtime_profile` | string | `live` or `offline` |
| `routes` | bounded JSON | Logical task to model/version/artifact digest |
| `config_fingerprint` | SHA-256 | Required |
| `model_repository_fingerprint` | SHA-256 | Required for production-valid state |
| `git_sha` | string | Required for production-valid state |
| `created_at` | timestamp | Immutable |
| `snapshot_digest` | SHA-256 | Unique reconstruction digest |
| `truth_state` | enum | Validity of capture/reconciliation |
| `unavailable_reasons` | bounded JSON | Explicit missing runtime truth |

**Constraints**:

- `snapshot_digest` unique.
- No update after publication.
- Production-valid explanations/scores require `truth_state=valid`.

## New Entity: XAISignalDefinition

**Owner**: `apps.behavior.explainability`.

**Purpose**: Governed registry entry for a raw or derived signal.

| Field | Type | Rule |
|---|---|---|
| `signal_key` | string | Stable primary identifier |
| `schema_version` | string | Part of semantic compatibility |
| `owner` | string | Existing owning module |
| `source_kind` | enum | Model, record, artifact, deterministic derivation |
| `scope` | enum | Frame, observation, track, window, scene, session, runtime |
| `unit` | string | Required |
| `value_contract` | bounded JSON | Shape/type/range |
| `validity_rule` | bounded JSON | Required |
| `missingness_rule` | bounded JSON | Required |
| `calibration_required` | boolean | Required |
| `fast_explainer_key` | string | Registered adapter |
| `deep_explainer_keys` | bounded array | Allowlisted adapters |
| `streaming_compatibility` | enum | Required |
| `retention_policy_key` | string | Required |
| `definition_digest` | SHA-256 | Unique |
| `enabled` | boolean | Configuration-governed |

**Constraints**:

- Active signal key/version must resolve to one definition.
- Active production route signals require a fast explainer.

## New Entity: XAIEvidenceEnvelope

**Owner**: `apps.behavior.explainability`.

**Purpose**: Common normalized evidence unit consumed by explainers and scoring.

| Field | Type | Rule |
|---|---|---|
| `envelope_id` | UUID | Primary key |
| `schema_version` | string | Required |
| `idempotency_key` | string | Unique per source/signal/version |
| `job_ref`, `frame_ref`, `track_ref` | nullable references | Source scope |
| `signal_key`, `signal_version` | string | Must resolve to registry |
| `route_snapshot_ref` | FK | Required |
| `source_refs` | bounded array | Required, nonempty for valid state |
| `source_start_ms`, `source_end_ms` | integer | Ordered |
| `timestamps` | bounded JSON | Source/ingest/queue/inference/persistence |
| `identity_scope` | bounded JSON | Local/canonical/ReID/lifecycle truth |
| `value_summary` | bounded JSON | Compact value; no unbounded dense arrays |
| `quality` | bounded JSON | Coverage/reliability/input quality |
| `missingness` | bounded JSON | Explicit missing/unavailable fields |
| `contradictions` | bounded JSON | Source-preserving conflicts |
| `truth_state`, `confidence_band` | enums | Required |
| `lineage_digest` | SHA-256 | Required |
| `created_at` | timestamp | Immutable |

**Constraints**:

- `source_end_ms >= source_start_ms`.
- Valid state requires nonempty source references and valid route snapshot.
- Payload-size guard at persistence boundary.
- Re-run with same idempotency key returns the existing record.

## New Entity: SignalPatternWindow

**Owner**: `apps.behavior.explainability`.

**Purpose**: Bounded per-student multivariate temporal feature window derived
from all compatible valid evidence.

| Field | Type | Rule |
|---|---|---|
| `window_id` | UUID | Primary key |
| `schema_version`, `feature_schema_version` | strings | Required |
| `job_ref`, `track_ref` | references | Required scope |
| `source_start_ms`, `source_end_ms` | integers | Required |
| `route_snapshot_ref` | FK | Required |
| `evidence_refs` | bounded array | Required |
| `feature_vector_ref` | artifact/value ref | Bounded, digest-addressed |
| `feature_validity` | bounded JSON | Missing/degraded/valid per feature |
| `identity_continuity` | bounded JSON | Required |
| `truth_state` | enum | Required |
| `window_digest` | SHA-256 | Unique |
| `created_at` | timestamp | Immutable |

**Constraints**:

- Features retain source, unit, timestamp, and validity lineage.
- Missing/invalid features are not encoded as normal or zero evidence.
- Window length/count and vector size are configuration-bounded.

## New Entity: ObservedPatternProfileSnapshot

**Owner**: `apps.anomalies.scoring`.

**Purpose**: Versioned contamination-aware summary of valid observed signal
patterns for one compatible scope. It is never known-normal ground truth.

| Field | Type | Rule |
|---|---|---|
| `profile_id` | UUID | Primary key |
| `schema_version`, `feature_schema_version` | strings | Required |
| `scope` | bounded JSON | Student/session/camera/scene/context/population |
| `baseline_tier` | enum | `student`, `session`, `camera`, `scene`, `context`, or `general` (population, corpus-ingested) |
| `route_compatibility` | bounded JSON | Required |
| `method_profile` | bounded JSON | Robust statistics and configured bounds |
| `source_window_refs` | bounded artifact/reference set | Required |
| `valid_window_count` | integer | Required |
| `quarantined_window_count` | integer | Required |
| `contamination_state`, `drift_state` | enums | Required |
| `valid_from`, `expires_at` | timestamps | Required |
| `truth_state` | enum | Valid/degraded/quarantined/invalidated |
| `knowledge_limits` | bounded array | Includes `not_known_normal_truth` |
| `profile_digest` | SHA-256 | Unique |
| `supersedes_ref` | nullable FK | Append-only evolution |

**Constraints**:

- Profile creation/update is deterministic, bounded, and append-only.
- Cold-start remains `insufficient_context` until minimum valid windows exist.
- High-deviation, disputed, drifting, invalid, or low-quality windows are
  quarantined from updates.
- No profile field may be described or consumed as behavioral ground truth.
- Route, ontology, feature-schema, or artifact incompatibility invalidates use.
- The `general` tier is learned by corpus ingestion under identical robust,
  contamination-aware, append-only, cold-start, quarantine, and drift governance;
  it stays assumed-normal and is never a per-student verdict.
- The `general` tier MUST aggregate at least the configured minimum number of
  distinct students/sessions; it is never derived from, or equal to, a single
  student's windows. It supplies General Boundaries; student-tier profiles supply
  Local Boundaries.

## New Entity: CalibrationSnapshot

**Owner**: `apps.anomalies.scoring`.

**Purpose**: Versioned calibration model and evaluation for one compatible
route/output/slice.

| Field | Type | Rule |
|---|---|---|
| `calibration_id` | UUID | Primary key |
| `schema_version` | string | Required |
| `route_snapshot_compatibility` | bounded JSON | Model/output/artifact constraints |
| `signal_key` | string | Required |
| `method` | string | Temperature, isotonic, Platt, etc. |
| `parameters_artifact_ref` | artifact ref | Digest-addressed |
| `evidence_cohort_manifest_ref` | artifact ref | Required when fitting |
| `sample_count` | integer | Required |
| `evaluation` | bounded JSON | ECE/Brier/task metrics/CIs/subgroups |
| `valid_from`, `expires_at` | timestamps | Required |
| `truth_state` | enum | Valid/degraded/invalidated |
| `snapshot_digest` | SHA-256 | Unique |
| `supersedes_ref` | nullable FK | Append-only evolution |

**Constraints**:

- Lookup rejects incompatible, expired, underpowered, or invalid snapshots.
- Fitting is offline-only; lookup may be stream-safe.
- This entity calibrates existing source-model outputs only where
  task-appropriate held-out evidence exists. It is not anomaly ground truth.

## New Entity: ConformalCalibrationSnapshot

**Owner**: `apps.anomalies.scoring`.

**Purpose**: Versioned conformal calibration, assumptions, and measured
coverage.

| Field | Type | Rule |
|---|---|---|
| `conformal_id` | UUID | Primary key |
| `signal_key` or `score_profile_key` | string | Required |
| `alpha` | decimal | `0 < alpha < 1` |
| `method` | string | Required |
| `assumptions` | bounded JSON | Required |
| `calibration_manifest_ref` | artifact ref | Required |
| `coverage_evaluation` | bounded JSON | Required |
| `drift_policy` | bounded JSON | Required |
| `truth_state` | enum | Required |
| `snapshot_digest` | SHA-256 | Unique |
| `valid_from`, `expires_at` | timestamps | Required |

**Constraints**:

- No conformal output when truth state or assumptions fail.

## New Entity: AnomalyScoreRecord

**Owner**: `apps.anomalies.scoring`.

**Purpose**: Reconstructable, non-accusatory review-priority result.

| Field | Type | Rule |
|---|---|---|
| `score_id` | UUID | Primary key |
| `schema_version` | string | Required |
| `idempotency_key` | string | Unique |
| `job_ref`, `track_ref` | references | Required scope |
| `source_start_ms`, `source_end_ms` | integer | Required |
| `route_snapshot_ref` | FK | Required |
| `baseline_snapshot_ref` | nullable FK/lineage ref | Optional legacy/auxiliary lineage only |
| `pattern_profile_ref` | FK | Required for valid score (student tier) |
| `general_baseline_ref` | nullable FK | Compatible general-baseline tier snapshot for dual comparison |
| `pattern_window_ref` | FK | Required for valid score |
| `deviation_vs_self` | decimal nullable | Summary deviation vs the student's own profile |
| `deviation_vs_population` | decimal nullable | Summary deviation vs the general baseline |
| `parameter_provenance_refs` | bounded array | Provenance of every operational value used |
| `calibration_refs` | bounded array | Required where calibrated |
| `score_profile_key/version` | strings | Required |
| `review_priority_score` | decimal nullable | 0-100 or withheld |
| `review_priority_band` | enum | Non-accusatory |
| `pattern_state` | enum | Within observed pattern/deviation/insufficient/withheld |
| `ground_truth_status` | enum | Always `unavailable_for_anomaly_behavior` in this plan |
| `evidence_coverage` | decimal | 0-1 |
| `reliability` | decimal | 0-1 |
| `uncertainty` | decimal | 0-1 |
| `context_support` | decimal | 0-1 |
| `persistence_support` | decimal | 0-1 |
| `truth_state` | enum | Required |
| `withheld_reasons` | bounded array | Required when score null |
| `contradiction_summary` | bounded JSON | Required |
| `reconstruction_digest` | SHA-256 | Unique |
| `created_at` | timestamp | Immutable |

**Constraints**:

- User-facing/persisted description cannot contain prohibited accusation terms.
- Valid numeric score requires all configured validity gates.
- Score is append-only and superseded by a new record.
- Numeric scores require compatible pattern window/profile references.
- `pattern_state` cannot be converted into a cheating/non-cheating or
  normal/abnormal-behavior verdict.
- Reviewer approval for one student may annotate that student's review context
  or a separate session aggregate, but it must not mutate any peer student's
  `AnomalyScoreRecord`.
- A valid score requires the student-tier profile; the general baseline is
  contextual and, when present, the score exposes `deviation_vs_self` and
  `deviation_vs_population` separately. `deviation_vs_self` uses the student's
  Local Boundaries; `deviation_vs_population` uses the General Boundaries
  (population baseline, never a peer student). Every operational value used
  resolves to a `ParameterProvenanceRecord`; no hardcoded constant is permitted.

## New Derived Entity: SessionReviewPriorityAggregate

**Owner**: `apps.anomalies.scoring`.

**Purpose**: Separate session/classroom-level aggregate derived from
student-scoped review-priority summaries.

| Field | Type | Rule |
|---|---|---|
| `scope_ref` | bounded reference | Session/classroom/camera/job scope |
| `student_count` | integer | Required |
| `base_mean_score` | decimal | Mean of individual student scores before session lift |
| `approved_deviation_count` | integer | Count of reviewer-approved student deviations with valid truth state |
| `approved_deviation_boost` | decimal | Separate session-only lift term |
| `session_review_priority_score` | decimal | 0-100 aggregate after boost and clamp |
| `general_classroom_deviation` | decimal nullable | Classroom-level deviation versus the General Boundaries (population baseline), distinct from any individual |
| `students` | bounded summary array | Student-local summaries with no peer mutation |

**Constraints**:

- `session_review_priority_score` is derived from student-local summaries and a
  configured session aggregation policy.
- Reviewer-approved deviation for one student may increase only
  `approved_deviation_count`, `approved_deviation_boost`, and
  `session_review_priority_score`; it must not relabel or rescore any peer
  student summary.
- A student's own neutral default remains `within_observed_pattern` or
  `insufficient_context` until valid evidence or a governed reviewer-approved
  deviation changes that student's own review context.
- `general_classroom_deviation` is derived only from the General Boundaries
  (population baseline) and describes the classroom aggregate; it is never
  attributed to, or substituted for, an individual student's deviation.

## New Entity: AnomalyScoreContribution

**Owner**: `apps.anomalies.scoring`.

**Purpose**: Exact per-signal/context contribution to one score.

| Field | Type | Rule |
|---|---|---|
| `contribution_id` | UUID | Primary key |
| `score_ref` | FK | Required |
| `evidence_envelope_ref` | FK | Required |
| `component_key` | string | Required |
| `pattern_deviation_magnitude` | decimal | Required for valid component |
| `deviation_vs_self` | decimal nullable | Deviation vs the student's own profile |
| `deviation_vs_population` | decimal nullable | Deviation vs the general baseline |
| `reliability` | decimal | Required |
| `temporal_support` | decimal | Required |
| `configured_weight` | decimal | Required |
| `parameter_provenance_ref` | FK | Provenance of the weight/threshold used (`learned`/`configured`) |
| `effective_contribution` | decimal nullable | Null when invalid/missing |
| `validity_state` | enum | Valid/missing/degraded/suppressed |
| `reason_codes` | bounded array | Required |
| `counterfactual_delta` | bounded JSON | Threshold/recovery delta |
| `contribution_digest` | SHA-256 | Unique within score |

**Constraints**:

- Invalid/missing contribution remains explicit and is not converted to zero.
- Sum/reconstruction tests must reproduce the parent score.

## New Entity: XAIAttributionArtifact

**Owner**: `apps.behavior.explainability`.

**Purpose**: Large or visual deep-XAI output plus execution lineage.

| Field | Type | Rule |
|---|---|---|
| `artifact_id` | UUID | Primary key |
| `request_id` | UUID/string | Required |
| `evidence_envelope_ref` | FK | Required |
| `route_snapshot_ref` | FK | Required |
| `method_key/version` | strings | Required |
| `parameters` | bounded JSON | Required |
| `artifact_ref` | authenticated artifact ref | Required if completed |
| `artifact_digest`, `size_bytes` | values | Required if completed |
| `evaluation_ref` | FK | Required for production-valid state |
| `status` | enum | Queued/running/completed/failed/expired |
| `truth_state` | enum | Required |
| `unavailable_reason` | string | Required when absent |
| `started_at`, `completed_at` | timestamps | Lifecycle |
| `idempotency_key` | string | Unique |

**Constraints**:

- Artifact-size guard and deadline.
- Job-scoped access and audit.
- Completed does not imply production-valid; evaluation decides validity.

## New Entity: ExplanationEvaluationRecord

**Owner**: `apps.behavior.explainability`.

**Purpose**: Method-specific fidelity, stability, sanity, cost, and acceptance
evaluation.

| Field | Type | Rule |
|---|---|---|
| `evaluation_id` | UUID | Primary key |
| `artifact_ref` or `explanation_ref` | reference | Required |
| `method_key/version` | strings | Required |
| `fidelity_metrics` | bounded JSON | Required or unavailable reason |
| `stability_metrics` | bounded JSON | Required or unavailable reason |
| `sanity_metrics` | bounded JSON | Required or unavailable reason |
| `latency_resource_metrics` | bounded JSON | Required |
| `decision` | enum | Valid/degraded/invalidated/probe-only |
| `reason_codes` | bounded array | Required |
| `evaluation_digest` | SHA-256 | Unique |

## New Entity: XAIExplanationRecord

**Owner**: `apps.behavior.explainability`.

**Purpose**: Composed machine-readable and user-facing explanation.

| Field | Type | Rule |
|---|---|---|
| `explanation_id` | UUID | Primary key |
| `schema_version` | string | Required |
| `scope` | bounded JSON | Job/frame/track/window |
| `route_snapshot_ref` | FK | Required |
| `score_ref` | nullable FK | Optional for model-only explanation |
| `evidence_refs` | bounded array | Required |
| `attribution_refs` | bounded array | Optional |
| `summary_code` | string | Non-accusatory stable code |
| `summary_text` | string | Vocabulary-gated |
| `supporting_factors` | bounded JSON | Required |
| `contradicting_factors` | bounded JSON | Required |
| `missing_factors` | bounded JSON | Required |
| `counterfactuals` | bounded JSON | Required when available |
| `knowledge_limits` | bounded JSON | Required |
| `truth_state`, `confidence_band` | enums | Required |
| `reconstruction_digest` | SHA-256 | Unique |
| `supersedes_ref` | nullable FK | Append-only evolution |

## New Entity: XAIReviewFeedback

**Owner**: `apps.anomalies` with audit integration.

**Purpose**: Governed reviewer assessment of score/explanation/evidence.

| Field | Type | Rule |
|---|---|---|
| `feedback_id` | UUID | Primary key |
| `reviewer_ref` | authenticated user ref | Required |
| `score_ref`, `explanation_ref` | references | At least one |
| `assessment` | governed enum | Operational review assessment, not truth label |
| `reason_codes`, `notes` | bounded fields | Required/optional |
| `evidence_refs` | bounded array | Required |
| `created_at` | timestamp | Immutable |
| `feedback_digest` | SHA-256 | Unique |
| `promotion_state` | enum | Evaluation-only by default |

**Constraints**:

- No direct production threshold/baseline/model mutation.
- No anomaly-model training/fine-tuning, ground-truth, or direct
  pattern-profile update use.
- Access and write audit required.

## New Entity: XAIRendererTelemetryRecord

**Owner**: `apps.telemetry`.

**Purpose**: Measure shared WebGL renderer behavior.

| Field | Type | Rule |
|---|---|---|
| `session_ref`, `job_ref` | references | Scope |
| `renderer_version`, `backend` | strings | `webgl2` required for accepted figure |
| `view_key` | string | Required |
| `frame_time_p50/p95/p99_ms` | decimals | Required |
| `update_latency_p50/p95/p99_ms` | decimals | Required |
| `context_count`, `context_loss_count` | integers | Required |
| `vertices_or_cells`, `upload_bytes`, `dropped_updates` | integers | Required |
| `memory_proxy_bytes` | integer nullable | Explicit unavailable reason if absent |
| `truth_state`, `unavailable_reasons` | fields | Required |

## New Entity: StudentInteractionGraphFrame

**Owner**: `apps.behavior.explainability` (deterministic graph builder).

**Purpose**: Bounded, identity-gated per-frame/per-window graph of student
relational evidence plus the per-student graph-derived features it produces. It
is relational context, never collusion or cheating truth.

| Field | Type | Rule |
|---|---|---|
| `graph_frame_id` | UUID | Primary key |
| `schema_version`, `feature_schema_version` | strings | Required |
| `job_ref` | reference | Required scope |
| `scope` | bounded JSON | Session/camera/scene |
| `source_start_ms`, `source_end_ms` | integers | Ordered |
| `route_snapshot_ref` | FK | Required |
| `node_refs` | bounded array | Resolved, identity-gated track refs |
| `edges` | bounded JSON array | `edge_type`, src/dst track, `directed`, `weight`, `persistence`, `identity_confidence`, `truth_state`, `evidence_refs` |
| `per_student_features_ref` | artifact/value ref | Digest-addressed bounded graph features |
| `node_count`, `edge_count`, `unresolved_edge_count` | integers | Required |
| `bounds` | bounded JSON | Configured node/edge caps and overflow policy |
| `source_signal_refs` | bounded array | SRVL/behavior/scene/pose/gaze lineage |
| `truth_state` | enum | Required |
| `graph_digest` | SHA-256 | Unique reconstruction digest |
| `created_at` | timestamp | Immutable |

**Constraints**:

- Edges are built only between resolved identities; an edge touching an
  ambiguous/unresolved identity is recorded `unresolved` and excluded from valid
  per-student features (it is not fabricated and not counted as zero).
- Node/edge counts are configuration-bounded; overflow uses bounded sampling and
  is logged, never silently truncated.
- Construction is deterministic and reconstructable from `source_signal_refs`.
- Per-student features feed `SignalPatternWindow` for the producing student only
  and never mutate a peer student's records.
- No field is described or consumed as collusion/cheating ground truth; learned
  graph-model outputs are not persisted here as production scores or states.
- Frames are produced incrementally per frame/window so a live, real-time
  node-link plot updates continuously (latest-frame-wins); the live graph stays
  bounded by `bounds` and stale-edge eviction over indefinite streams.

## New Entity: ParameterProvenanceRecord

**Owner**: `apps.anomalies.scoring` (parameter resolver).

**Purpose**: Bind every operational value to a verifiable source so that no value
is hardcoded and every value is reconstructable for XAI.

| Field | Type | Rule |
|---|---|---|
| `provenance_id` | UUID | Primary key |
| `schema_version` | string | Required |
| `param_key` | string | Stable key (e.g. `score.weight.head_yaw`, `geom.gaze_cone_deg`) |
| `value` | bounded JSON | The resolved value/bound |
| `source_kind` | enum | `learned` or `configured` |
| `learned_source_ref` | nullable FK | Baseline snapshot the value was derived from (when `learned`) |
| `config_source_key` | nullable string | `.env`/config key (when `configured`) |
| `config_fingerprint` | nullable SHA-256 | Required when `configured` |
| `scope` | bounded JSON | Where the binding applies |
| `valid_from`, `expires_at` | timestamps | Required |
| `provenance_digest` | SHA-256 | Unique |
| `created_at` | timestamp | Immutable |

**Constraints**:

- Exactly one source is set per `source_kind` (`learned` → `learned_source_ref`;
  `configured` → `config_source_key` + `config_fingerprint`).
- A `learned` value must be reconstructable from the referenced baseline snapshot.
- An operational value without a provenance record is a hardcoding violation and
  fails acceptance.

## New Entity: ModelPromotionRecord

**Owner**: `apps.pipeline` (promotion registry); read through a neutral versioned
contract.

**Purpose**: Immutable record of one model stage transition and the evidence that
justified it, so promotion from probe to production-mandatory is auditable and
reversible.

| Field | Type | Rule |
|---|---|---|
| `promotion_id` | UUID | Primary key |
| `schema_version` | string | Required |
| `model_key`, `model_version`, `artifact_digest` | strings | Required |
| `route_snapshot_ref` | FK | Required |
| `from_stage`, `to_stage` | enums | `PROBE_ONLY`/`SHADOW`/`CANARY`/`MANDATORY`/`ARCHIVED`/`ROLLED_BACK` |
| `promotion_status` | enum | Current stage |
| `target_role` | enum | `signal` or `representation` only (never `decision_authority`) |
| `benchmark_ref` | reference | Native RTX 5090 stride-1 benchmark |
| `serving_metrics` | bounded JSON | Latency/throughput/resource, serving distribution, shadow duration |
| `gate_results` | bounded JSON | G1-G6 pass/fail with evidence references |
| `model_card_ref` | reference | Required for `MANDATORY` |
| `ledger_ref` | reference | Benchmark ledger entry |
| `approver_ref` | authenticated user ref | Governed approver (role-gated) |
| `decision` | enum | `promoted`/`held`/`rejected`/`rolled_back` |
| `rollback_ref` | nullable reference | Proven rollback |
| `valid_from` | timestamp | Required |
| `promotion_digest` | SHA-256 | Unique |
| `supersedes_ref` | nullable FK | Append-only evolution |
| `created_at` | timestamp | Immutable |

**Constraints**:

- Append-only; each transition creates a new record.
- `MANDATORY` requires `benchmark_ref`, complete `serving_metrics`, all gate
  results passing, `model_card_ref`, `approver_ref`, and a proven `rollback_ref`.
- `target_role` is `signal`/`representation` only; a behavioral
  `decision_authority` target is invalid under the doctrine.
- No behavioral accuracy/AUROC metric is recorded.
- A rollback creates a `ROLLED_BACK` record and restores the prior stage.

## New Entity: ProbeFineTuneRun

**Owner**: `apps.pipeline` (probe fine-tuning lane); read through a neutral
versioned contract.

**Purpose**: Immutable record of one corpus adaptation of a probe COPY, making
the fine-tune fully reconstructable and the frozen parent provably untouched.

| Field | Type | Rule |
|---|---|---|
| `finetune_id` | UUID | Primary key |
| `schema_version` | string | Required |
| `option` | enum | `pseudo_label_self_training`, `frozen_vs_finetuned_comparison`, `ssl_continued_pretraining`, `test_time_adaptation_ephemeral`, `distillation_from_governed_signals` |
| `parent_model_key`, `parent_artifact_digest` | strings | Frozen parent; never mutated |
| `child_model_key`, `child_artifact_digest` | strings nullable | New copy lineage (null for ephemeral TTA) |
| `corpus_manifest_ref` | artifact ref | Videos/windows used, digest-addressed |
| `filter_policy` | bounded JSON | Accepted-standard-method gates with parameter provenance refs |
| `decision_ledger_ref` | artifact ref | Accepted/refused/edited inference ledger |
| `accepted_count`, `refused_count`, `edited_count` | integers | Required for option (a) |
| `holdout_manifest_ref` | artifact ref | Held-out corpus slice for comparison |
| `benchmark_ref` | reference | Champion/challenger vs frozen parent + deterministic baseline |
| `benchmark_deltas` | bounded JSON | Serving metrics, signal stability, drift, baseline agreement |
| `promotion_status` | enum | Always starts `PROBE_ONLY` |
| `truth_state` | enum | Required |
| `run_digest` | SHA-256 | Unique |
| `created_at` | timestamp | Immutable |

**Constraints**:

- The parent artifact digest must verify unchanged after the run.
- Option (d) ephemeral TTA must have `child_artifact_digest = null`; persisting
  adapted weights without converting to option (a) + a promotion record is a
  violation.
- The decision ledger and filter policy are required for option (a); filtered
  inferences are never labeled ground truth.
- Reviewer feedback may only contribute exclusions to `corpus_manifest_ref`.
- No field stores a behavioral accuracy/AUROC claim.

## Relationships

```text
XAIModelRouteSnapshot 1 --- * XAIEvidenceEnvelope
XAISignalDefinition  1 --- * XAIEvidenceEnvelope
XAIEvidenceEnvelope  * --- * SignalPatternWindow
SignalPatternWindow  * --- 1 ObservedPatternProfileSnapshot
SignalPatternWindow  1 --- * AnomalyScoreRecord
ObservedPatternProfileSnapshot 1 --- * AnomalyScoreRecord
XAIEvidenceEnvelope  1 --- * AnomalyScoreContribution
AnomalyScoreRecord   1 --- * AnomalyScoreContribution
CalibrationSnapshot * --- * AnomalyScoreRecord
AnomalyScoreRecord   0..1 --- * XAIExplanationRecord
XAIEvidenceEnvelope  1 --- * XAIAttributionArtifact
XAIAttributionArtifact 1 --- 1 ExplanationEvaluationRecord
XAIExplanationRecord 1 --- * XAIReviewFeedback
StudentInteractionGraphFrame 1 --- * SignalPatternWindow
StudentInteractionGraphFrame * --- 1 XAIModelRouteSnapshot
ObservedPatternProfileSnapshot(general tier) 0..1 --- * AnomalyScoreRecord
AnomalyScoreRecord   * --- * ParameterProvenanceRecord
AnomalyScoreContribution * --- 1 ParameterProvenanceRecord
ObservedPatternProfileSnapshot 1 --- * ParameterProvenanceRecord (learned source)
XAIModelRouteSnapshot 1 --- * ModelPromotionRecord
ModelPromotionRecord 0..1 --- 1 ModelPromotionRecord (supersedes)
ProbeFineTuneRun * --- 1 (parent registry model, frozen)
ProbeFineTuneRun 0..1 --- 1 ModelPromotionRecord (child promotion path)
```

## State Transitions

### Evidence Envelope

```text
captured -> valid
captured -> degraded
captured -> unavailable
valid/degraded -> invalidated (new superseding evidence, never mutation)
```

### Deep XAI Artifact

```text
queued -> running -> completed -> evaluated_valid
queued/running -> failed
queued/running -> expired
completed -> evaluated_degraded
completed -> evaluated_invalid
```

### Anomaly Score

```text
requested -> scored_valid
requested -> scored_degraded
requested -> withheld
scored_* -> superseded (new append-only record)
```

### Observed Pattern Profile

```text
cold_start -> valid
cold_start -> insufficient_context
valid -> superseded (bounded append-only update)
valid/degraded -> quarantined
valid/degraded/quarantined -> invalidated
```

## Index And Retention Plan

- Index score/evidence by job, track, source-time range, signal key, truth state,
  and created time.
- Index signal-pattern windows/profiles by scope, route/feature compatibility,
  source-time range, truth/contamination/drift state, and digest.
- Index student-interaction graph frames by job, scope, source-time range,
  truth state, and graph digest.
- Index observed-pattern profiles additionally by `baseline_tier` so the general
  baseline resolves separately from student-tier profiles.
- Index parameter provenance by `param_key`, `source_kind`, scope, and digest.
- Index model promotion records by `model_key`/`model_version`, `promotion_status`,
  `decision`, and promotion digest.
- Index probe fine-tune runs by parent/child model key, option, truth state, and
  run digest.
- Index route/calibration snapshots by digest and compatibility fields.
- Index artifact lifecycle by request/status/created time.
- Partition or retention-manage high-volume evidence only after measured
  production volume; do not introduce partition complexity preemptively.
- Retention is configuration-governed and preserves published manifests and
  digests even when large artifacts expire.
