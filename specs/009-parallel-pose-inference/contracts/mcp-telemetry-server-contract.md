# Contract: Read-Only MCP Telemetry Server

**Feature**: [Parallel Pose Inference with Hybrid Telemetry](../spec.md)  
**Data Model**: [data-model.md](../data-model.md)  
**Dashboard Contract**: [dashboard-telemetry-api-contract.md](dashboard-telemetry-api-contract.md)  
**Capture Contract**: [telemetry-capture-contract.md](telemetry-capture-contract.md)

## Purpose And Boundary

The MCP service is a mandatory AI/operator observability adapter over normalized persisted telemetry. It is not a dashboard backend, collector controller, arbitrary artifact reader, or inference dependency.

## Deployment And Transport

| Concern | Initial Contract |
|---------|------------------|
| Feature enablement | Separately configurable; dashboard and inference operation must pass with it disabled. |
| Protocol revision | Target MCP specification revision `2025-11-25`, validated through protocol initialization/capability negotiation. |
| Deployed transport | MCP Streamable HTTP on one authenticated endpoint supporting protocol `POST` and `GET`. |
| Local development | `stdio` may be used only for authorized local diagnostics; no production exposure by assumption. |
| Network security | Validate permitted `Origin` values, reject invalid origins, apply TLS at deployment boundary, and authenticate every remote connection. |
| Authorization | Apply application RBAC and least-privilege scopes to every resource read and tool invocation. |
| Audit | Record permitted and rejected operations without secrets or student-identifying response content. |

## Read-Only Resources

The service advertises resources or resource templates under the custom `telemetry://` scheme. All payloads use `application/json`, are derived from normalized persistence, and apply result bounds and PII-safe projection.

| Resource URI | Content | Required Bounds |
|--------------|---------|-----------------|
| `telemetry://runs/{run_id}/summary` | KPI summary, source availability, stage/verdict overview | Single authorized run |
| `telemetry://runs/{run_id}/timeline` | Downsampled timeline samples | Time window and server sample cap |
| `telemetry://runs/{run_id}/assignments` | Per-model runtime/device/worker proof | Authorized run only |
| `telemetry://runs/{run_id}/artifacts` | Manifest metadata and availability | Metadata only; no raw bytes/path traversal |
| `telemetry://runs/{run_id}/fidelity` | Final output media/fidelity verdict | Authorized run only |
| `telemetry://runs/{run_id}/comparison` | Baseline versus optimized evidence | Authorized comparison only |

## Read-Only Tools

| Tool | Input | Output | Required Controls |
|------|-------|--------|-------------------|
| `get_accelerator_health` | Optional approved profile/lane filter | Readiness and collector status summary | No collector command execution; normalized/current safe status only |
| `compare_hybrid_runs` | Authorized baseline and candidate run IDs | KPI deltas, parity checks, evidence verdicts | Reject non-comparable or unauthorized runs |
| `explain_run_failure` | Authorized run ID | Blocking failed stages and linked safe evidence metadata | No raw trace return; redact PII |
| `validate_evidence_package` | Authorized run ID | Missing/invalid required artifact and gate results | Read-only validation over persisted manifest |

Minimum tool result envelope:

```json
{
  "run_id": "uuid",
  "tool": "validate_evidence_package",
  "verdict": "passed",
  "evidence": {},
  "pii_safe": true,
  "generated_at": "ISO-8601"
}
```

## Explicitly Prohibited Initial Capabilities

- No `start_validation_capture` tool or any other mutating tool.
- No arbitrary shell, PowerShell, Docker, Nsight, PresentMon, Triton, or OpenVINO command execution.
- No arbitrary profiler arguments, raw local path input, unrestricted artifact read, or directory enumeration.
- No unbounded timeline or comparison query.
- No student-identifying output or bypass of backend role checks.

## Request Processing And Audit

1. Authenticate the MCP client and resolve its authorized application principal/role.
2. Validate protocol, transport security fields, resource URI or tool schema, and query bounds.
3. Authorize the requested run and projection class.
4. Query only normalized persisted evidence or authorized safe health state.
5. Redact prohibited fields and apply server-enforced bounds.
6. Return a read-only result or safe denial.
7. Persist an `McpAccessAuditEvent` for success, denial, invalid input, or internal failure.

## Isolation And Failure Semantics

| Condition | MCP Result | Application Requirement |
|-----------|------------|-------------------------|
| MCP unavailable or unhealthy | Return service-unavailable with audit/log evidence and health alarms | Inference, mandatory telemetry ingestion/persistence, and dashboards continue normally while MCP is restored under operational response. |
| Invalid origin or authentication | Reject request and audit when principal/context exists | No run-state or artifact mutation. |
| Unauthorized run/resource/tool | Deny and audit | No data leakage or dashboard impact. |
| Unbounded/raw/mutating request | Deny and audit as prohibited capability | No collector invocation or filesystem access. |
| Internal MCP failure | Safe error and operational log/audit | Processing and dashboard service remain isolated. |

## Contract And Security Tests

- MCP capability listing advertises only permitted resources and read-only tools.
- Streamable HTTP accepts supported protocol operations only after required network/authentication validation.
- Resource and tool results match normalized backend evidence and apply range/sample limits.
- Role/PII/raw-artifact/prohibited-tool requests are rejected and audited.
- `start_validation_capture`, arbitrary commands, arbitrary profiler arguments, and unbounded reads are absent and rejected.
- Disabling, overloading, or failing MCP does not interrupt processing, capture persistence, or dashboard REST/WebSocket delivery.
