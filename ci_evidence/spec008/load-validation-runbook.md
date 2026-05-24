# Spec 008 Load/Perf Evidence Runbook

Date: 2026-05-21

## Scope
- Staged concurrency comparison: low / medium / high
- Before/after profile format
- Starvation check for non-upload queue

## Partial Results (Infra-Independent)
Executed:
```powershell
python load-tests/staged_validation.py
```
Generated artifacts:
- `ci_evidence/spec008/staged-load-results.json`
- `specs/008-triton-upload-pipeline-optimization/evidence/final/staged-load-validation.md`

Key outcomes:
- Upload P95 improved in low/medium/high after-profile stages.
- Non-upload queue wait P95 improved in all after-profile stages.
- Starvation verdict for non-upload queue in after-profile: `FALSE` across all stages.

## Reproducible Commands (When Full Infra Is Available)
1. Bring dependencies up:
```powershell
docker compose -f docker-compose.dev.yml up -d postgres redis
```

2. Execute staged harness (always runnable):
```powershell
python load-tests/staged_validation.py
```

3. Run performance and resilience tests:
```powershell
.\.venv\Scripts\python.exe -m pytest backend/tests/performance -n auto --dist=loadscope -q --tb=short
.\.venv\Scripts\python.exe -m pytest backend/tests/resilience -n auto --dist=loadscope -q --tb=short
```

4. Optional k6/Locust smoke scaffolds:
```powershell
k6 run --vus 50 --duration 5m load-tests/baseline.js
k6 run load-tests/scenarios/stress.js
locust -f load-tests/locustfile.py --headless -u 200 -r 20 -t 10m --processes -1
```

## Notes
- Current evidence uses simulated harness mode because Triton/Celery end-to-end infra validation was not executed in this worker run.
- JSON artifact is designed for CI diffing between before/after profiles.

## Quick Rollback / Safe Mode
If upload throughput regresses or queue latency spikes, revert to the conservative profile first:

```powershell
$env:TRITON_OFFLINE_BATCH_QUEUE_ENABLED = "0"
$env:TRITON_UPLOAD_TARGET_BATCH_SIZE = "1"
$env:TRITON_UPLOAD_MAX_BATCH_WAIT_MS = "10"
$env:TRITON_SHADOW_PRECHECK_MAX_CONCURRENCY = "1"
$env:CELERY_WORKER_CONCURRENCY = "2"
$env:CELERY_WORKER_PREFETCH_MULTIPLIER = "1"
```

Then restart the backend worker and re-run the staged harness:

```powershell
python load-tests/staged_validation.py
```

## Recommended Production Defaults

Single GPU:
```text
CELERY_WORKER_CONCURRENCY=2
CELERY_WORKER_PREFETCH_MULTIPLIER=1
TRITON_UPLOAD_TARGET_BATCH_SIZE=8
TRITON_UPLOAD_MAX_BATCH_WAIT_MS=40
TRITON_OFFLINE_BATCH_QUEUE_MAX_PENDING=256
TRITON_MAX_INFLIGHT_REQUESTS=8
TRITON_SHADOW_PRECHECK_MAX_CONCURRENCY=1
```

Multi-GPU:
```text
CELERY_WORKER_CONCURRENCY=4
CELERY_WORKER_PREFETCH_MULTIPLIER=1
TRITON_UPLOAD_TARGET_BATCH_SIZE=8
TRITON_UPLOAD_MAX_BATCH_WAIT_MS=40
TRITON_OFFLINE_BATCH_QUEUE_MAX_PENDING=512
TRITON_MAX_INFLIGHT_REQUESTS=16
TRITON_SHADOW_PRECHECK_MAX_CONCURRENCY=2
```

## Release Watch Window

- Observe for first `30 minutes` post-deploy, then re-check at `+24 hours`.
- Alert thresholds:
  - Upload latency P95 > `1.25x` pre-deploy baseline for 10+ minutes.
  - Non-upload queue wait P95 > `800ms`.
  - Timeout/retry/failure counters show sustained increase.
- If threshold trips, apply rollback env profile above, restart worker processes, and rerun staged validation.
