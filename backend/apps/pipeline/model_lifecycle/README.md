# Model Lifecycle

Model inventory, export, benchmark, deployment, and Triton repository utilities.

## Purpose

This package manages model discovery and runtime-target selection for the YOLO pipeline.

## Public API

- `scan_model_inventory()`
- `detect_runtime_capabilities()`
- `ExportOrchestrator`
- `ExportWorker`
- `BenchmarkRunner`
- `BenchmarkOrchestrator`
- `DeploymentMatrix`
- `TritonAdapter`

## Dependencies

- `apps.pipeline.config.PipelineConfig`
- `apps.pipeline.services.PipelineService`
- `opencv-python`
- `ultralytics`

## Usage

```python
from apps.pipeline.model_lifecycle import scan_model_inventory
```

## Related Documents

- [../README.md](../README.md) - Pipeline package overview
- [../../../../docs/backend/apps/pipeline/model_lifecycle/__init__.md](../../../../docs/backend/apps/pipeline/model_lifecycle/__init__.md) - Lifecycle package docs
- [../../../../docs/backend/apps/pipeline/model_lifecycle/inventory.md](../../../../docs/backend/apps/pipeline/model_lifecycle/inventory.md) - Model inventory scanning
- [../../../../docs/backend/apps/pipeline/model_lifecycle/capabilities.md](../../../../docs/backend/apps/pipeline/model_lifecycle/capabilities.md) - Runtime capability probing
- [../../../../docs/backend/apps/pipeline/model_lifecycle/export_orchestrator.md](../../../../docs/backend/apps/pipeline/model_lifecycle/export_orchestrator.md) - Export planning
- [../../../../docs/backend/apps/pipeline/model_lifecycle/benchmark_runner.md](../../../../docs/backend/apps/pipeline/model_lifecycle/benchmark_runner.md) - Benchmark replay runner
- [../../../../docs/backend/apps/pipeline/model_lifecycle/deployment_matrix.md](../../../../docs/backend/apps/pipeline/model_lifecycle/deployment_matrix.md) - Deployment ranking
- [../../../../docs/backend/apps/pipeline/model_lifecycle/triton_adapter.md](../../../../docs/backend/apps/pipeline/model_lifecycle/triton_adapter.md) - Triton repository generation
- [../../../../.specify/memory/constitution.md](../../../../.specify/memory/constitution.md) - Project governance
