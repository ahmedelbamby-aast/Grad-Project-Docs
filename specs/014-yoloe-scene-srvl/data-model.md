# Data Model: YOLOE Scene Segmentation And SRVL

**Date**: 2026-06-07  
**Feature**: `014-yoloe-scene-srvl`

## Entity Overview

The implementation should keep compact, queryable state in PostgreSQL and store
large masks, matrices, maps, figures, snapshots, traces, and MP4 outputs as
job-scoped authenticated artifacts. All entities below are logical entities; the
final Django model split may reuse existing tables where compatible.

## ScenePromptProfile

Purpose: named prompt profile used for export, testing, benchmarking, and
runtime validation.

Fields:

- `profile_name`: string, default `classroom_roi_guard_v1`.
- `schema_version`: string.
- `ordered_classes`: ordered list of prompt class names.
- `classes_sha256`: digest of the ordered class list and normalization rules.
- `source`: enum `profile_default`, `env_override`, `test_fixture`.
- `env_keys`: list of `.env` keys that contributed to prompt resolution.
- `created_at`: timestamp.
- `is_benchmark_default`: boolean.

Validation:

- Class order is part of the digest.
- Empty prompt lists are invalid.
- `.env` override changes `classes_sha256` and forces re-export.

## YoloeExportManifest

Purpose: immutable evidence that a deployed model artifact was exported from the
required checkpoint after prompts were applied.

Fields:

- `manifest_id`: UUID or digest-derived identifier.
- `checkpoint_name`: expected `yoloe-26s-seg.pt` for V1.
- `checkpoint_url`: source URL.
- `checkpoint_sha256`: digest when available.
- `prompt_profile_name`: string.
- `prompt_classes_sha256`: digest.
- `ordered_classes`: ordered prompt classes.
- `export_format`: enum `onnx`, `tensorrt`, `onnx_then_tensorrt`.
- `onnx_artifact_ref`: optional artifact reference.
- `engine_artifact_ref`: optional artifact reference.
- `artifact_sha256`: digest map.
- `ultralytics_version`: string.
- `torch_version`: string.
- `tensorrt_version`: string when applicable.
- `env_fingerprint`: digest of relevant runtime/export env keys.
- `export_started_at`, `export_completed_at`: timestamps.
- `status`: enum `created`, `prompt_locked`, `exported`, `validated`,
  `deployed`, `rejected`, `stale`.

Validation:

- `prompt_locked` requires `set_classes()` evidence before export.
- Runtime use requires prompt digest equality between `.env` and manifest.
- Wrong checkpoint or missing manifest fails closed.

## SceneFrameSummary

Purpose: one compact row per frame where the YOLOE scene lane runs or records an
unavailable state.

Fields:

- `job_id`: foreign key to the existing video-analysis job.
- `video_id`: string or existing video reference.
- `frame_number`: integer.
- `timestamp_ms`: integer.
- `source_fps`: numeric.
- `prompt_profile_name`: string.
- `export_manifest_id`: reference.
- `object_count`: integer.
- `person_count`: integer.
- `non_roi_object_count`: integer.
- `non_roi_coverage_ratio`: float.
- `scene_status`: enum `valid`, `unavailable`, `degraded`, `disabled_for_live`.
- `unavailable_reason`: nullable string.
- `artifact_manifest_ref`: artifact reference.
- `trace_ref`: artifact reference.
- `created_at`: timestamp.

Validation:

- Unique idempotency key: `(job_id, frame_number, prompt_digest,
  export_manifest_id)`.
- `scene_status=valid` requires object summary and manifest references.
- Unavailable frames must record a reason.

## SceneObjectObservation

Purpose: normalized object observation from the unified YOLOE instance
segmentation result.

Fields:

- `summary_id`: reference to `SceneFrameSummary`.
- `frame_local_object_id`: integer.
- `raw_yoloe_output_ref`: optional artifact reference for backend-exposed raw
  output tensors, prototypes, coefficients, or route payload metadata.
- `class_name`: string from prompt profile.
- `class_id`: integer from prompt order/runtime output.
- `confidence`: float.
- `confidence_source`: enum `yoloe_score`, `calibrated_score`, `unavailable`.
- `confidence_threshold_env_key`: `.env` key used for inclusion decisions.
- `confidence_threshold_value`: numeric threshold snapshot.
- `bbox_xyxy`: `[x1, y1, x2, y2]`.
- `center_xy`: `[x, y]`.
- `anchor_xy`: `[x, y]`, usually centroid or bottom-center depending on class.
- `mask_artifact_ref`: RLE-Zstd artifact reference.
- `mask_area_px`: integer.
- `mask_confidence`: nullable float when exposed by the backend.
- `output_shape_metadata`: JSON object for model/image/output shapes.
- `unavailable_output_reasons`: map from expected YOLOE output field to reason.
- `track_id`: nullable source-scoped local track ID if matched.
- `provisional_id`: nullable recovery/local object ID.
- `is_person`: boolean.
- `is_non_roi`: boolean.

Validation:

