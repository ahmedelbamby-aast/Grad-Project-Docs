# Real-Data Test Policy

**Ref**: `specs/005-architecture-refactoring/tasks.md` (T034, US2)  
**Updated**: 2026-05-07

---

## Purpose

This document defines the policy governing the use of real raw video test
datasets in development, testing, and production environments. It is binding
for all contributors and CI pipelines.

---

## Policy Statement

> **Raw video test datasets are RESTRICTED to development (`dev`) and test
> (`test`) environments only. They are FORBIDDEN from production (`prod`) and
> staging (`staging`) environments.**

This policy is enforced programmatically by:
- `DatasetPolicyGuard` (`backend/tests/utils/dataset_policy_guard.py`)
- `DatasetPolicySettings` (`backend/core/settings_models.py`)
- The session-scoped `dataset_policy_guard` fixture (`backend/tests/conftest.py`)
- CI Stage 3 (`ci-staging-system.yml`) which verifies the policy before system tests

---

## Allowed Environments

| Environment | Raw datasets permitted | Notes |
|---|---|---|
| `dev` | ✅ YES | Local development, unit, and integration testing |
| `test` | ✅ YES | CI unit/integration runs |
| `staging` | ❌ NO | Use approved non-raw fixtures |
| `prod` | ❌ NO | Never — production safety boundary |

---

## Evidence Requirements per CI Stage

### Stage 1 (Bootstrap — Informational)
- No raw dataset access required.
- Unit and integration tests use mock/fixture data.

### Stage 2 (Blocking Merge Gate)
- All unit and integration tests must pass with mock data only.
- Coverage ≥ 85% must be achieved without raw video datasets.
- If a test requires real video data it must be guarded by `dataset_policy_guard`.

### Stage 3 (Staging Release Gate)
- System tests run in `dev`/`test` environment (never `prod`).
- `DatasetPolicyGuard.assert_raw_data_allowed()` is called at workflow start.
- Any test file using real video frames must include the `dataset_policy_guard` fixture.
- Evidence of guard verification is uploaded as a CI artifact.

---

## Developer Guidelines

### Writing tests that use real datasets

```python
import pytest
from tests.utils.dataset_policy_guard import DatasetPolicyGuard, PolicyViolationError

@pytest.mark.system
class TestWithRealData:
    def test_example(self, dataset_policy_guard):
        # fixture automatically enforces policy
        # real dataset access here is permitted in dev/test
        pass
```

### Checking policy in code

```python
from tests.utils.dataset_policy_guard import DatasetPolicyGuard

# Raises PolicyViolationError if environment is prod or staging
DatasetPolicyGuard.assert_raw_data_allowed()

# Non-raising check
if DatasetPolicyGuard.is_raw_data_allowed():
    # safe to access raw dataset
    pass
```

### CI release gate check

```bash
python scripts/ci/verify_release_gate.py \
  --phase staging-system \
  --triton-result success \
  --behavior-result success \
  --system-result success
```

---

## Violation Handling

If `DatasetPolicyGuard.assert_raw_data_allowed()` detects a forbidden
environment, it raises `PolicyViolationError` with:
- The violating environment name
- The policy rule that was broken
- A reference to this document

CI jobs configured with `INFERENCE_ENVIRONMENT=prod` will fail at the
policy guard step and must not be used for real dataset testing.

---

## Policy Updates

Any changes to this policy require:
1. Updated `DatasetPolicySettings` validator in `backend/core/settings_models.py`.
2. Updated `DatasetPolicyGuard` constants in `backend/tests/utils/dataset_policy_guard.py`.
3. Updated CI workflow environment variables.
4. Updated this document with rationale and date.

---

## Related Documents

- [ci-policy.md](ci-policy.md) — Overall CI test policy
- [data-model.md](../../specs/005-architecture-refactoring/data-model.md) — DatasetPolicy entity
- [deployment-topology-contract.md](../../specs/005-architecture-refactoring/contracts/deployment-topology-contract.md)
