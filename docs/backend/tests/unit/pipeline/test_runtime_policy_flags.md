# backend/tests/unit/pipeline/test_runtime_policy_flags.py

## Coverage

- Verifies cutover mode forces Triton requirement (no local fallback when Triton is unavailable).
- Verifies canary mode selects Triton with local fallback when Triton health is ready.
- Verifies shadow mode keeps default local fallback behavior when Triton is disabled.

## Intent

Protect rollout-control wiring (`shadow`, `canary`, `cutover`, `legacy_disable`) in runtime policy decisions.

