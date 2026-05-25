# Quickstart Guide: Exam Monitoring Dashboard

Feature: `001-exam-monitor-dashboard`  
Last aligned: `2026-05-21`

This spec quickstart now defers to the canonical repository quickstart and records only validated command paths.

## Canonical Startup Path

Use the repository root quickstart:

- `quickstart.md`

That document is the maintained source for:

- infrastructure startup (`docker-compose.dev.yml`)
- backend/frontend boot flow
- optional Triton profile
- Windows shutdown/recovery notes

## Minimum Validation Commands (Observed)

The following command classes were explicitly validated in `specs/007-pose-behavior-pipeline/evidence/final/quickstart-validation.md` on `2026-05-15`:

```powershell
docker compose -f docker-compose.dev.yml ps
uv sync --project backend
python backend/manage.py check
cd frontend; npm run type-check
cd frontend; npm run build
```

Observed result: all commands above passed after TypeScript compatibility fixes captured in the evidence file.

## Release Readiness References

- `docs/release-beta-signoff-checklist.md`
- `specs/007-pose-behavior-pipeline/evidence/final/quickstart-validation.md`
- `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-test-results.md`

