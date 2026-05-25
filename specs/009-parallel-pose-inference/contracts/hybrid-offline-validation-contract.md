# Contract: Required Hybrid Offline Validation Run

**Feature**: [Parallel Pose Inference with Hybrid Telemetry](../spec.md)  
**Data Model**: [data-model.md](../data-model.md)  
**Telemetry Contract**: [telemetry-capture-contract.md](telemetry-capture-contract.md)  
**Dashboard Contract**: [dashboard-telemetry-api-contract.md](dashboard-telemetry-api-contract.md)  
**MCP Contract**: [mcp-telemetry-server-contract.md](mcp-telemetry-server-contract.md)

## Purpose

The feature is not complete until required real-video offline evidence proves a correctly integrated hybrid pipeline, improved measured performance, complete artifacts, and acceptable final-video fidelity across:

1. One hybrid assignment run demonstrating Intel/OpenVINO and NVIDIA/Triton assignment in the same run.
2. One full-model NVIDIA stress lane run (all selected models from `.env` on Triton).
3. One full-model Intel stress lane run (all selected models from `.env` on OpenVINO).

## Validation Input

Initial required real-video proof input:

```text
Raw Data/Arguing Students only/Arguing_004.mp4
```

The evidence package must include a source checksum and media probe. Additional representative videos may be run after the initial gate passes.

## Required Runtime Assignment

| Requirement | Passing Evidence |
|-------------|------------------|
| Full-model NVIDIA stress run | One required run uses all selected models from `.env` with runtime assignment and successful execution evidence on NVIDIA/Triton. |
| Full-model Intel stress run | One required run uses all selected models from `.env` with runtime assignment and successful execution evidence on Intel GPU/OpenVINO. |
| NVIDIA telemetry application | Nsight Systems CLI installation/version in the instrumented Triton validation image and readable non-empty trace capture. |
| Intel telemetry application | PresentMon installation/version on Windows and readable non-empty capture. |
| Celery isolation | Exactly one dedicated worker identity is recorded for each model used in the run. |
| Equivalent comparison | Baseline and optimized runs for each stress lane use the same input, enabled models, device assignment intent, telemetry mode, and artifact obligations, with instrumentation overhead reported. |

## Blocking Stage Results

The following stages must exist and pass:

| Stage | Required Output Or Verification |
|-------|---------------------------------|
| `decode` | Source opens; frame count is recorded. |
| `preprocess` | Frames submitted for inference are accounted for. |
| `object_inference` | Detection results and NVIDIA/Triton assignment evidence where assigned. |
| `pose_inference` | Pose outputs and pose policy threshold results. |
| `association_tracking` | Joined object/pose/tracking artifacts. |
| `artifact_persistence` | Required JSON/JSONL artifacts readable and hashed. |
| `overlay_render` | Required visual overlays rendered without unresolved error. |
| `final_video_export` | Playable final annotated output media file. |
| `frame_reconciliation` | Per-frame verdict under the missing-pose/gap policy. |
| `video_fidelity_validation` | Required media property and non-overlay quality verdict. |

A job cannot be reported as validation-passed when a blocking stage is absent, failed, or degraded outside an explicit specification allowance.

## Artifact Package

Required artifacts under the job evidence directory:

| Artifact | Required Content |
|----------|------------------|
| `run_manifest.json` | Source/profile/configuration identity, model assignments, collector sessions, verdict. |
| `telemetry_installation_manifest.json` | Nsight Systems image/package/version/readiness and PresentMon installation/version/readiness evidence. |
| `stage_results.json` | Blocking-stage statuses, durations, frame counts, errors, references. |
| `nvidia_nsight_trace.nsys-rep` | Non-empty bounded NVIDIA/Triton profiling capture from the instrumented container. |
| `intel_presentmon.csv` | Non-empty Windows Intel telemetry capture aligned to the run interval. |
| `accelerator_telemetry.jsonl` | Raw NVIDIA/Intel capture samples or recorded source unavailability. |
| `telemetry_summary.json` | Correlated metric aggregates and placement proof summary. |
| `normalized_telemetry_summary.json` | Validated PII-safe summary/projection readiness for dashboard and MCP access. |
| `benchmark_comparison.json` | Baseline/candidate provenance, throughput/latency deltas, thresholds, verdict. |
| `frame_reconciliation.json` | Decoded/object/pose/association accounting and threshold verdict. |
| `artifact_manifest.json` | Integrity and readiness of all required output/evidence files. |
| `final_video_fidelity.json` | Source/output probes, playability, media preservation checks, quality verdict. |
| `pose_results.json` | Required pose results. |
| `pose_quality_summary.json` | Required pose quality result. |
| `final_output_video.mp4` | Final annotated output for end users. |

