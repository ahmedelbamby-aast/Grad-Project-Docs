# backend/tests/unit/pipeline/test_runtime_scaffold_models.py

## Coverage

- Verifies raw-truth scaffold model fields required for partitioned lifecycle storage.
- Verifies rollup scaffold model fields required for aggregate runtime dashboards.

## Intent

Protect migration scaffolding entities from accidental schema regression while phase-2 ingestion/rollup logic is still in progress.

