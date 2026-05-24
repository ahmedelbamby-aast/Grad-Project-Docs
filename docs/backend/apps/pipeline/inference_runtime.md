# backend/apps/pipeline/inference_runtime.py

## Purpose

Shared runtime event facade used by live and upload inference paths.

## Interface Contracts

- `SourceAdapter`
- `InferenceExecutor`
- `TrackingAndStudentLinker`
- `PredictionNormalizer`
- `ResultSink`

These runtime-checkable protocol contracts define the intended extension surface while preserving current event-emission behavior.

## Rollout Metadata

- `RuntimeContext` now carries rollout attributes:
  - `rollout_mode`
  - `rollout_reason`
- Emitted lifecycle events include rollout metadata in both JSONL audit rows and `raw_event` persistence payloads for runtime observability.
