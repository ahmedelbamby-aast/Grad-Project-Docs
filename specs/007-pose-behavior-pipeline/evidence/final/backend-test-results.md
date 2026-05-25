# T087 Backend Full Test Suite + Coverage

Date: 2026-05-15

Command:
`./.venv/Scripts/python.exe -m pytest backend/tests --cov=backend/apps --cov-branch --cov-report=term-missing --cov-report=xml:backend/coverage.xml`

Result: FAILED

Summary:
- Collected: 562
- Passed: 508
- Failed: 46
- Skipped: 8
- Runtime: 1742.68s
- Total coverage (line+branch report summary): 62%
- Coverage XML generated: `backend/coverage.xml`

Notable failing areas:
- Progressive preview / video export flow
- Triton outage handling and graceful degradation
- Behavior category and e2e Triton system tests
- Tracking/rendering contract expectations
- Evidence pack scaffold expectation

Gate impact:
- Phase 7 backend gate execution complete (run performed, output captured).
- Coverage gate target (100% line + branch) not met; see `coverage-gate.md` and `coverage-exceptions.md`.
