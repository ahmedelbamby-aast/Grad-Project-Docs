# Quickstart: Telemetry Provisioning And Hybrid Offline Validation

**Feature**: [Parallel Pose Inference with Hybrid Telemetry](spec.md)  
**Plan**: [plan.md](plan.md)  
**Contracts**: [Telemetry Capture](contracts/telemetry-capture-contract.md), [Hybrid Offline Validation](contracts/hybrid-offline-validation-contract.md), [Dashboard Telemetry API](contracts/dashboard-telemetry-api-contract.md), and [MCP Telemetry Server](contracts/mcp-telemetry-server-contract.md)

## Intent

This is an implementation and validation guide. Telemetry applications are mandatory for acceptance, but this planning artifact does not state that they have already been installed or that the hybrid completion gate has passed.

## Current Preflight Facts

| Item | Current Planning Observation |
|------|------------------------------|
| Triton offline metrics | Existing development Compose topology maps the offline Triton metrics endpoint to host port `8102`. |
| NVIDIA required application | Install NVIDIA Nsight Systems CLI in a pinned instrumented Triton validation image and capture a non-empty bounded trace. |
| NVIDIA KPI source | Use run-scoped capture from Triton's existing metrics endpoint in addition to Nsight trace evidence. |
| Concurrent DCGM Exporter | Do not enable in the default pinned Triton `24.01` metrics topology; evaluate only as an alternate single-authority mode after compatibility proof. |
| Intel required application | Install Intel PresentMon on Windows and require non-empty run-correlated capture output. |
| Intel execution proof | OpenVINO must report a specific Intel GPU execution device for assigned models. |
| Dashboard data plane | Persist normalized telemetry and serve authenticated REST/WebSocket projections; dashboards cannot depend on MCP. |
| MCP interface | Implement a mandatory read-only MCP service over normalized persisted evidence for authorized AI/operator clients. |
| Required input | `Raw Data/Arguing Students only/Arguing_004.mp4` is the initial real-video proof input. |

## Implementation Order

### 1. Add Tests Before Runtime Changes

Add failing tests for:

- Required telemetry application installation/readiness and missing-tool failure states.
- Triton metric sample parsing and capture-to-run correlation.
- Nsight Systems capture artifact generation and empty/unreadable trace rejection.
- OpenVINO explicit Intel device assignment/profiling proof.
- PresentMon capture artifact generation and empty/unreadable output rejection.
- Baseline/candidate telemetry-mode parity and measurement overhead reporting.
- Normalized telemetry ingestion plus authenticated summary, timeline, artifact, assignment, comparison REST/WebSocket projections.
- Dashboard functionality with MCP disabled or unavailable.
- MCP read-only resources/tools, Streamable HTTP security validation, RBAC/redaction, result bounds, denial auditing, and failure isolation.
- Blocking stage verdicts and artifact manifest readiness.
- Final-video playability, dimensions, timing, audio-presence parity, and non-overlay quality evidence.
- Required hybrid validation rejection when NVIDIA/Triton or Intel/OpenVINO proof is absent.

Execute affected suites one at a time:

```powershell
.\.venv\Scripts\python.exe -m pytest backend/tests/unit -n auto --dist=loadscope -q --tb=short
.\.venv\Scripts\python.exe -m pytest backend/tests/contract -n auto --dist=loadscope -q --tb=short
```

### 2. Install NVIDIA Nsight Systems In The Triton Validation Image

Implementation tasks:

1. Extend the existing Triton Docker build path with a pinned NVIDIA Nsight Systems CLI package in a dedicated instrumented validation image/profile.
2. Record the base Triton image identity, Nsight Systems version/package identity, and final validation-image digest in `telemetry_installation_manifest.json`.
3. Add a bounded `nsys` capture command path for the Triton validation process and write the report into the per-job evidence directory.
4. Fail required validation preflight when the application is not installed or not invocable.
5. Fail the required evidence gate when the Nsight report is missing, empty, or unreadable.

Planned readiness check after implementation:

```powershell
docker compose -f docker-compose.dev.yml --profile triton-offline run --rm triton-offline nsys --version
```

### 3. Configure Triton KPI Capture

Implementation tasks:

1. Add offline-specific metrics configuration, such as `TRITON_OFFLINE_METRICS_URL`, aligned to the offline Triton endpoint rather than the live endpoint.
2. Add a collector invoked around the offline benchmark execution window.
3. Scrape Triton's Prometheus metrics endpoint on a configured interval.
4. Persist raw samples to `accelerator_telemetry.jsonl` and aggregate values to `telemetry_summary.json`.
5. Attach collector availability and evidence references to the run manifest and job status summary.

