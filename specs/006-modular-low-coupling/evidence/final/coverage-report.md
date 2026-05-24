# Coverage Report

## Frontend

`npm test -- --coverage --run` executed 202 passing tests, but failed configured thresholds:

- Statements: 62.99%
- Branches: 54.96%
- Functions: 64.4%
- Lines: 64.14%

Approved temporary exception: see `plan.md` Complexity Tracking and `docs/architecture/coupling-risk-register.md`.

## Backend

Backend full coverage could not be generated because broad pytest suites are blocked by local PostgreSQL test database contention, missing TensorRT Python module, and missing raw video assets. Approved temporary exception recorded in `plan.md`.
