# Deployment And Startup Validation Governance

## Purpose

This document defines production deployment validation for the native Linux runtime, dual Triton endpoint profiles with one active mode, endpoint authority, startup readiness graph, model repository integrity, and fail-closed behavior.

## Deployment Constraints

Production deployment MUST satisfy:

- Linux runtime
- no Docker dependency in production runtime path
- no sudo operational assumption
- Triton-only inference
- GPU-backed execution
- dual configured Triton endpoint profiles
- exactly one active production mode selected by environment
- probe-backed health and readiness

## Active Mode Selection

`TRITON_EXECUTION_MODE` MUST be one of:

```env
TRITON_EXECUTION_MODE=live
TRITON_EXECUTION_MODE=offline
```

Startup MUST reject unset, unknown, mixed, or contradictory modes. The inactive endpoint profile may exist in configuration, but it MUST NOT be routed, scheduled, or reported production-ready.

## Startup Readiness Graph

Startup preflight MUST validate:

1. environment schema
2. active runtime mode
3. active endpoint reachability
4. inactive endpoint isolation
5. active metrics endpoint reachability
6. Triton model repository digest
7. model version manifest
8. `config.pbtxt` schema
9. RTMPose IO names and shapes
10. model warmup shape compatibility
11. TensorRT engine lineage
12. Redis connectivity
13. Celery routing map
14. DB connectivity
15. backend health
16. frontend health when required for full-stack validation
17. GPU visibility
18. production local fallback disabled

## Hard Startup Failures

Startup MUST fail when:

- Triton ready mismatch exists
- `required_offline=True required_live=True reason=forced_triton_not_ready` appears
- active endpoint unavailable
- inactive endpoint receives production traffic
- active mode ambiguous
- model version drift detected
- `config.pbtxt` invalid
- RTMPose warmup mismatch detected
- TensorRT engine lineage invalid
- GPU not visible for production inference
- local fallback is enabled in production

## Machine-Readable Diagnostics

Preflight failure output MUST include:

```json
{
  "runtime_mode": "live|offline",
  "failed_node": "string",
  "reason": "string",
  "active_endpoint": "string",
  "inactive_endpoint": "string",
  "model_repository_digest": "string|null",
  "engine_lineage_digest": "string|null",
  "remediation_hint": "string"
}
```

## Evidence Requirements

Deployment validation MUST produce:

```text
ci_evidence/production/wave1/hash_parity.md
ci_evidence/production/wave1/endpoint_health.md
ci_evidence/production/wave1/ports_snapshot.txt
ci_evidence/production/wave1/backend_health.json
ci_evidence/production/wave1/model_serving_health.json
ci_evidence/production/wave1/boundary_verifier.txt
ci_evidence/production/runtime/triton_repository_validation.json
ci_evidence/production/runtime/runtime_mode_validation_report.md
```
