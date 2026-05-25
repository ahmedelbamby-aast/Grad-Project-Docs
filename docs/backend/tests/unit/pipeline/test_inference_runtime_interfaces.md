# backend/tests/unit/pipeline/test_inference_runtime_interfaces.py

## Coverage

- Verifies unified runtime extension interfaces are exported as runtime-checkable protocols.
- Verifies `RuntimeContext` supports rollout metadata fields.

## Intent

Protect the `InferenceRuntime` contract surface from accidental interface regressions.