Development readiness check after implementation:

```powershell
docker compose -f docker-compose.dev.yml --profile triton ps
Invoke-WebRequest -UseBasicParsing http://localhost:8102/metrics
```

Do not concurrently start DCGM Exporter in the default validation path while Triton GPU metrics remain the selected KPI source. Nsight Systems satisfies the mandatory in-container telemetry-application requirement without introducing that competing DCGM-agent topology.

### 4. Install And Configure Intel PresentMon On Windows

Implementation tasks:

1. Obtain Intel PresentMon from Intel's official distribution.
2. Install or unpack it into a configured local tooling location.
3. Add configurable executable path, sampling/output arguments, and evidence destination settings.
4. Verify it can produce a run-scoped telemetry artifact on the target Windows machine.
5. Persist tool version, installation/readiness state, output artifact path, and collector errors.
6. Fail the required validation run when PresentMon is missing, unavailable, or produces no usable capture evidence.

PresentMon is mandatory telemetry evidence. Hybrid placement still also depends on OpenVINO execution proof, because host activity alone does not bind model execution to the Intel device.

### 5. Enforce Explicit Intel OpenVINO Placement

Implementation tasks:

1. Enumerate OpenVINO devices at preflight and identify the intended Intel GPU device precisely.
2. Map Intel-assigned models to the explicit Intel `GPU.X` device identifier used in validation.
3. Persist requested and effective devices, model identity, worker identity, profiling results, and fallback behavior.
4. Fail the hybrid evidence gate if an Intel-assigned model cannot be shown to execute on the intended Intel GPU.

Representative preflight exploration during implementation:

```powershell
.\.venv\Scripts\python.exe -c "from openvino import Core; core=Core(); print(core.available_devices)"
```

### 5a. Implement Tiered Overload Buffering (RAM + Disk Spill)

Implementation tasks:

1. Add explicit buffering configuration with units:
   - `buffer.ram.max_frames`
   - `buffer.ram.max_bytes`
   - `buffer.spill.max_frames`
   - `buffer.spill.max_bytes`
   - `buffer.spill.path`
   - `buffer.overflow_policy` (`block`, `drop_oldest`, `drop_newest`)
2. Implement bounded in-memory queue and bounded disk-backed spill queue with frame-order preservation and traceable frame IDs.
3. Persist queue/spill metrics and events (queue depth, spill depth, spill latency, overflow decisions, dropped-frame counts/reasons).
4. Implement deterministic drain behavior after pressure recovers, preserving ordering and session/job state consistency.
5. Reject reliance on OS pagefile/virtual-memory tuning as a primary buffering strategy.
6. Add failure handling for disk-full, permissions, spill-write timeout, and spill corruption with explicit operator-visible status.

### 6. Normalize Telemetry And Provide Dashboard APIs

Implementation tasks:

1. Extend runtime ingestion/persistence so validated Nsight, Triton, PresentMon, OpenVINO, assignment, fidelity, and comparison evidence produces run-correlated normalized records.
2. Implement authenticated REST projections for telemetry summary, bounded timeline, artifact metadata, runtime assignment proof, and benchmark comparison.
3. Implement authorized WebSocket telemetry events for live/delayed and offline progress using the same redaction and bounded projection rules.
4. Validate dashboard operation with MCP disabled and when MCP is unavailable.
5. Keep raw trace files behind existing authorized artifact handling rather than serializing them into ordinary dashboard responses.

### 7. Implement The Read-Only MCP Observability Layer

Implementation tasks:

1. Add a separately configurable MCP service that reads normalized backend telemetry/evidence rather than scraping collector tools or unrestricted storage.
2. Use authenticated Streamable HTTP for deployed AI/operator clients, including Origin validation and bounded results; permit `stdio` only for authorized local development diagnostics.
3. Expose the six `telemetry://runs/{run_id}/...` resources and four read-only diagnostic tools defined in the [MCP contract](contracts/mcp-telemetry-server-contract.md).
4. Apply role checks, PII-safe projection, raw-artifact restrictions, and audit events for allowed and denied operations.
5. Reject and test `start_validation_capture`, shell execution, arbitrary profiler arguments, raw unrestricted file reads, and unbounded timeline requests.
6. Prove MCP disablement/failure/rejection has no effect on ingestion, inference, artifacts, or dashboard delivery.

Planned safe defaults:

```dotenv
TELEMETRY_MCP_ENABLED=false
TELEMETRY_MCP_TRANSPORT=streamable_http
TELEMETRY_MCP_MAX_TIMELINE_SAMPLES=1000
```

### 8. Correct Artifact And Export Completion Gates

Implementation must ensure:

