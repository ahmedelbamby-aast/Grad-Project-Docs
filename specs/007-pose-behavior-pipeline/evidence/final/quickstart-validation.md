# T097 Quickstart Validation

Date: 2026-05-15

Validated path:
- `quickstart.md` (repository root canonical quickstart document)

Spec-local note:
- `specs/007-pose-behavior-pipeline/quickstart.md` is not present; validation used the canonical root quickstart.

Executed command set (quickstart-aligned):
- `docker compose -f docker-compose.dev.yml ps` -> PASS
- `uv sync --project backend` -> PASS
- `python backend/manage.py check` -> PASS
- `npm run type-check` (frontend) -> PASS
- `npm run build` (frontend) -> PASS after fixing TS config and test setup typing compatibility

Observed blocker and resolution during validation:
- Initial frontend build failed due to TypeScript deprecation option mismatch (`ignoreDeprecations`) and `ResizeObserver` mock typing in `frontend/vitest.setup.ts`.
- Resolved by updating:
  - `frontend/tsconfig.app.json`
  - `frontend/vitest.setup.ts`
- Re-ran build successfully after these fixes.

Conclusion:
- Quickstart validation is complete against the canonical quickstart path, with all executed validation commands passing.
