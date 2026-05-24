# RT104 Quality Gates

Status: Partial complete.

## Security
- `npm audit --audit-level=high` -> pass (`0 vulnerabilities`).
- `npm run security:audit:high` -> pass (`0 vulnerabilities`).
- `.\.venv\Scripts\python.exe -m pip_audit` -> runnable path fixed (module invocation uses underscore, not hyphen).

## Accessibility / Visual / Smoke (Playwright tags)
- `npx playwright test --grep '@a11y' --workers=50%` -> no matching tests.
- `npx playwright test --grep '@visual' --workers=50%` -> no matching tests.
- `npx playwright test --grep '@smoke' --workers=50%` -> no matching tests.

## Canary / Smoke Markers (Backend)
- `python -m pytest -m canary -n auto --dist=loadscope -q --tb=short` -> failed during broad collection due Django settings context.
- `python -m pytest -m "smoke or canary" -n auto --dist=loadscope -q --tb=short` -> same blocker.

## Notes
Core unit/integration/contract/system suites passed and are recorded in `rtmpose-test-results.md`.

## Remediation / Risk Acceptance
- Remediation applied: `pip-audit` is now pinned in backend dependency manifests and invoked via `pip_audit` module path.
- Remediation applied: frontend repeatable security command added as `npm run security:audit:high`.
- Risk acceptance (temporary): no `@a11y`, `@visual`, or frontend `@smoke` tagged Playwright tests exist yet, so those gates are non-executable until tags are implemented in test files.