- Pose inference failures outside policy prevent validation completion.
- All required artifacts are readable, hashed, and referenced from `artifact_manifest.json`.
- Final output is a playable MP4 file.
- Source audio presence is preserved when applicable.
- Source resolution, aspect ratio, nominal frame rate, and timeline duration are preserved within the contract tolerance.
- Non-overlay video quality passes the approved encoding baseline comparison.

### 9. Run Baseline And Optimized Offline Hybrid Validation

Use the same real video, enabled models, assignment manifest, mandatory telemetry mode, and output/evidence contract for both runs. Capture instrumentation overhead or an equivalent uninstrumented control result before accepting an optimization gain.

Existing benchmark entrypoint:

```powershell
.\.venv\Scripts\python.exe backend\manage.py benchmark_offline_video_job `
  --video "E:\grad_project\Raw Data\Arguing Students only\Arguing_004.mp4" `
  --pipeline-mode full_frame
```

Implementation must extend or wrap this command to:

- Select the offline hybrid profile.
- Run a full-model NVIDIA lane where all selected models from `.env` are bound to NVIDIA/Triton.
- Run a full-model Intel lane where all selected models from `.env` are bound to Intel GPU/OpenVINO.
- Run the instrumented Triton profile and start/stop mandatory Nsight Systems, Triton metrics, PresentMon, and OpenVINO profiling capture.
- Emit the required evidence package.
- Associate each optimized stress-lane run with its equivalent unoptimized baseline for that same lane.
- Persist normalized projections that can be retrieved from dashboard APIs with MCP disabled.

### 10. Evaluate Acceptance

A passing completion result requires:

| Gate | Passing Condition |
|------|-------------------|
| Runtime placement | Required stress lanes prove all `.env` selected models on NVIDIA/Triton in one run and all `.env` selected models on Intel/OpenVINO in another run. |
| Tiered buffering | RAM and disk spill queues are bounded, configurable, observable, and replay/drain behavior preserves frame-order consistency; overflow policy outcomes are persisted and reported. |
| Installed telemetry applications | Nsight Systems CLI in the instrumented Triton validation image and PresentMon on Windows have verified installation/version/readiness evidence. |
| Mandatory captures | Non-empty readable Nsight and PresentMon artifacts plus correlated Triton metrics and OpenVINO profiling evidence exist. |
| Stage completion | Every blocking stage passes with no unresolved error. |
| Pose completeness | Missing/gap policy in the specification passes. |
| Performance | Optimized hybrid p95 latency improves at least 20% over the equivalent unoptimized hybrid baseline and throughput is reported against target. |
| Artifacts | Required inference, pose, telemetry, reconciliation, benchmark, manifest, and video artifacts exist and are valid. |
| Measurement parity | Baseline and optimized runs use identical telemetry modes and report instrumentation overhead/control evidence. |
| Fidelity | Final annotated video passes playability, media-property, audio-presence, and non-overlay visual-quality checks. |
| Dashboard observability | Authenticated REST/WebSocket integration tests read normalized telemetry successfully while MCP is disabled or failed. |
| MCP observability | Enabled MCP contract/security tests return only bounded read-only authorized evidence and audit rejected/prohibited requests without pipeline impact. |

## Expected Evidence Directory

```text
backend/data/videos/{job_id}/
|-- run_manifest.json
|-- telemetry_installation_manifest.json
|-- stage_results.json
|-- nvidia_nsight_trace.nsys-rep
|-- intel_presentmon.csv
|-- accelerator_telemetry.jsonl
|-- telemetry_summary.json
|-- normalized_telemetry_summary.json
|-- benchmark_comparison.json
|-- frame_reconciliation.json
|-- artifact_manifest.json
|-- final_video_fidelity.json
|-- pose_results.json
|-- pose_quality_summary.json
`-- final_output_video.mp4
```

## Known Implementation Risks

- Triton readiness and available models alone do not prove that assigned model calls actually executed through NVIDIA/Triton.
- Nsight Systems instrumentation can change timing; comparisons must control and report this overhead.
- OpenVINO Intel execution still requires explicit effective `GPU.X` identity and profiling evidence.
- The current export implementation needs verification and likely correction for audio and source-fidelity requirements.
- Current job completion handling must be hardened so evidence or export failure cannot be reported as a successful completed validation run.
- PresentMon support and usable metrics must be tested on the actual Windows/Intel device environment; inability to capture prevents the required hybrid validation run from passing.
- A remote MCP deployment without Origin validation, authenticated authorization, bounded projections, and access auditing is not acceptable.
- MCP must remain outside the dashboard and inference critical paths; wiring frontend charts or capture control through MCP violates the specified boundary.
