# Documentation Index

**Last Updated**: 2026-06-02

> **Reading-order note (2026-06-02):** the canonical reading order through
> every project-owned narrative doc lives in the README under
> ["Documentation Reading Order"](../README.md#documentation-reading-order-read-these-in-this-order-to-understand-the-complete-project).
> If a doc's `**Last updated:**` header disagrees with where it appears
> in that reading order, **the reading order is authoritative.** Use this
> file as a topical index; use the README for the narrative reading path.

## Inference Pipeline Optimization (active work)

- [Cycle 9 + Cycle 10 Improvements TODO — the single entry point](cycle_9_and_10_improvements_todo.md)
- [Cycles 9–12 Implementation Playbook](cycles_9_to_12_implementation_playbook.md)
- [Runtime SLA: video + 5 min](runtime_sla_video_plus_5min.md)
- [Triton Models & Tensor Anatomy](triton_models_and_tensor_anatomy.md)
- [Inference Parallelization Plan](inference_parallelization_plan.md)
- [Inference Bottlenecks & Solution Matrix](inference_bottlenecks_and_solution_matrix.md)
- [Production Inference Benchmark](production_inference_benchmark.md)
- [Cycle 9 Investigation](cycle_9_investigation.md)
- [Cycle 9 Results (post-mortem — NOT ACCEPTED)](cycle_9_results.md)
- [Cycle 9b Output Fusion Investigation](cycle_9b_output_fusion_investigation.md)
- [Cycle 9b Output Fusion Results](cycle_9b_output_fusion_results.md)
- [Cycle 9b Exact Slice Investigation](cycle_9b_exact_slice_investigation.md)
- [Cycle 9b Top-K Anchor Packing Investigation](cycle_9b_topk_anchor_packing_investigation.md)
- [Cycle 9b Top-K Anchor Packing Results — current accepted baseline](cycle_9b_topk_anchor_packing_results.md)
- [Cycle 9b Child Critical-Path Results (Step 1, pre-Top-K)](cycle_9b_child_critical_path_results.md)
- [Cycle 9b Child Critical-Path REMEASURE vs Top-K (Step 1 redo)](cycle_9b_child_critical_path_remeasure_topk_results.md)
- [Logical Path Matrix Spec (Cycle 10)](logical_path_matrix_spec.md)
- [Cycle 10 Investigation](cycle_10_investigation.md)
- [Cycle 10 LPM Phase 1 Results — NOT ACCEPTED](cycle_10_lpm_phase1_results.md)
- [Cycle 11 Input Size Investigation](cycle_11_input_size_investigation.md)
- [Cycle 11 Input Size Results — real benchmark rejection](cycle_11_input_size_results.md)
- [Next Agent Starter Prompt](next_agent_starter_prompt.md)
- [YOLOE / Depth Anything v2 Timing Decision — DEFERRED](new_models_yoloe_depth_anything_v2_timing_decision.md)
- [Crop-Frame Optimization Audit](crop_frame_optimization_audit.md)
- [Crop-Frame Optimization Execution Log](crop_frame_optimization_execution.md)
- [Crop-Frame RTX 5090 Bottleneck Investigation](crop_frame_rtx5090_bottleneck_investigation.md)
- [Crop GPU vs CPU Comparison](crop_gpu_vs_cpu_comparison.md)
- [RTT Root-Cause Investigation (job 77650001)](rtt_root_cause_investigation_77650001.md)

## Architecture Spine

- [Architecture Overview](ARCHITECTURE.md)
- [System Mermaid Atlas](diagrams/SYSTEM_MERMAID_ATLAS.md)
- [Mermaid Style System](diagrams/MERMAID_STYLE_SYSTEM.md)
- [Source File Mirror](diagrams/SOURCE_FILE_MIRROR.md)
- [Diagram Compiler Plan](diagram_compiler_plan.md)
- [Generated Diagrams Guide](diagrams/README_GENERATED.md)

## Backend Architecture

- [Data Flow — API, Streaming, Inference, Runtime Telemetry](backend/architecture/data-flow.md)
- [Deployment Topology](backend/architecture/deployment-topology.md)
- [Observability Runbook](backend/architecture/observability-runbook.md)
- [Triton Operations](backend/architecture/triton-operations.md)

## Backend App Deep Dives

- [Video Analysis App](backend/apps/video_analysis/README.md)
- [Tracking App](backend/apps/tracking/README.md)
- [API Reference](backend/api/README.md)
- [Video Upload Sequence](backend/VIDEO_UPLOAD_SEQUENCE.md)
- [Job State Diagram](backend/JOB_STATE_DIAGRAM.md)

## Frontend

- [Frontend Components](frontend/src/components/README.md)

## Infrastructure & Runtime

- [Systemd Triton Service](infra/systemd/triton-server.md)
- [Triton Dockerfile Doc](infra/docker/triton/Dockerfile.md)
- [Triton Inference Speed Stabilization Plan](triton_inference_speed_stabilization_plan.md)
- [Triton Strategy Implementation Matrix](triton_strategy_implementation_matrix.md)
- [Triton Throughput and Latency GPU Optimization](triton_throughput_latency_gpu_optimization.md)
- [Scripts: Triton Sync](scripts/sync-triton-tensorrt-repository.md)
- [Scripts: TensorRT Export](scripts/export-tensorrt-models.md)
- [Scripts: CI Docs Gate](scripts/ci/verify_docs_diagrams.md)
- [Scripts: CI Generated Diagram Gate](../scripts/ci/verify_generated_diagrams.py)
- [Scripts: CI Triton Memory Pool Sweep](scripts/ci/triton_memory_pool_sweep.md)
- [Scripts: CI Triton Memory Pool Sweep Wrapper](scripts/ci/run_triton_memory_pool_sweep.md)
- [Workflow: CI Bootstrap (includes Triton stabilization sweep)](../.github/workflows/ci-bootstrap.yml)

## Testing & CI

- [CI Policy](backend/testing/ci-policy.md)
- [Real-Data Test Policy](backend/testing/real-data-test-policy.md)

## Modular Low-Coupling Documents

- [Modular system overview](architecture/modular-system-overview.md)
- [Module boundary map](architecture/module-boundary-map.md)
- [Runtime scenario matrix](architecture/runtime-scenario-matrix.md)
- [Compatibility contracts](architecture/compatibility-contracts.md)
- [Coupling risk register](architecture/coupling-risk-register.md)
- [Documentation diagram coverage](architecture/documentation-diagram-coverage.md)

## Spec Cross-References

| Feature | Spec | Plan | Tasks |
|---|---|---|---|
| 004 - Video upload inference tab | [spec.md](../specs/004-video-upload-inference-tab/spec.md) | [plan.md](../specs/004-video-upload-inference-tab/plan.md) | [tasks.md](../specs/004-video-upload-inference-tab/tasks.md) |
| 006 - Modular low coupling | [spec.md](../specs/006-modular-low-coupling/spec.md) | [plan.md](../specs/006-modular-low-coupling/plan.md) | [tasks.md](../specs/006-modular-low-coupling/tasks.md) |
| 005 - Architecture refactoring | [spec.md](../specs/005-architecture-refactoring/spec.md) | [plan.md](../specs/005-architecture-refactoring/plan.md) | [tasks.md](../specs/005-architecture-refactoring/tasks.md) |
