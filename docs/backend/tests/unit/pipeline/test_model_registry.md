# backend/tests/unit/pipeline/test_model_registry.py

## Coverage

- Validates known model-to-task mappings in the descriptor registry.
- Validates unknown model behavior returns `None`.
- Validates descriptor metadata includes overlay capability and Triton support.

## Intent

Protect registry-driven runtime wiring from regressions when adding or renaming model keys.

