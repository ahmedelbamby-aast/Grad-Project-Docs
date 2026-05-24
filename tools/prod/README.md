# Production Queue Workers (No Docker / No sudo)

This folder contains production-native scripts to launch dedicated Celery workers per queue and avoid queue contention between live/offline control paths and model-specific dispatch paths.

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

