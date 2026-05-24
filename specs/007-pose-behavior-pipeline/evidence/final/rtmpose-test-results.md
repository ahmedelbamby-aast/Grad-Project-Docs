# RT102 Test Results

Status: Partial complete (core suites passed; some marker-specific suites unavailable in current repo/runtime).

## Executed Commands and Outcomes
- `./.venv/Scripts/python.exe -m pytest backend/tests/unit/pipeline backend/tests/unit/tracking -n auto --dist=loadscope -q --tb=short` -> `211 passed, 6 skipped`
- `npm run test:unit:parallel` -> `41 files passed, 395 tests passed`
- `./.venv/Scripts/python.exe -m pytest backend/tests/integration -n auto --dist=loadscope -q --tb=short` -> `92 passed, 1 skipped`
- `npm run test:all:parallel` -> `type-check passed, unit passed, e2e 6 passed`
- `./.venv/Scripts/python.exe -m pytest backend/tests/contract -n auto --dist=loadscope -q --tb=short` -> `44 passed`
- `./.venv/Scripts/python.exe -m pytest backend/tests/system -n auto --dist=loadscope -q --tb=short` -> `52 passed, 5 skipped`

## Targeted RTMPose Added/Changed Tests
- `./.venv/Scripts/python.exe -m pytest <32 targeted RTMPose test files> -n auto --dist=loadscope -q --tb=short` -> `45 passed`

## Scientific Validation Suite
- `./.venv/Scripts/python.exe -m pytest backend/tests/system/test_temporal_stability_score.py backend/tests/system/test_temporal_noise_reduction_score.py backend/tests/system/test_temporal_continuity_score.py backend/tests/system/test_multi_person_long_session_continuity.py backend/tests/system/test_contrastive_tensor_export_sanity.py -n auto --dist=loadscope -q --tb=short` -> `5 passed`

## Blocked/Unavailable in This Run
- `backend/tests/performance` and `backend/tests/resilience`: no tests collected.
- Marker sweep (`-m canary`, `-m "smoke or canary"`) failed from current root run context due Django settings/import-time collection constraints.