- Coordinates must be finite and clipped or marked invalid with reason.
- Mask artifact digest must match the manifest.
- `frame_local_object_id` is scoped to frame and prompt export only.
- Confidence scores are preserved even when below a later decision threshold;
  threshold decisions record the env key and threshold value.
- Every YOLOE output emitted by the runtime route is represented directly,
  artifact-referenced, or listed in `unavailable_output_reasons`.

## NonRoiRegionMask

Purpose: union mask for objects that are not regions of interest for person or
behavior reasoning.

Fields:

- `summary_id`: reference.
- `included_classes`: list of non-ROI classes.
- `union_mask_artifact_ref`: RLE-Zstd artifact reference.
- `coverage_ratio`: float.
- `overlap_threshold`: float.
- `overlap_threshold_env_key`: `.env` key for the overlap threshold.
- `min_confidence_env_key`: `.env` key for minimum confidence used by the guard.
- `created_at`: timestamp.

Validation:

- Union mask is derived only from prompt classes marked non-ROI.
- Absence of YOLOE output yields `unavailable`, not an empty truth claim.

## SceneContradictionEvent

Purpose: append-only evidence that a downstream detection or recovery candidate
overlaps non-ROI regions or lacks person-mask support.

Fields:

- `event_id`: UUID.
- `job_id`, `frame_number`, `timestamp_ms`.
- `source_detection_id`: nullable existing detection reference.
- `source_candidate_id`: nullable recovery candidate reference.
- `non_roi_mask_ref`: artifact reference.
- `overlap_ratio`: float.
- `source_confidence`: nullable float from the source YOLOE or downstream
  detection.
- `decision_thresholds`: JSON object containing threshold values and `.env`
  key names used for this contradiction decision.
- `event_type`: enum `non_roi_overlap`, `person_mask_absent`,
  `scene_unavailable`.
- `severity`: enum `info`, `warning`, `blocking_candidate`.
- `action`: fixed `flag_only` for V1.
- `reason_codes`: list.
- `created_at`: timestamp.

Validation:

- Original detection rows are never deleted or mutated by this event.
- Idempotency key includes source object, frame, event type, and prompt digest.

## PeopleMismatchEvent

Purpose: per-frame reconciliation between YOLOE person count and the existing
student-plus-teacher count.

Fields:

- `event_id`: UUID.
- `summary_id`: reference.
- `yoloe_person_count`: integer.
- `student_count`: integer.
- `teacher_count`: integer.
- `count_delta`: integer.
- `redis_lookup_ms`: numeric or unavailable reason.
- `confidence_thresholds`: JSON object containing YOLOE confidence and
  recovery threshold snapshots with `.env` key names.
- `matching_backend`: enum `redis_embedding`, `db_embedding`, `geometry_only`,
  `unavailable`.
- `matched_existing_count`: integer.
- `false_alarm_count`: integer.
- `missing_candidate_count`: integer.
- `unresolved_count`: integer.
- `match_manifest_ref`: artifact reference.
- `created_at`: timestamp.

Validation:

- Raw local track IDs are not identity proof.
- Assignment is deterministic and globally one-to-one within the frame event.
- Ambiguous cases remain `unresolved`.

## SceneRecoveryCandidate

Purpose: append-only candidate for a person YOLOE located but the current
student/teacher detector did not account for.

Fields:

- `candidate_id`: UUID.
- `mismatch_event_id`: reference.
- `source_object_id`: frame-local YOLOE object reference.
- `provisional_person_id`: source-scoped provisional ID.
- `bbox_xyxy`, `center_xy`, `mask_artifact_ref`.
- `appearance_embedding_ref`: Redis key or DB reference when available.
- `score_total`: float.
- `score_breakdown`: JSON object.
- `yoloe_confidence`: float or unavailable reason.
- `threshold_snapshot`: JSON object containing `.env` key names and values used
  for eligibility, scoring, and re-pass decisions.
- `classification`: enum `matched_existing`, `false_alarm`, `duplicate`,
  `missing_candidate`, `unresolved`.
- `repass_state`: enum `disabled`, `not_eligible`, `eligible`,
  `queued`, `completed`, `failed`.
- `timeline_policy`: fixed `append_only_original_frame_time` for V1.
- `created_at`, `updated_at`.

Validation:

- V1 default keeps `repass_state=disabled` unless a separate measured candidate
  enables it.
- Candidate re-pass, when future-enabled, must not retroactively mutate prior
  detections.

## SpatialRelationFrame

Purpose: ordered object set and coordinate basis used by SRVL.

Fields:

- `relation_frame_id`: UUID.
- `summary_id`: reference.
- `ordered_object_ids`: ordered list of scene object references.
- `coordinate_basis`: enum `center_xy`, `anchor_xy`.
- `frame_width`, `frame_height`: integers.
- `matrix_mode`: enum `full`, `thresholded`, `top_k`, `heatmap_only`.
- `object_count`: integer.
- `distance_matrix_ref`: optional NPZ artifact reference.
- `angle_matrix_ref`: optional NPZ artifact reference.
- `vector_matrix_ref`: optional NPZ artifact reference.
- `cartesian_vector_ref`: optional NPZ artifact reference.
- `map_artifact_refs`: map of distance/angle/correlation refs.
- `compute_backend`: enum `torch_cuda`, `cupy`, `numpy_cpu`, `unavailable`.
- `timings_ms`: JSON object.
- `created_at`: timestamp.

