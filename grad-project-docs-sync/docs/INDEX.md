# Documentation Index

**Last Updated**: 2026-05-21

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
