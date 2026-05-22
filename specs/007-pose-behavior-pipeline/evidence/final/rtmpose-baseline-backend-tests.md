# RT004 Baseline Backend Tests

Date captured: 2026-05-17

## Baseline reference run (existing evidence)
Source: `specs/007-pose-behavior-pipeline/evidence/final/backend-test-results.md`

Command:
`./.venv/Scripts/python.exe -m pytest backend/tests --cov=backend/apps --cov-branch --cov-report=term-missing --cov-report=xml:backend/coverage.xml`

Result snapshot:
- Collected: 562
- Passed: 508
- Failed: 46
- Skipped: 8
- Runtime: 1742.68s
- Coverage summary: 62%

## RT067-RT071 targeted command (post-implementation)
`./.venv/Scripts/python.exe -m pytest backend/tests/system/test_temporal_stability_score.py backend/tests/system/test_temporal_noise_reduction_score.py backend/tests/system/test_temporal_continuity_score.py backend/tests/system/test_multi_person_long_session_continuity.py backend/tests/system/test_contrastive_tensor_export_sanity.py -n auto --dist=loadscope -q --tb=short`

Result: recorded in `rtmpose-test-results.md` (template pending full gate consolidation).
