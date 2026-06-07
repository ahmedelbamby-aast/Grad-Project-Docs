# Contract: API Scene Artifacts

**Version**: `scene-api.v1`  
**Feature**: `014-yoloe-scene-srvl`

## Compatibility Rules

- Scene fields are optional while `YOLOE_SCENE_ENABLED=0` or `SRVL_ENABLED=0`.
- Existing job status and playback endpoints must remain valid when scene
  fields are absent.
- Full masks, matrices, snapshots, traces, and MP4 artifacts are never embedded
  in default job status payloads.
- All artifact reads require authenticated job access.

## GET /api/v1/video-analysis/jobs/{job_id}/scene/summary/

Purpose: return compact scene state for a job.

Response `200`:

```json
{
  "schema_version": "scene-api.v1",
  "job_id": "uuid",
  "scene_enabled": true,
  "srvl_enabled": true,
  "live_status": "disabled_for_live",
  "prompt_profile": {
    "name": "classroom_roi_guard_v1",
    "classes_sha256": "sha256:..."
  },
  "export_manifest": {
    "manifest_id": "sha256:...",
    "checkpoint": "yoloe-26s-seg.pt",
    "status": "validated"
  },
  "runtime_configuration": {
    "env_fingerprint": "sha256:...",
    "declared_env_contract": "scene-env.v1",
    "hardcoded_operational_values_allowed": false
  },
  "frames": {
    "scene_frame_count": 1136,
    "unavailable_count": 0,
    "degraded_count": 0
  },
  "people_mismatch": {
    "enabled": false,
    "mismatch_frame_count": 0,
    "missing_candidate_count": 0,
    "unresolved_count": 0
  },
  "non_roi": {
    "contradiction_event_count": 0
  },
  "spatial": {
    "relation_frame_count": 1136,
    "dense_limit": 128,
    "dropped_visualization_frames": 0
  },
  "artifacts": {
    "manifest_url": "/api/v1/video-analysis/jobs/uuid/scene/artifacts/manifest/"
  }
}
```

Errors:

- `401`: unauthenticated.
- `403`: authenticated user lacks job access.
- `404`: job not found.

## GET /api/v1/video-analysis/jobs/{job_id}/frames/{frame_number}/scene/

Purpose: return compact frame scene data without loading full mask payloads.

Response `200`:

```json
{
  "schema_version": "scene-api.v1",
  "job_id": "uuid",
  "frame_number": 1024,
  "timestamp_ms": 32000,
  "scene_status": "valid",
  "prompt_profile": "classroom_roi_guard_v1",
  "export_manifest_id": "sha256:...",
  "objects": [
    {
      "frame_local_object_id": 1,
      "class_id": 0,
      "class_name": "person",
      "confidence": 0.94,
      "confidence_source": "yoloe_score",
      "confidence_threshold": {
        "env_key": "YOLOE_SCENE_CONFIDENCE_THRESHOLD",
        "value": 0.25
      },
      "bbox": [300, 180, 380, 260],
      "center": [340.5, 220.0],
      "anchor": [340.5, 260.0],
      "mask_ref": "artifact:mask:...",
      "raw_yoloe_output_ref": "artifact:yoloe-output:...",
      "unavailable_output_reasons": {},
      "is_non_roi": false
    }
  ],
  "events": {
    "contradictions": [],
    "people_mismatch": null
  },
  "relations": {
    "matrix_mode": "full",
    "object_count": 24,
    "distance_matrix_ref": "artifact:npz:...",
    "vector_matrix_ref": "artifact:npz:...",
    "correlation_map_ref": "artifact:map:..."
  },
  "trace_ref": "artifact:trace:..."
}
```

## GET /api/v1/video-analysis/jobs/{job_id}/scene/artifacts/{artifact_id}/

Purpose: authenticated artifact download or signed internal stream for masks,
matrices, snapshots, traces, and MP4 scene maps.

Required behavior:

- Validate user access to `job_id`.
- Validate artifact belongs to `job_id`.
- Return `Content-Type` based on manifest encoding.
- Return `X-Scene-Artifact-SHA256`.
- Return `404` for missing artifact and `409` for digest mismatch.

## GET /api/v1/video-analysis/jobs/{job_id}/scene/map-video/

Purpose: download or stream the rendered scene-map MP4 when available.

Response states:

- `200`: MP4 artifact is available.
- `202`: render pending.
- `204`: render disabled.
- `409`: render failed with traceable reason.

## Serializer Rules

- Do not use `fields = '__all__'` for new scene serializers.
- Use explicit read-only fields for summaries.
- Large artifacts are exposed by refs, not inline binary or huge JSON matrices.
- Unknown future fields must not break old clients.
- Confidence scores, threshold snapshots, and unavailable YOLOE output reasons
  are evidence fields and must not be dropped by serializers.
- API behavior must read operational values from the governed settings layer
  backed by `.env`; hardcoded prompt classes, thresholds, score weights,
  renderer choices, matrix modes, or artifact codecs are not allowed.
