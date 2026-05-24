# Backend Test Results

## Commands

| Command | Result |
| --- | --- |
| `.venv\Scripts\python.exe manage.py check` | Passed: no issues |
| `.venv\Scripts\python.exe -m pytest tests\unit tests\integration tests\contract tests\system -q --tb=short` | Timed out after 244s |
| `.venv\Scripts\python.exe -m pytest tests\unit -q --tb=short` | Failed: 177 passed, 7 skipped, 11 failed, 101 errors |
| `.venv\Scripts\python.exe -m pytest tests\contract -q --tb=short` | Failed: 31 passed, 9 errors |
| `.venv\Scripts\python.exe -m pytest tests\integration -q --tb=short` | Failed: 67 passed, 3 skipped, 3 failed, 10 errors |

## Blocking Causes

The broad backend suites are not green in this local environment. Main blockers were PostgreSQL test database contention (`test_exam_monitor` already exists/in use), missing `tensorrt` Python module, missing raw video asset `backend/data/videos/.../input.mp4`, and existing pipeline/tracker expectation failures.

Focused modularity suites passed earlier in this implementation evidence.
