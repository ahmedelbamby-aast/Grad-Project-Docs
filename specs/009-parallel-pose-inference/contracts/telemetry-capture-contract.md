# Contract: Accelerator Telemetry Capture

**Feature**: [Parallel Pose Inference with Hybrid Telemetry](../spec.md)  
**Data Model**: [data-model.md](../data-model.md)  
**Dashboard Projection**: [dashboard-telemetry-api-contract.md](dashboard-telemetry-api-contract.md)  
**MCP Projection**: [mcp-telemetry-server-contract.md](mcp-telemetry-server-contract.md)

## Scope

This contract defines how a validation run captures, reports, and persists accelerator telemetry for:

- NVIDIA GPU execution through the offline Triton service.
- Intel GPU execution through OpenVINO on Windows.

Telemetry applications are mandatory acceptance dependencies and augment, but do not replace, per-model runtime assignment proof. Raw collector outputs are ingestion inputs, not direct frontend or MCP integration surfaces.

## Required Sources

| Lane | Default Evidence Source | Role | Completion Requirement |
|------|-------------------------|------|------------------------|
| NVIDIA/Triton application | NVIDIA Nsight Systems CLI installed inside an instrumented Triton validation container and used for bounded capture | Mandatory in-container NVIDIA trace evidence | Installed, invocable, and readable non-empty trace output required for a passing run. |
| NVIDIA/Triton KPI capture | Triton Prometheus `/metrics` sampled during the run | Low-overhead model-serving and Triton-published NVIDIA GPU metrics | Required for a passing run. |
| Intel/OpenVINO | OpenVINO requested/effective device and profiling output | Authoritative per-model Intel execution proof and stage metrics | Required for a passing hybrid run. |
| Intel/Windows application | Intel PresentMon installed on Windows and used for run capture | Mandatory host-level Intel GPU timeline telemetry | Installed, invocable, and readable non-empty capture output required for a passing run. |

DCGM Exporter is not enabled concurrently by default for the pinned Triton `24.01` topology. It may be evaluated as an alternate, separately configured monitoring authority only after metrics compatibility is proven and it replaces, rather than conflicts with, the selected GPU-metrics authority.

## Configuration Contract

Configuration must be profile-specific and secrets-free in evidence outputs.

| Setting Intent | Required Behavior |
|----------------|-------------------|
| Instrumented Triton image | Use a pinned image/build identity containing a pinned Nsight Systems CLI installation and an isolated validation profile. |
| Nsight capture configuration | Configure bounded trace collection and export/report paths under the job evidence directory. |
| Offline Triton metrics URL | Resolve independently from any live endpoint; default development mapping is the offline Triton metrics service on host port `8102`. |
| Telemetry sampling interval | Tunable by environment/profile and recorded in each capture session. |
| Intel device selector | Resolve and record a specific Intel `GPU.X` device and execution/profiling evidence. |
| PresentMon executable path/settings | Configurable on Windows; record installed version and capture outcome, not machine-private data. |
| Telemetry mode | Baseline and optimized comparisons must use the same tool/capture configuration identity. |
| Collector artifact directory | Stored beneath the job evidence directory and included in the artifact manifest. |

## Capture Lifecycle

1. Create the `HybridValidationRun` and persist non-secret profile/configuration identity.
2. Verify required Triton and OpenVINO runtime readiness.
3. Verify Nsight Systems CLI is installed and invocable inside the instrumented Triton validation container and PresentMon is installed and invocable on Windows; persist installation evidence.
4. Resolve model-to-device assignments and fail preflight if a required Intel device cannot be proven.
5. Start mandatory telemetry capture sessions immediately before measured inference execution.
6. Execute all required inference and export stages.
7. Stop collectors after export/fidelity measurement completes.
8. Persist raw samples, traces, summaries, installation/capture states, and collector errors.
9. Reconcile telemetry evidence with assignments, stage results, artifacts, and benchmark instrumentation parity.
10. Normalize validated summary, timeline, assignment, artifact, fidelity, and comparison records in backend persistence.
11. Project authorized normalized records through dashboard REST/WebSocket contracts and through the mandatory read-only MCP contract.

## Evidence Payload

Each run must expose a summary document conforming to this minimum shape:

