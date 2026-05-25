# Production Queue Workers (No Docker / No sudo)

This folder contains production-native scripts to launch dedicated Celery workers per queue and avoid queue contention between live/offline control paths and model-specific dispatch paths.

The active implementation wave for this runbook is the behavioral maturity
closure feature at `specs/010-behavioral-maturity-closure/`. Production helper
scripts are expected to emit evidence that distinguishes active profile,
inactive profile isolation, real versus mock execution, and runtime mode.

## 1) Preflight ports and runtime env

Run on production host:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_port_preflight.sh
```

If ports are occupied, override in `backend/.env` before start:

- `BACKEND_PORT` (default: `8011`)
- `FRONTEND_PORT` (default: `5174`)
- `TRITON_LIVE_HTTP_PORT` (default: `39000`)
- `TRITON_OFFLINE_HTTP_PORT` (default: `39100`)

Backend runtime entrypoints load only `backend/.env`. Repo-root `.env` must not be relied on for backend startup or health validation on production.
Celery workers now follow the same rule through `backend/config/celery.py`, so worker startup and Django startup share one production env authority.

## 1.1) Enforce Triton single-endpoint policy (M5)

Run on production host:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_triton_endpoint_policy.sh --profile offline
```

or for live profile:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_triton_endpoint_policy.sh --profile live
```

This check fails unless all policy rules pass:
- only one endpoint is active (`200` on selected endpoint, inactive endpoint not `200`)
- inactive endpoint port is not listening
- `TRITON_LIVE_URL` and `TRITON_OFFLINE_URL` both point to active endpoint
- each model `config.pbtxt` matches selected profile config (`configs/live.pbtxt` or `configs/offline.pbtxt`)

Wave 1 acceptance also requires:
- `TRITON_EXECUTION_MODE` matches the selected `--profile`
- `TRITON_URL` points to the same active endpoint
- the backend health snapshot records explicit active/inactive readiness targets

Backend health interpretation on production:
- `/api/v1/health/model-serving/` is the authoritative Wave 1 Triton contract
- `/api/v1/health/` may report `redis=degraded` when the Redis service is reachable but the runtime lacks valid credentials; this is not equivalent to Redis process-down

From the Windows workstation you can write the Wave 1 evidence bundle with:

```powershell
.\tools\prod\prod-wave1-evidence.ps1 -Mode offline
```

## 2) Start dedicated workers

Run on production host:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_start_celery_workers.sh
```

This starts the following detached workers (queue -> concurrency):

- `pipeline.live.control` -> `1`
- `pipeline.live.person_detector.worker` -> `1`
- `pipeline.live.rtmpose_model.worker` -> `1`
- `pipeline.live.behavior.worker` -> `1`
- `video_inference` (offline control alias) -> `2`
- `pipeline.offline.person_detector.worker` -> `2`
- `pipeline.offline.rtmpose_model.worker` -> `2`
- `pipeline.offline.behavior.worker` -> `2`
- `default` -> `1`

Wave 2 queue governance notes:
- active/inactive control routing is resolved from `apps.pipeline.queue_routes`
- dead-letter names follow the `.dlq` suffix contract
- offline upload dispatch resolves its primary queue from runtime queue authority instead of legacy hardcoded queue names
- `.\tools\prod\prod-wave2-ingestion-evidence.ps1` captures the Wave 2 queue/drop/reconnect evidence bundle from Windows
- the Wave 2 prod pytest wrapper runs with `DJANGO_SETTINGS_MODULE=config.settings.test` and `USE_POSTGRES_TEST_DB=0` so evidence collection does not depend on PostgreSQL test-database create privileges on the server

Wave 5 behavior evidence:
- run `.\tools\prod\prod-wave5-behavior-evidence.ps1` from Windows to generate
  `ci_evidence/production/wave5/<timestamp>/` artifacts on production.

## 3) Stop all queue workers started by this repo

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_stop_celery_workers.sh
```

## 4) Verify active queue binding

```bash
cd /home/bamby/grad_project/backend
source .venv/bin/activate
celery -A config inspect active_queues
celery -A config inspect stats
```

## 5) Optional: run from Windows laptop

```powershell
.\tools\prod\prod-ssh.ps1 "cd /home/bamby/grad_project && bash tools/prod/prod_port_preflight.sh"
.\tools\prod\prod-ssh.ps1 "cd /home/bamby/grad_project && bash tools/prod/prod_start_celery_workers.sh"
```

## 6) M4 live cadence/reuse/telemetry validation (Windows helper)

Run from `E:\grad_project`:

```powershell
.\tools\prod\prod-validate-m4-live.ps1
```

This executes:
- backend and model-serving health checks
- active/inactive runtime mode summary from `backend/.env`
- Triton live/offline readiness checks
- targeted M4 test commands on production backend:
  - `tests/unit/video_analysis/test_tasks_detection_cadence.py`
  - `tests/unit/video_analysis/test_live_detection_policy_counters.py`
  - `tests/integration -k "timeout and live and reuse"`
  - `tests/resilience -k "triton_timeout"`

## 7) Wave 6 observability evidence

Run from `E:\grad_project`:

```powershell
.\tools\prod\prod-wave6-observability-evidence.ps1
```

Wave 6 notes:
- backend evidence uses isolated Django test settings with `USE_POSTGRES_TEST_DB=0`
- frontend evidence uses a temporary user-space Node 22 toolchain staged in `/dev/shm/node22`
- the `/dev/shm` staging path is required on the current production host because the root filesystem is full and the system Node 18 runtime is too old for the current Vite/Vitest stack

## 8) Wave 8 final acceptance evidence

Run from `E:\grad_project`:

```powershell
.\tools\prod\prod-wave8-final-evidence.ps1
```

Wave 8 notes:
- the script captures a fresh production health snapshot first
- it runs only the owned Wave 8 pytest slice:
  - `tests/system/test_wave8_final_acceptance.py`
  - `tests/performance/test_wave8_live_soak_offline_validation.py`
- backend evidence uses isolated Django test settings with `USE_POSTGRES_TEST_DB=0`
- the final acceptance manifest is written by `manage.py run_maturity_acceptance --skip-tests` so the evidence package records artifact-gate evaluation separately from raw pytest output