Validation:

- Dense `full` mode requires `object_count <= 128`.
- Above 128 objects requires `top_k`, `thresholded`, or `heatmap_only`.
- Distance/angle/vector artifacts must share the same ordered object list.

## SceneVisualizationArtifact

Purpose: human-facing map, heatmap, snapshot, or MP4 output.

Fields:

- `artifact_id`: UUID.
- `job_id`: reference.
- `frame_number`: nullable for job-level MP4.
- `artifact_type`: enum `distance_heatmap`, `angle_map`, `correlation_map`,
  `scene_snapshot_png`, `scene_snapshot_svg`, `scene_snapshot_jpg`,
  `scene_map_mp4`.
- `render_backend`: enum `pixijs_webgl`, `deck_gl`, `three_js`,
  `matplotlib_agg`, `canvas_2d`, `unavailable`.
- `encoding`: string.
- `width`, `height`: integers.
- `artifact_ref`: authenticated artifact reference.
- `sha256`: digest.
- `render_ms`: numeric or unavailable reason.
- `dropped_visualization_frames`: integer.
- `created_at`: timestamp.

Validation:

- Production real-time renderer cannot be Matplotlib.
- MP4 output must preserve source frame count and timing.

## SceneArtifactManifest

Purpose: index of every scene artifact in a job or frame.

Fields:

- `manifest_id`: UUID.
- `schema_version`: string.
- `job_id`, `video_id`, optional `frame_number`.
- `artifacts`: list of path/ref, type, encoding, digest, size, and source.
- `raw_input_digests`: map.
- `generated_by`: script/module/version.
- `created_at`: timestamp.

Validation:

- Every referenced file must resolve or carry explicit unavailable reason.
- Digest mismatch blocks acceptance.

## SceneArtifactAccessGrant

Purpose: job-scoped authorization boundary for classroom imagery artifacts.

Fields:

- `job_id`: reference.
- `user_id` or reviewer group reference.
- `scope`: enum `scene_summary`, `scene_artifact_read`, `scene_export_read`.
- `expires_at`: nullable timestamp.
- `created_at`: timestamp.

Validation:

- Unauthenticated access is denied.
- Authenticated users without job access cannot retrieve masks, snapshots,
  matrices, videos, or traces containing classroom imagery.

## SceneRuntimeConfiguration

Purpose: immutable snapshot of every `.env`/settings value that can alter
YOLOE output capture, contradiction decisions, mismatch recovery, SRVL output,
rendering, artifact generation, benchmarking, or rollback.

Fields:

- `config_id`: digest-derived identifier.
- `job_id`: nullable job reference for runtime snapshots.
- `export_manifest_id`: nullable reference for export snapshots.
- `env_fingerprint`: SHA-256 digest of normalized scene-related env values.
- `declared_env_keys`: map of key name to typed value, source, and default
  benchmark value.
- `prompt_keys`: subset for prompt profile/classes/digest.
- `threshold_keys`: subset for confidence, mask, overlap, support, recovery,
  top-k, distance, and queue thresholds.
- `render_keys`: subset for renderer, codec, visual queue, and MP4 settings.
- `benchmark_keys`: subset for trace/benchmark/GPU/frontend metrics toggles.
- `created_at`: timestamp.

Validation:

- A scene runtime cannot start when an operational key used by the feature is
  absent from `declared_env_keys`.
- Feature code cannot bypass this entity with hardcoded operational values.
- Changing prompt keys marks prior YOLOE export manifests `stale`.
- Changing threshold/render/SRVL keys requires a new runtime snapshot and
  benchmark lineage entry.

## State Transitions

### Export Manifest

```text
created -> prompt_locked -> exported -> validated -> deployed
created -> prompt_locked -> exported -> rejected
deployed -> stale
```

### Frame Scene State

```text
disabled -> unavailable
enabled -> segmented -> guarded -> spatial_ready -> visualized
enabled -> unavailable
enabled -> degraded
```

### Recovery Candidate

```text
created -> matched_existing
created -> false_alarm
created -> missing_candidate -> eligible -> queued -> completed
created -> missing_candidate -> disabled
created -> unresolved
queued -> failed
```

## Persistence Rules

- PostgreSQL is the only relational authority.
- Large artifact payloads are never exposed as public unauthenticated files.
- Existing detection rows are not mutated by contradiction or recovery evidence.
- Re-runs must use documented idempotency keys to avoid duplicate summaries,
  contradiction events, recovery candidates, and relation artifacts.
- Missing GPU, Redis, frontend, power, or render metrics must be recorded as
  `unavailable` with a reason.
- YOLOE confidence scores and output metadata are evidence fields; filtering
  must not erase the original score or hide the threshold used.
- Every operational feature value must originate from `.env`/settings and be
  captured in `SceneRuntimeConfiguration`.