```json
{
  "run_id": "uuid",
  "job_id": "uuid",
  "telemetry_installations": [
    {
      "tool": "nvidia_nsight_systems_cli",
      "execution_boundary": "triton_validation_container",
      "version": "pinned-version",
      "installed": true,
      "capture_capable": true
    },
    {
      "tool": "intel_presentmon",
      "execution_boundary": "windows_host",
      "version": "pinned-version",
      "installed": true,
      "capture_capable": true
    }
  ],
  "capture_sessions": [
    {
      "source": "nsight_systems",
      "availability": "available",
      "artifact_path": "nvidia_nsight_trace.nsys-rep",
      "non_empty": true
    },
    {
      "source": "triton_prometheus",
      "availability": "available",
      "sampling_interval_ms": 1000,
      "started_at": "ISO-8601",
      "ended_at": "ISO-8601",
      "artifact_path": "accelerator_telemetry.jsonl",
      "summary": {
        "accelerator_vendor": "nvidia",
        "effective_device": "validated-device-id",
        "gpu_utilization_percent": {
          "mean": 0.0,
          "p95": 0.0
        },
        "memory_used_bytes": {
          "peak": 0
        }
      }
    },
    {
      "source": "presentmon",
      "availability": "available",
      "artifact_path": "intel_presentmon.csv",
      "non_empty": true
    },
    {
      "source": "openvino_profile",
      "availability": "available",
      "summary": {
        "accelerator_vendor": "intel",
        "requested_device": "explicit-intel-device",
        "effective_device": "reported-intel-device",
        "models": []
      }
    }
  ],
  "placement_proof": {
    "nvidia_triton_model_present": true,
    "intel_openvino_model_present": true,
    "verdict": "passed"
  },
  "normalized_projection": {
    "summary_ready": true,
    "timeline_ready": true,
    "artifacts_ready": true,
    "assignments_ready": true,
    "comparison_ready": true,
    "pii_safe": true
  }
}
```

## Failure Semantics

| Condition | Runtime Job Behavior | Validation Verdict |
|-----------|----------------------|--------------------|
| Nsight Systems missing/uninvocable or trace output empty | Preserve any safe artifacts and record telemetry failure. | Fail preflight or evidence gate. |
| PresentMon missing/uninvocable or capture output empty | Preserve any safe artifacts and record telemetry failure. | Fail preflight or evidence gate. |
| Triton metrics endpoint unreachable during required NVIDIA measurement | Continue only long enough to persist the failure state and artifacts safely. | Fail evidence gate. |
| Intel OpenVINO device identity cannot be proven | Do not claim hybrid validation success. | Fail preflight or evidence gate. |
| A collector crashes while inference is running | Inference safety and artifact preservation take priority. | Persist degraded/failure evidence; do not silently pass. |
| Baseline/candidate telemetry modes differ or overhead is not accounted for | Retain data for diagnosis. | Reject performance acceptance. |
| Concurrent alternative DCGM mode conflicts with Triton metrics | Stop using the conflicted metric source and record incompatibility. | Fail that evidence run; redesign source authority before retry. |
| MCP disabled, unavailable, or rejects an unauthorized request | Continue ingestion and dashboard delivery; write audit verdict when a request occurred. | Does not fail capture/inference validation if required evidence remains valid. |
| Normalized telemetry cannot be generated from mandatory evidence | Preserve raw evidence and expose the failure safely. | Fail evidence/API acceptance; do not expose contradictory projections. |

## Projection Boundary

High-volume samples remain in controlled evidence artifacts. Consumer interfaces expose normalized bounded information:

| Consumer Surface | Summary Fields |
|------------------|----------------|
| Job status extension | `validation_gate_status`, `evidence_ready`, `telemetry_availability`, `placement_proof_verdict` |
| Dashboard REST/WebSocket contract | Summary, bounded timeline, artifact metadata, assignments, comparison, and live/delayed updates |
| MCP resources/tools | Bounded PII-safe views and read-only diagnostic calculations over the same normalized evidence |

The MCP service must not access collectors independently, serve unrestricted raw trace files, or change capture/run state in its initial release.

## Acceptance Tests

- Instrumented Triton validation image exposes a pinned, invocable Nsight Systems CLI installation.
- PresentMon installation/readiness is required and versioned on the Windows host.
- Offline metrics URL is selected independently from live Triton configuration.
- Triton metrics parsing accepts valid metric payloads and records invalid/unreachable endpoints.
- OpenVINO evidence records requested and effective Intel device identity.
- Explicit Intel `GPU.X` execution/profiling evidence is persisted.
- Empty/missing Nsight or PresentMon capture output rejects validation.
- Equivalent baseline/candidate telemetry modes and overhead evidence are enforced.
- Telemetry timestamps overlap corresponding stage timestamps.
- A passing evidence package includes both NVIDIA/Triton and Intel/OpenVINO model execution records.
- Normalized persisted records are the source for both dashboard and MCP projections.
- Dashboard telemetry remains available with MCP disabled or failed.
- MCP-enabled tests prove bounded PII-safe read-only projection and auditable denial of prohibited access.
