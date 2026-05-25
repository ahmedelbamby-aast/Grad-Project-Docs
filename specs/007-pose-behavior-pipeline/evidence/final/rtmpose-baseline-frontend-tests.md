# RT005 Baseline Frontend Tests

Date captured: 2026-05-17

## Baseline reference run (existing evidence)
Source: `specs/007-pose-behavior-pipeline/evidence/final/frontend-test-results.md`

Commands snapshot:
- `npm test` -> FAILED (camera grid tests; `ResizeObserver is not defined`)
- `npm run type-check` -> PASSED
- `npm run build` -> PASSED
- `npm run test:e2e` -> FAILED (`Camera Feeds` heading + `ECONNREFUSED` flow)

## Notes
- RT067-RT074 backend-only changes do not alter frontend test contracts.
- Final frontend gate evidence remains tracked in RT102/RT104 templates.