Existing supporting artifacts such as inference audit and lifecycle event logs must be included in the artifact manifest where generated.

## Performance Gate

| Metric | Constraint |
|--------|------------|
| Comparison basis | Same source checksum, required stages, models, assignments, output contract, telemetry mode, and recorded instrumentation overhead/control basis. |
| Optimized offline p95 processing latency | At least 20% lower than the unoptimized hybrid baseline. |
| Throughput | Reported for both runs and evaluated against the feature performance target. |
| Errors | Zero unresolved blocking-stage failures. |
| Grayscale experiment | Accepted only when its documented throughput and quality thresholds pass; it is not assumed by default. |

## Pose Completeness Gate

The frame reconciliation verdict fails when:

- More than 2% of decoded frames lack required pose status.
- Any continuous pose-output gap exceeds 10 seconds.
- Duplicate or unresolved frame evidence makes the run non-reconcilable.

## Final Video Fidelity Gate

| Verification | Passing Rule |
|--------------|--------------|
| Playability | Output is a decodable video file, not a frame directory. |
| Resolution and aspect ratio | Preserve source values. |
| Frame rate and duration | Preserve within a tolerance no greater than one source frame. |
| Audio presence | Preserve audio presence if the input contains audio. |
| Visual comparison | Non-overlay regions do not exceed the approved encoding quality regression threshold. |
| Overlay treatment | Intentional overlay areas are masked/excluded from source parity comparison. |

## Status/API Projection

The standard job status should expose concise verdict fields:

```json
{
  "run_id": "uuid",
  "validation_gate_status": "passed",
  "stage_summary": {
    "blocking_failed": 0,
    "blocking_passed": 10
  },
  "placement_proof_verdict": "passed",
  "benchmark_verdict": "passed",
  "frame_reconciliation_verdict": "passed",
  "fidelity_verdict": "passed",
  "evidence_ready": true
}
```

Detailed evidence should be retrieved through dedicated evidence, telemetry, benchmark, reconciliation, fidelity, and artifact resources.

## Dashboard And MCP Boundary

The required offline validation run must persist normalized evidence that the dashboard can retrieve through authenticated REST/WebSocket interfaces while the mandatory MCP service is deployed as a separate access surface. Dashboard verification must demonstrate independence from MCP request-path availability.

The MCP service is in feature scope as a mandatory, separately tested, read-only observability interface. Contract/security tests must verify it exposes only authorized, PII-safe, bounded normalized resources and read-only diagnostic tools. Its disablement, outage, overload, or rejection of a prohibited request does not fail an otherwise valid hybrid processing run and must not modify run artifacts or states.

## Test Obligations

- Unit/contract tests for schema, gate calculation, collector failure states, and status/API projection.
- Integration tests for mandatory Nsight Systems and PresentMon installation/readiness/capture failure behavior.
- Integration tests for runtime assignment persistence and model-worker routing.
- System tests with real input media and actual artifact generation.
- Performance test comparing baseline and optimized hybrid runs.
- Required hybrid assignment run using Intel/OpenVINO and NVIDIA/Triton on the same video job.
- Required full-model NVIDIA stress lane run (all `.env` selected models on Triton).
- Required full-model Intel stress lane run (all `.env` selected models on OpenVINO).
- Dashboard integration tests showing normalized telemetry delivery while MCP is disabled or unavailable.
- MCP contract/security/isolation tests for bounded authorized reads, denial auditing, PII redaction, and no interference with inference, capture persistence, or dashboard delivery.
