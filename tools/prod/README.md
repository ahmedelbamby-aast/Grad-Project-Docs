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

This starts detached workers for the active `TRITON_EXECUTION_MODE` in
`backend/.env`. Defaults:

- always: `default` -> `2`
- live mode: `pipeline.live.control`, `pipeline.live.person_detector.worker`,
  `pipeline.live.rtmpose_model.worker`, `pipeline.live.behavior.worker` -> `2` each
- offline mode: `video_inference`, `pipeline.offline.person_detector.worker`,
  `pipeline.offline.rtmpose_model.worker`, `pipeline.offline.behavior.worker` -> `4` each

The launcher reads worker time limits, concurrency, and guardrails from `backend/.env`
(`CELERY_TASK_SOFT_TIME_LIMIT`, `CELERY_TASK_TIME_LIMIT`,
`CELERY_MAX_TASKS_PER_CHILD`, `CELERY_GPU_CONCURRENCY_CAP`,
`CELERY_NOFILE_LIMIT`, `CELERY_DEFAULT_WORKER_CONCURRENCY`,
`CELERY_LIVE_WORKER_CONCURRENCY`, `CELERY_OFFLINE_WORKER_CONCURRENCY`) with
shell environment variables taking precedence.
Run `prod_stop_celery_workers.sh` before every restart to remove stale children.

Wave 2 queue governance notes:
- active/inactive control routing is resolved from `apps.pipeline.queue_routes`
- dead-letter names follow the `.dlq` suffix contract
- offline upload dispatch resolves its primary queue from runtime queue authority instead of legacy hardcoded queue names
- `.\tools\prod\prod-wave2-ingestion-evidence.ps1` captures the Wave 2 queue/drop/reconnect evidence bundle from Windows
- the Wave 2 prod pytest wrapper runs with `DJANGO_SETTINGS_MODULE=config.settings.acceptance` and PostgreSQL only
- pytest-backed production evidence requires a dedicated PostgreSQL test role with `CREATEDB`; the runtime application role is not an accepted workaround

Wave 5 behavior evidence:
- run `.\tools\prod\prod-wave5-behavior-evidence.ps1` from Windows to generate
  `ci_evidence/production/wave5/<timestamp>/` artifacts on production.

## 2.1) Canonical local-video operational ingestion

Run a production-local video through the same authenticated upload admission and
Celery dispatch path used by the frontend:

```powershell
.\tools\prod\prod-runtime-ingest-video.ps1 `
  -VideoPath "/home/bamby/grad_project/Raw Data/<video>.mp4" `
  -ActorUsername "<teacher-or-admin-user>" `
  -RuntimeProfile offline `
  -PipelineMode crop_frame `
  -ReplayKey "acceptance:<video>:v1" `
  -Wait `
  -OutputPath "/home/bamby/grad_project/ci_evidence/production/runtime/runtime_ingest_summary.json"
```

The command stores the video by invoking `VideoAnalysisJobListCreateView` and
lets `_enqueue_process_video_upload` choose the governed Celery queue. It does
not call the inference task inline. Use an existing teacher/admin account and
the active `offline` runtime profile.
Supported pipeline modes are `legacy_crop`, `full_frame`, and `crop_frame`.

For one-shot benchmark runs with GPU telemetry:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_run_benchmark.sh --pipeline-mode crop_frame --timeout 14400
```

## 2.2) Optimized parallel-flow benchmark helpers

For the production parallelization run, use the chained helper. It cancels active
jobs, purges stale video-queue messages, enables the optimized `backend/.env` defaults, restarts Triton/workers,
probes the flow, runs the benchmark, and probes the resulting job:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_run_parallel_flow_benchmark.sh --profile per-frame-signals
```

Default target:

- video: `/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4`
- pipeline mode: `crop_frame`
- profile: `per-frame-signals` (`stride=1`; person-box reuse allowed; no reused behaviour/gaze/embedding predictions; true Triton batches for crop fanout; YOLO decode candidate cap before NMS)
- timeout: `7200`

Individual helper scripts:

```bash
bash tools/prod/prod_cancel_video_jobs.sh --all-active
bash tools/prod/prod_enable_parallel_flow.sh --profile per-frame-signals
bash tools/prod/prod_parallel_flow_probe.sh --watch 30
bash tools/prod/prod_verify_per_frame_signals.sh --job-id <job_id> --watch 30
```

For DB-backed progress on the subjective `all_merged.mp4` run:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_check_subjective_progress.sh --watch 30
```

Cycle 9b B.4 batch-window benchmark helper:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_run_batch_window_benchmark.sh \
  --max-frames 4 \
  --rss-limit-mib 4096 \
  --tag cycle9b-b4-maxframes4-$(date -u +%Y%m%dT%H%M%SZ) \
  --timeout 7200
```

The helper keeps the accepted 320 Top-K behavior route, overrides only
`TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES`, samples Celery worker RSS, writes
metrics/model-agreement evidence under `backend/logs/<tag>/`, and restores the
accepted `max_frames=2` profile at exit.

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
- backend evidence uses PostgreSQL-only acceptance settings through `config.settings.acceptance`
- configure a dedicated PostgreSQL test role with `CREATEDB` before running pytest-backed production evidence
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
- backend evidence uses PostgreSQL-only acceptance settings through `config.settings.acceptance`
- pytest-backed acceptance requires the dedicated PostgreSQL test role with `CREATEDB`; SQLite fallback is prohibited
- the final acceptance manifest is written by `manage.py run_maturity_acceptance` and cannot pass if its owned system and performance slices are skipped
- the self-hosted `ci-runtime-gpu.yml` workflow requires repository secrets `RUNTIME_CI_POSTGRES_DB`, `RUNTIME_CI_POSTGRES_USER`, `RUNTIME_CI_POSTGRES_PASSWORD`, `RUNTIME_CI_POSTGRES_HOST`, and `RUNTIME_CI_POSTGRES_PORT`; the configured role must have `CREATEDB`
