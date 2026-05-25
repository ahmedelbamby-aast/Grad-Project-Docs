# Quickstart: Video Upload + Inference + Overlay

Feature: `004-video-upload-inference-tab`  
Last aligned: `2026-05-21`

This quickstart documents the implemented flow and avoids synthetic timings or fabricated payload values.

## Prerequisites

- Backend running (`uv run python manage.py runserver`)
- Celery worker running
- Frontend running (`npm run dev`)
- Authenticated user session in the UI or valid bearer token
- A real `.mp4` file available locally

## Canonical Environment Setup

Use the repository root setup first:

- `quickstart.md`

## Implemented Upload Flow

1. Open `/video-analysis` in the frontend.
2. Upload a `.mp4` file from the UI and choose `legacy_crop` or `full_frame`.
3. Confirm a job appears in the jobs list with `queued` or `processing` status.
4. Open the job details page and verify:
- progress updates appear
- output video/result surfaces render when complete
- overlay visibility toggles apply and persist for that job
- when available, `Annotated video` mode can be toggled and downloaded
- failed or partial jobs expose `Retry analysis` without leaving the page

## API Smoke Path (Optional)

Use this only if you need direct endpoint verification in parallel with UI checks.

```bash
curl -X POST http://localhost:8000/api/v1/video-analysis/jobs/ \
  -H "Authorization: Bearer $AUTH_TOKEN" \
  -F "video_file=@$TEST_VIDEO" \
  -F "title=Quickstart Upload"
```

Then poll status:

```bash
curl -X GET http://localhost:8000/api/v1/video-analysis/jobs/$JOB_ID/status/ \
  -H "Authorization: Bearer $AUTH_TOKEN"
```

When completed, validate output bundle:

```bash
curl -X GET http://localhost:8000/api/v1/videos/$JOB_ID/results \
  -H "Authorization: Bearer $AUTH_TOKEN"
```

## Release Validation Commands (Observed)

The following runs are captured as observed evidence for current behavior:

- Frontend: `npm run test:unit:parallel` -> 42 files passed, 398 tests passed (`2026-05-21`)
- Frontend: `npm run test:e2e:parallel` -> 6 passed, 0 failed (`2026-05-21`)
- Backend contract/integration subset for Triton upload path -> 29 passed (`spec 008 evidence`)

See:

- `specs/007-pose-behavior-pipeline/evidence/final/frontend-test-results.md`
- `specs/008-triton-upload-pipeline-optimization/evidence/final/implementation-evidence.md`
- `specs/008-triton-upload-pipeline-optimization/evidence/final/staged-load-validation.md`

## Known Gaps / Risks

- Full backend coverage gate evidence remains below target in older full-suite run (`62%` on `2026-05-15`).
- Several quality-gate marker suites are unavailable/missing tags in current repo state.
- Spec 008 load validation currently uses a deterministic simulated harness; real infra load evidence is still pending.

## Troubleshooting

If jobs remain stuck in `queued`/`processing`, use the worker reset runbook in `quickstart.md` and re-run upload validation.
