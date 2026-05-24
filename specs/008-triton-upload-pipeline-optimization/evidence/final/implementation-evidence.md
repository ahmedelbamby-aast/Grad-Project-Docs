# Implementation Evidence: Triton Upload Pipeline Optimization

## Metadata

- Feature: `008-triton-upload-pipeline-optimization`
- Branch: `unknown (dirty worktree)`
- Commit SHA: `uncommitted`
- Date (UTC): `2026-05-21`
- Executor: `Codex multi-agent rollout`

## Environment Snapshot

- GPU model/count: `NVIDIA GeForce RTX 3050 Ti Laptop GPU (1)`
- CUDA/Triton versions: `not captured in this evidence pack`
- Host CPU/RAM: `not captured in this evidence pack`
- Effective env tuning values used: `captured below under Env Defaults`

## Baseline Evidence (Before Changes)

### Commands Run

```text
docker compose -f docker-compose.dev.yml up -d postgres redis
docker compose -f docker-compose.dev.yml ps postgres redis
python load-tests/staged_validation.py --seed 42
nvidia-smi --query-gpu=name,utilization.gpu,utilization.memory,memory.total,memory.used,temperature.gpu,power.draw --format=csv,noheader,nounits
.\backend\.venv\Scripts\python.exe -m celery -A config inspect stats
docker exec grad_project-redis-1 redis-cli LLEN default
docker exec grad_project-redis-1 redis-cli LLEN video_inference
docker exec grad_project-redis-1 redis-cli INFO clients
```

### Metrics Collected

- Upload throughput (jobs/min): `before-profile low/medium/high = 70.33 / 149.67 / 259.33 (simulated harness)`
- End-to-end latency P50/P95/P99: `before-profile upload P95 low/medium/high = 679.86ms / 770.10ms / 1314.80ms`
- Queue wait P50/P95: `before-profile non-upload queue P95 low/medium/high = 71.62ms / 237.28ms / 738.40ms`
- Triton request latency P50/P95: `not captured from live infra in this file`
- GPU utilization avg/p95: `point snapshot at baseline capture time: gpu=0%, mem=0%`
- Error rate/timeouts/retries: `not captured from live infra in this file`

### Logs/Artifacts

```text
postgres: Up (healthy), port 5432 mapped
redis: Up (healthy), port 6379 mapped
nvidia-smi:
NVIDIA GeForce RTX 3050 Ti Laptop GPU, 0, 0, 4096, 398, 54, 12.53

celery inspect:
Error: No nodes replied within time constraint

redis queue depth:
default=0
video_inference=0
connected_clients=1
```

## Post-Implementation Evidence (After Changes)

### Commands Run

```text
E:\grad_project\backend\.venv\Scripts\python.exe -m pytest tests/unit/video_analysis/test_external_factor_events.py tests/unit/pipeline/test_rtmpose_model_identity.py tests/unit/pipeline/test_triton_client_validation.py -n auto --dist=loadscope -q --tb=short
E:\grad_project\backend\.venv\Scripts\python.exe -m pytest tests/unit/video_analysis/test_triton_offline_batch_queue.py -q --tb=short
E:\grad_project\backend\.venv\Scripts\python.exe -m pytest tests/unit/video_analysis tests/unit/pipeline/test_rtmpose_model_identity.py tests/unit/pipeline/test_triton_client_validation.py -n auto --dist=loadscope -q --tb=short
E:\grad_project\backend\.venv\Scripts\python.exe -m pytest tests/contract/test_triton_inference_contract.py tests/integration/test_triton_unavailable_handling.py tests/integration/test_full_inference_flow.py -n auto --dist=loadscope -q --tb=short
```

### Metrics Collected

- Upload throughput (jobs/min): `simulation only; see staged-load-validation.md`
- End-to-end latency P50/P95/P99: `simulation only; P95 deltas captured`
- Queue wait P50/P95: `simulation only; deltas captured`
- Triton request latency P50/P95: `not captured separately`
- GPU utilization avg/p95: `not captured`
- Error rate/timeouts/retries: `0.0 -> 0.0 in simulated harness`

### Logs/Artifacts

```text
25 passed in 280.71s (0:04:40)
2 passed in 265.04s (0:04:25)
94 passed in 29.19s
29 passed in 26.92s
```

## Test Output Summary

- Targeted unit/contract/integration path for upload+Triton behavior: `PASS` (see counts above).
- Staged load validation command:

```text
python load-tests/staged_validation.py
wrote ci_evidence\spec008\staged-load-results.json
wrote specs\008-triton-upload-pipeline-optimization\evidence\final\staged-load-validation.md
```

- Staged-load summary (simulated harness, infra-independent):
  - Low: upload P95 679.86ms -> 413.56ms, non-upload queue P95 71.62ms -> 0.00ms
  - Medium: upload P95 770.10ms -> 420.62ms, non-upload queue P95 237.28ms -> 40.72ms
  - High: upload P95 1314.80ms -> 513.98ms, non-upload queue P95 738.40ms -> 39.22ms
  - Starvation verdict (non-upload queue): FALSE in all staged after-profile runs

