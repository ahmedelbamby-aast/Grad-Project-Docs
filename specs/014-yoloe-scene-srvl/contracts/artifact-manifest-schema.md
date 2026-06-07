# Contract: Artifact Manifest Schema

**Version**: `scene-artifact-manifest.v1`  
**Feature**: `014-yoloe-scene-srvl`

## Export Manifest

```json
{
  "schema_version": "yoloe-export-manifest.v1",
  "checkpoint": {
    "name": "yoloe-26s-seg.pt",
    "url": "https://github.com/ultralytics/assets/releases/download/v8.4.0/yoloe-26s-seg.pt",
    "sha256": "sha256:..."
  },
  "prompt": {
    "profile": "classroom_roi_guard_v1",
    "ordered_classes": ["person", "table", "desk", "chair"],
    "classes_sha256": "sha256:...",
    "source": "profile_default",
    "env_keys": [
      "YOLOE_SCENE_PROFILE",
      "YOLOE_SCENE_PROMPT_CLASSES",
      "YOLOE_SCENE_PROMPT_CLASSES_SHA256"
    ]
  },
  "runtime_configuration": {
    "env_fingerprint": "sha256:...",
    "declared_env_contract": "scene-env.v1",
    "hardcoded_operational_values_allowed": false
  },
  "export": {
    "set_classes_called_before_export": true,
    "format": "onnx_then_tensorrt",
    "onnx_ref": "artifact:model:onnx:...",
    "engine_ref": "artifact:model:engine:...",
    "artifact_sha256": {
      "onnx": "sha256:...",
      "engine": "sha256:..."
    }
  },
  "environment": {
    "git_sha": "abcdef0",
    "env_fingerprint": "sha256:...",
    "ultralytics": "8.4.56",
    "torch": "2.7.1",
    "tensorrt": "10.16.1.11"
  },
  "created_at": "2026-06-07T00:00:00Z"
}
```

Validation:

- `set_classes_called_before_export` must be `true`.
- `classes_sha256` must match runtime `.env` prompt resolution.
- `checkpoint.name` must be `yoloe-26s-seg.pt` for V1 unless a later governed
  benchmark changes it.

## Frame Artifact Manifest

```json
{
  "schema_version": "scene-artifact-manifest.v1",
  "job_id": "uuid",
  "video_id": "classroom_video_01",
  "frame_number": 1024,
  "timestamp_ms": 32000,
  "prompt_profile": "classroom_roi_guard_v1",
  "export_manifest_id": "sha256:...",
  "runtime_configuration": {
    "env_fingerprint": "sha256:...",
    "thresholds": {
      "YOLOE_SCENE_CONFIDENCE_THRESHOLD": 0.25,
      "YOLOE_SCENE_CONTRADICTION_OVERLAP": 0.4
    }
  },
  "artifacts": [
    {
      "artifact_id": "mask-1",
      "type": "instance_mask",
      "encoding": "rle_zstd",
      "object_id": 1,
      "class_id": 0,
      "confidence": 0.94,
      "confidence_threshold_env_key": "YOLOE_SCENE_CONFIDENCE_THRESHOLD",
      "path": "job-scoped/path/mask-1.rlezst",
      "sha256": "sha256:...",
      "size_bytes": 12345
    },
    {
      "artifact_id": "raw-yoloe-1",
      "type": "yoloe_output_payload",
      "encoding": "json_or_npz",
      "contains": ["boxes", "classes", "confidences", "masks", "shape_metadata"],
      "path": "job-scoped/path/yoloe-output.json",
      "sha256": "sha256:...",
      "size_bytes": 23456
    },
    {
      "artifact_id": "relation-1",
      "type": "vector_matrix",
      "encoding": "npz_compressed",
      "shape": [24, 24, 2],
      "dtype": "float32",
      "path": "job-scoped/path/vector.npz",
      "sha256": "sha256:...",
      "size_bytes": 54321
    }
  ],
  "trace": {
    "path": "job-scoped/path/trace.json",
    "sha256": "sha256:..."
  }
}
```

## Benchmark Figure Manifest

```json
{
  "schema_version": "scene-figure-manifest.v1",
  "benchmark_key": "yoloe-scene-20260607T000000Z",
  "raw_inputs": [
    {
      "path": "backend/logs/.../metrics.json",
      "sha256": "sha256:..."
    }
  ],
  "figures": [
    {
      "path": "docs/architecture/.../latency.png",
      "sha256": "sha256:...",
      "derived_from": ["backend/logs/.../metrics.json"]
    }
  ],
  "unavailable_metrics": [
    {
      "metric": "gpu_power_watts",
      "reason": "nvidia-smi field unavailable on host"
    }
  ]
}
```

Rules:

- Missing metrics must be listed as `unavailable`, not omitted or zero-filled.
- Figures must be generated from the same raw artifacts as the decision table.
- Digest mismatch invalidates the evidence package.
- YOLOE confidence scores and output payload references are required evidence
  for contradiction and recovery decisions.
- Every threshold or mode in a manifest must include the `.env` key that
  supplied it; hardcoded operational values invalidate the evidence package.
