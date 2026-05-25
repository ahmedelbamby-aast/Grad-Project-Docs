# Triton Runtime Governance

## Purpose

This document defines Triton runtime correctness requirements for timeout handling, dynamic batching, partial-batch salvage, model readiness, model repository validation, TensorRT lineage, RTMPose configuration, and inference-path attribution.

It exists because live benchmark execution showed timeout spikes near 3300 ms, wall latency dominated by orchestration while GPU utilization stayed near zero, partial batch resolution failures triggered single-call fallback paths, and readiness checks produced contradictory live/offline requirements.

## Endpoint Authority

The system MUST preserve separate live and offline Triton endpoint profiles while enforcing exactly one active production execution mode.

| Profile | HTTP | gRPC | Metrics | Production Use |
|---------|------|------|---------|----------------|
| live | 39000 | 39001 | 39002 | active only when `TRITON_EXECUTION_MODE=live` |
| offline | 39100 | 39101 | 39102 | active only when `TRITON_EXECUTION_MODE=offline` |

Inactive endpoints MUST NOT receive production inference traffic, scheduler requests, Celery task routing, or production-ready health status.

## Startup Readiness Graph

Startup MUST validate a readiness graph with these nodes:

```text
runtime_mode
endpoint_authority
active_endpoint_reachable
inactive_endpoint_isolated
triton_model_repository
model_version_manifest
config_pbtxt_schema
warmup_shape_validation
TensorRT_engine_lineage
RTMPose_binding_validation
metrics_endpoint
```

Startup MUST fail if any required node fails. A readiness output such as `required_offline=True required_live=True reason=forced_triton_not_ready` is a hard failure because it contradicts single active production mode semantics.

## Triton Repository Consistency

The repository validator MUST check:

- model names expected by runtime policy
- model versions expected by deployment manifest
- TensorRT engine file digests
- generated `config.pbtxt` compatibility
- warmup input names and shapes
- RTMPose input/output tensor names
- RTMPose `simcc_x` and `simcc_y` output dimensions
- dynamic batching configuration per live/offline profile
- metrics availability for active endpoint

Evidence MUST include `ci_evidence/production/wave4/rtmpose_config_validation.txt` and `ci_evidence/production/runtime/triton_repository_validation.json`.

## Timeout And Retry Semantics

A Triton timeout is not automatically a retry authorization. The inference runtime MUST classify the timeout before any retry:

| Timeout Context | Allowed Response |
|-----------------|------------------|
| Triton internal queue saturation | mark Triton queue pressure and allow bounded retry if policy permits |
| application orchestration delay | do not retry Triton blindly; attribute orchestration overhead |
| GPU compute over SLO | mark GPU compute breach and evaluate dynamic batching profile |
| item-level crop failure | use partial-batch salvage, not frame-wide retry |
| endpoint readiness failure | fail closed; no fallback inference |

Every retry triggered after a Triton timeout MUST persist retry lineage and inference path.

## Partial-Batch Failure Governance

Partial-batch execution MUST support batch salvage. Required semantics:

- successful items are preserved
- unresolved items are isolated
- failed items are explicitly tagged
- unresolved indexes include `unresolved_index_origin`
- retries are per-item, not frame-wide
- retry generation increments per item
- source batch ID remains attached to every item
- duplicate dispatch ratio is measured and bounded

Forbid:

- invalidating a full frame because one crop failed
- redispatching every unresolved item through uncontrolled single-call fallback
- merging successful and failed item telemetry without item status
- silently reporting partial success as full success

Acceptance evidence MUST include `ci_evidence/production/runtime/fallback_resolution_report.md` and `ci_evidence/production/wave4/pose_batch_partial_success.json`.

## Inference Path Attribution

Every Triton or fallback inference event MUST include:

```text
inference_path: batch | single_fallback | degraded | retry | timeout_recovery
batch_id
source_batch_size
effective_batch_size
retry_generation
unresolved_index_origin
```

This attribution MUST flow into runtime telemetry, pose artifacts, benchmark evidence, and forensic traces.

## Dynamic Batching Efficiency

Benchmark acceptance MUST measure dynamic batching efficiency, not only end-to-end wall time. Required fields:

| Metric | Purpose |
|--------|---------|
| source batch size | expected batch opportunity |
| effective batch size | actual batch size sent to Triton |
| Triton queue delay | time waiting inside Triton scheduler |
| dynamic batch wait | scheduler batching wait contribution |
| model execution time | GPU execution contribution |
| fallback-path ratio | portion bypassing main batch path |
| GPU duty cycle | amount of wall time GPU is doing useful work |
| idle GPU windows | periods where orchestration prevents GPU use |

Low GPU utilization with high wall latency invalidates throughput claims unless these fields identify the root cause.

## Required Evidence

```text
ci_evidence/production/runtime/latency_decomposition.json
ci_evidence/production/runtime/gpu_utilization_analysis.md
ci_evidence/production/runtime/orchestration_overhead_report.md
ci_evidence/production/runtime/timeout_correlation_report.md
ci_evidence/production/runtime/retry_amplification_matrix.json
ci_evidence/production/wave8/dynamic_batch_signoff.md
```