## Before vs After Delta Summary

- Throughput change: `No throughput loss in simulation; same completed upload jobs/min per stage with lower latency under after profile`
- Latency change (P95): `Upload P95 reduced by ~39.2% (low), ~45.4% (medium), ~60.9% (high)`
- GPU utilization change: `not captured`
- Error-rate change: `0.0 -> 0.0 in simulated harness`

## Regression Check

- Upload correctness parity status: `PASS for targeted upload/inference/contract suites listed above`
- Non-upload queue impact status: `No starvation detected in staged after-profile runs; non-upload queue wait P95 improved across low/medium/high in simulated harness`
- Known issues / residual risks: `Real infra-backed load evidence (Triton/GPU/Celery/Redis) is still required before final non-sim performance sign-off`

## Rollback Notes

- Rollback toggles used/tested: `prepared snippet below; runtime verification output not attached in this file`
- Outcome: `not executed as part of this evidence file`

## Env Defaults

```text
CELERY_DEFAULT_QUEUE=default
CELERY_VIDEO_INFERENCE_QUEUE=video_inference
CELERY_WORKER_POOL=prefork
CELERY_WORKER_CONCURRENCY=4
CELERY_WORKER_PREFETCH_MULTIPLIER=1
CELERY_TASK_ACKS_LATE=true
CELERY_TASK_ACKS_ON_FAILURE_OR_TIMEOUT=true
CELERY_TASK_REJECT_ON_WORKER_LOST=true
TRITON_RETRY_ATTEMPTS=2
TRITON_RETRY_TIMEOUT_SCALE=2.0
TRITON_OFFLINE_BATCH_QUEUE_ENABLED=1
TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=8
TRITON_OFFLINE_BATCH_QUEUE_MAX_WAIT_MS=40
TRITON_OFFLINE_BATCH_QUEUE_MAX_PENDING=256
TRITON_OFFLINE_BATCH_QUEUE_MAX_CONCURRENCY=8
TRITON_SHADOW_PRECHECK_MAX_CONCURRENCY=2
TRITON_MAX_INFLIGHT_REQUESTS=16
TRITON_DYNAMIC_BATCHING_ENABLED=1
TRITON_UPLOAD_TARGET_BATCH_SIZE=8
TRITON_UPLOAD_MAX_BATCH_WAIT_MS=40
TRITON_UPLOAD_MAX_PENDING_FRAMES=512
```

## Rollback Snippet

```powershell
$env:TRITON_OFFLINE_BATCH_QUEUE_ENABLED = "0"
$env:TRITON_SHADOW_PRECHECK_MAX_CONCURRENCY = "1"
$env:TRITON_MAX_INFLIGHT_REQUESTS = "4"
$env:CELERY_WORKER_CONCURRENCY = "2"
$env:CELERY_WORKER_PREFETCH_MULTIPLIER = "1"
```

## Production Defaults by Hardware Tier

### Single GPU

```text
CELERY_WORKER_CONCURRENCY=2
CELERY_WORKER_PREFETCH_MULTIPLIER=1
TRITON_OFFLINE_BATCH_QUEUE_ENABLED=1
TRITON_UPLOAD_TARGET_BATCH_SIZE=8
TRITON_UPLOAD_MAX_BATCH_WAIT_MS=40
TRITON_OFFLINE_BATCH_QUEUE_MAX_PENDING=256
TRITON_MAX_INFLIGHT_REQUESTS=8
TRITON_SHADOW_PRECHECK_MAX_CONCURRENCY=1
```

### Multi-GPU

```text
CELERY_WORKER_CONCURRENCY=4
CELERY_WORKER_PREFETCH_MULTIPLIER=1
TRITON_OFFLINE_BATCH_QUEUE_ENABLED=1
TRITON_UPLOAD_TARGET_BATCH_SIZE=8
TRITON_UPLOAD_MAX_BATCH_WAIT_MS=40
TRITON_OFFLINE_BATCH_QUEUE_MAX_PENDING=512
TRITON_MAX_INFLIGHT_REQUESTS=16
TRITON_SHADOW_PRECHECK_MAX_CONCURRENCY=2
```

## Post-Deploy Watch Window Checklist

- Watch window duration: first `30 minutes` and again at `24 hours` after deploy.
- Track `upload latency P95`, `upload throughput jobs/min`, `non-upload queue wait P95`, `timeout/retry/failure counts`, `GPU utilization/memory`.
- Roll back to safe profile if upload P95 regresses by `>25%` vs pre-deploy baseline for 10+ minutes, or non-upload queue P95 exceeds `800ms`.
- Re-run `python load-tests/staged_validation.py --seed 42` and attach refreshed artifacts under `ci_evidence/spec008/`.

## Sign-off

- Main agent verification: `documented as partial; simulation-backed load + targeted suite pass`
- Reviewer: `pending`
