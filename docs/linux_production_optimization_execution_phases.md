# Linux Production Optimization Execution Phases (RTX 5090)

> **Superseded (2026-05-29):** This plan has been superseded by the
> [Heterogeneous Production Runtime Maturity Plan](heterogeneous_production_runtime_maturity_plan.md)
> which is the current top-level controlling plan. This document remains
> useful as phase history and implementation detail reference.
> **Maturity closure completed:** accepted job `b1d2311c`, SHA `af3fce3`.

This runbook defines the execution order for production optimization work across:
- dev laptop (`E:\grad_project`)
- GitHub (`009-parallel-pose-inference`)
- production server (`/home/bamby/grad_project`)

Constraints:
- Linux production host: no Docker, no sudo.
- Triton-only inference on production GPU — `TRITON_FORCE_DOCKER=False` in backend/.env (native Linux).
- Do not revert unrelated work from other contributors.
- Backend runtime entrypoints must source only `backend/.env`; repo-root `.env` must not silently diverge or participate in backend startup on production.

## Milestone Map

| Milestone | Title | Outcome |
| --- | --- | --- |
| M0 | Hygiene + Sync Baseline | Clean working trees, controlled stash state, same branch lineage |
| M1 | Config Contract | Runtime profile/cadence/reuse/queue env contract finalized |
| M2 | Celery Isolation | Per-model queue routing and worker topology active |
| M3 | Offline Pipeline | stride=1 + sparse detect + reuse/interpolation behavior implemented |
| M4 | Live Pipeline | live cadence/reuse behavior with timeout-safe degrade path |
| M5 | Triton Single-Endpoint Profiles | one active endpoint at a time + profile-specific model config binding |
| M6 | TensorRT Dynamic Batch | regenerated dynamic-batch engines active in Triton |
| M7 | Benchmark Artifacts | profile matrix + JSON/CSV/MD ranking outputs |
| M8 | Sign-off + Sync Closure | acceptance gates evidence + stash closure + hash sync |

## M0: Hygiene + Sync Baseline

### Phase 0.1 Tasks (mandatory)
1. Record repo state on dev:
   - `git status --short`
   - `git stash list`
   - `git branch --show-current`
2. Record repo state on production:
   - `ssh -p 27681 bamby@0.tcp.eu.ngrok.io "cd /home/bamby/grad_project && git status --short && git stash list && git branch --show-current"`
3. Detect CRLF-only churn before staging:
   - `git diff --ignore-cr-at-eol --stat`
4. Move non-product temporary scripts to untracked `tmp/` if needed.
5. Confirm no accidental revert of others' tracked changes.

### Phase 0.1 Tests
```powershell
git status --short
git stash list
git diff --ignore-cr-at-eol --stat
git diff --cached --ignore-cr-at-eol --stat
```

### Phase 0.1 Commit Stage
- No feature commit required.
- Optional:
  - `chore(repo): add/adjust ignores for transient local runtime artifacts`

### Phase 0.2 Tasks (mandatory)
1. Ensure branch is `009-parallel-pose-inference` on dev and prod.
2. Push dev branch head to GitHub.
3. Pull on prod with non-destructive strategy:
   - `git pull --rebase --autostash origin 009-parallel-pose-inference`
4. Resolve only true conflicts; avoid broad file rewrites.
5. Verify `HEAD` hash parity across dev/prod/GitHub.

### Phase 0.2 Tests
```powershell
git rev-parse HEAD
git rev-parse origin/009-parallel-pose-inference
git log --oneline -n 3
```

### Phase 0.2 Commit Stage
- If conflict resolution modifies tracked files:
  - `chore(sync): resolve production rebase conflicts and align branch head`

## M1: Config Contract

### Phase 1.1 Tasks (mandatory)
1. Enforce `TRITON_EXECUTION_PROFILE` enum:
   - `throughput_guardrails`
   - `live_latency_first`
2. Apply default fallback: `throughput_guardrails`.
3. Update `.env.example` and `backend/.env.example` with approved defaults.
4. Document settings in quickstart + backend README.

### Phase 1.1 Tests
```powershell
.\.venv\Scripts\python.exe -m pytest backend/tests/unit -k "triton and profile" -n auto --dist=loadscope -q --tb=short
```

### Phase 1.1 Commit Stage
- `feat(config): enforce triton execution profile contract and defaults`

### Phase 1.2 Tasks (mandatory)
1. Add/validate cadence variables:
   - `OFFLINE_DETECT_EVERY_N_FRAMES=2`
   - `LIVE_DETECT_EVERY_N_FRAMES=3`
2. Add/validate reuse TTL variables:
   - `OFFLINE_REUSE_LAST_BOXES_TTL_FRAMES=20`
   - `LIVE_REUSE_LAST_BOXES_TTL_FRAMES=8`
3. Add bounded parser validation in settings.
4. Ensure values are surfaced in runtime diagnostics.

### Phase 1.2 Tests
```powershell
.\.venv\Scripts\python.exe -m pytest backend/tests/unit -k "detect_every_n_frames or reuse_last_boxes or settings" -n auto --dist=loadscope -q --tb=short
```

### Phase 1.2 Commit Stage
- `feat(config): add detector cadence and reuse ttl environment contract`

## M2: Celery Isolation

### Phase 2.1 Tasks (mandatory)
1. Define per-model queues:
   - `pipeline.live.person_detector.worker`
   - `pipeline.live.rtmpose_model.worker`
   - `pipeline.live.behavior.worker`
   - `pipeline.offline.person_detector.worker`
   - `pipeline.offline.rtmpose_model.worker`
   - `pipeline.offline.behavior.worker`
2. Wire task routes for live/offline dispatch.
3. Keep:

Phase 2.1 implementation status update:
- runtime queue authority now exposes a canonical route map and `.dlq` naming contract
- Celery worker bootstrap loads `backend/.env`, matching Django/ASGI runtime bootstrap
- offline upload queue selection is resolved from runtime queue authority instead of legacy hardcoded queue names
   - `worker_prefetch_multiplier=1`
   - `acks_late=true`
4. Add queue names to env examples and documentation.

### Phase 2.1 Tests
```powershell
.\.venv\Scripts\python.exe -m pytest backend/tests/unit -k "celery and queue" -n auto --dist=loadscope -q --tb=short
.\.venv\Scripts\python.exe -m pytest backend/tests/contract -k "queue or routing" -n auto --dist=loadscope -q --tb=short
```

### Phase 2.1 Commit Stage
- `feat(celery): add per-model live and offline queue routing`

### Phase 2.2 Tasks (mandatory)
1. Add Linux user-space worker launch scripts with explicit queue and concurrency.
2. Validate active queue bindings with `celery inspect`.
3. Add restart-safe runbook commands.

### Phase 2.2 Tests
```bash
celery -A config inspect active_queues
celery -A config inspect stats
```

### Phase 2.2 Commit Stage
- `feat(ops): add production worker launch scripts for per-model isolation`

## M3: Offline Pipeline Optimization

### Phase 3.1 Tasks (mandatory)
1. Keep `TRITON_OFFLINE_FRAME_STRIDE=1`.
2. Run detector every `OFFLINE_DETECT_EVERY_N_FRAMES`.
3. Reuse last good boxes for bounded TTL.
4. Mark source for each frame box: `detected|reused|interpolated`.

### Phase 3.1 Tests
```powershell
.\.venv\Scripts\python.exe -m pytest backend/tests/unit -k "offline and cadence and reuse" -n auto --dist=loadscope -q --tb=short
.\.venv\Scripts\python.exe -m pytest backend/tests/integration -k "offline and full_frame" -n auto --dist=loadscope -q --tb=short
```

### Phase 3.1 Commit Stage
- `feat(offline): sparse detector cadence with bounded last-good reuse`

### Phase 3.2 Tasks (mandatory)
1. Add interpolation for intermediate frames.
2. Clamp interpolated boxes to frame bounds.
3. Disable interpolation when continuity confidence breaks.

### Phase 3.2 Tests
```powershell
.\.venv\Scripts\python.exe -m pytest backend/tests/unit -k "interpolation and bbox bounds" -n auto --dist=loadscope -q --tb=short
```

### Phase 3.2 Commit Stage
- `feat(offline): add deterministic bbox interpolation for intermediate frames`

## M4: Live Pipeline Optimization

### Phase 4.1 Tasks (mandatory)
1. Apply `LIVE_DETECT_EVERY_N_FRAMES` cadence.
2. Keep overlay render at stride=1.
3. Prioritize latency and non-blocking display.

### Phase 4.1 Tests
```powershell
.\.venv\Scripts\python.exe -m pytest backend/tests/unit -k "live and cadence" -n auto --dist=loadscope -q --tb=short
.\.venv\Scripts\python.exe -m pytest backend/tests/integration -k "live stream" -n auto --dist=loadscope -q --tb=short
```

### Phase 4.1 Commit Stage
- `feat(live): implement live sparse-detect cadence with steady per-frame overlay`

### Phase 4.2 Tasks (mandatory)
1. Reuse last good live boxes during timeout for TTL window.
2. Emit timeout/reuse counters.
3. Ensure no crash path on detector timeout.

### Phase 4.2 Tests
```powershell
.\.venv\Scripts\python.exe -m pytest backend/tests/integration -k "timeout and live and reuse" -n auto --dist=loadscope -q --tb=short
.\.venv\Scripts\python.exe -m pytest backend/tests/resilience -k "triton_timeout" -n auto --dist=loadscope -q --tb=short
```

### Phase 4.2 Commit Stage
- `feat(live): timeout-safe box reuse with runtime telemetry counters`

### Phase 4 / Wave 2 ingestion evidence notes
- Queue/drop/reconnect validation is emitted through `tools/prod/prod-wave2-ingestion-evidence.ps1`.
- The production evidence wrapper uses `backend/.env` as the single runtime env authority, then enforces `DJANGO_SETTINGS_MODULE=config.settings.acceptance` and `USE_POSTGRES_TEST_DB=1` for the PostgreSQL-backed Wave 2 pytest slice.
- This isolation is required because the production PostgreSQL role is not guaranteed to have test-database create/drop privileges, while the Wave 2 integration contract only requires deterministic application-level persistence and API validation.

## M5: Triton Single-Endpoint Mode and Profile Binding

### Phase 5.1 Tasks (mandatory)
1. Enforce single active Triton endpoint policy (never run live+offline endpoints together).
2. Allowed endpoint profiles:
   - Live mode: `39000/39001/39002`
   - Offline mode: `39100/39101/39102`

## M7 / Wave 6 observability execution notes

- Wave 6 evidence is captured with `tools/prod/prod-wave6-observability-evidence.ps1`.
- The backend Wave 6 pytest slice runs with `DJANGO_SETTINGS_MODULE=config.settings.acceptance`, `USE_POSTGRES_TEST_DB=1`, and a known-good `FIELD_ENCRYPTION_KEY`.
- The frontend Wave 6 test path uses a temporary Node 22 binary in `/dev/shm/node22` because the production host currently exposes only system Node 18 and the root filesystem is full, which blocks persistent user-space installs under `/home/bamby`.
- `frontend/vite.config.ts` must stay ESM-safe and must not rely on `__dirname`; Wave 6 prod validation uses Vitest `--configLoader runner` specifically to avoid bundled-config writes onto the full root filesystem.
3. Declare runtime mode explicitly:
   - `TRITON_EXECUTION_MODE=live` for live profile
   - `TRITON_EXECUTION_MODE=offline` for offline profile
4. Set all application URLs to the currently active endpoint:
   - `TRITON_URL=http://127.0.0.1:<active_http_port>`
   - `TRITON_LIVE_URL=http://127.0.0.1:<active_http_port>`
   - `TRITON_OFFLINE_URL=http://127.0.0.1:<active_http_port>`
5. Add preflight checks for ports, model repository, readiness, and inactive-port rejection.
6. Distinguish backend subsystem states precisely:
   - `healthy`: dependency reachable and authenticated
   - `degraded`: dependency reachable but runtime contract incomplete (for example Redis auth challenge)
   - `unhealthy`: dependency unreachable or failing hard

### Phase 5.1 Tests
```bash
# Example: offline mode active
curl -fsS http://127.0.0.1:39100/v2/health/ready
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:39000/v2/health/ready || true
ss -ltn | awk 'NR==1 || /:39000|:39100|:8000|:8011|:5173|:5174/'
```

### Phase 5.1 Commit Stage
- `feat(triton): enforce single-endpoint triton mode with preflight checks`

### Phase 5.2 Tasks (mandatory)
1. Bind active endpoint to the selected profile config:
   - `configs/live.pbtxt` when live mode is active.
   - `configs/offline.pbtxt` when offline mode is active.
2. Verify loaded model configs on the active endpoint.
3. Ensure inactive endpoint is stopped and no Celery runtime on the node routes inference work to it.

### Phase 5.2 Tests
```bash
# Example: offline mode active
curl -s http://127.0.0.1:39100/v2/models/person_detector/config
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:39000/v2/models/person_detector/config || true
```

### Phase 5.2 Commit Stage
- `feat(triton): enforce per-profile model-config binding for single active endpoint`

## M6: TensorRT Dynamic Batch Activation

### Phase 6.1 Tasks (mandatory)
1. Rebuild TensorRT engines from ONNX with dynamic batch profiles.
2. Version outputs per model family.
3. Preserve previous engines for rollback.

### Phase 6.1 Tests
```bash
python backend/scripts/build_tensorrt_engines.py --workspace-mib 4096
```

### Phase 6.1 Commit Stage
- `build(trt): regenerate dynamic-batch TensorRT engines from ONNX artifacts`

### Phase 6.2 Tasks (mandatory)
1. Update Triton model configs:
   - `max_batch_size > 1`
   - profile-specific preferred batch sizes
   - tuned `instance_group`
2. Restart Triton and validate loaded configs.

### Phase 6.2 Tests
```bash
curl -s http://127.0.0.1:39100/v2/models/rtmpose_model/config
curl -s http://127.0.0.1:39100/v2/models/person_detector/config
```

### Phase 6.2 Commit Stage
- `feat(triton): activate dynamic batching and tuned instance groups`

## M7: Benchmark and Evidence Artifacts

### Phase 7.1 Tasks (mandatory)
1. Ensure benchmark outputs include:
   - `profile_run_manifest.json`
   - `model_metrics.csv`
   - `job_summary.csv`
   - `comparison_summary.md`
   - `tuning_matrix_results.json`
2. Include per-model latency/throughput/timeout/error and pose quality KPIs.

### Phase 7.1 Tests
```powershell
.\.venv\Scripts\python.exe backend/scripts/benchmark_export_csv.py --help
.\.venv\Scripts\python.exe -m pytest backend/tests/unit -k "benchmark_export_csv" -n auto --dist=loadscope -q --tb=short
```

### Phase 7.1 Commit Stage
- `feat(benchmark): add mandatory profile artifacts and KPI export coverage`

### Phase 7.2 Tasks (mandatory)
1. Run profile matrix for:
   - baseline
   - throughput_guardrails
   - live_latency_first
2. Use same video set and comparable detections.
3. Output ranked score + pass/fail gates.

### Phase 7.2 Tests
```bash
python scripts/pose_eval/compare_models.py --help
python scripts/pose_eval/run_profile_matrix.py --help
```

### Phase 7.2 Commit Stage
- `feat(benchmark): add profile matrix runner and ranked gate summary`

## M8: Final Acceptance + Sync/Stash Closure

### Phase 8.1 Tasks (mandatory)
1. Execute offline tests on production videos (including synced `Video Project 1.mp4`).
2. Execute live soak test with telemetry capture.
3. Validate acceptance gates:
   - Live p99 latency
   - Timeout rate
   - Offline throughput gain
   - Overlay completeness
   - Temporal jitter
   - Error rate

### Phase 8.1 Tests
```powershell
.\.venv\Scripts\python.exe -m pytest backend/tests/system -n auto --dist=loadscope -q --tb=short
.\.venv\Scripts\python.exe -m pytest backend/tests/performance -n auto --dist=loadscope -q --tb=short
```

Wave 8 orchestration/evidence slice status:
- `backend/apps/video_analysis/management/commands/run_maturity_acceptance.py` evaluates Wave 1-7 evidence presence, Wave 8 artifact presence, representative dataset coverage, profile-matrix reproducibility, performance-report pass state, and paper traceability before writing `ci_evidence/production/wave8/final_acceptance_manifest.json`
- `tools/prod/prod-wave8-final-evidence.ps1` captures a health snapshot and invokes `run_maturity_acceptance` without skipped test execution; skipped test results cannot satisfy final production acceptance
- this slice does not implement the earlier Wave 8 artifact generators (`T109-T118`); if those artifacts are absent or structurally invalid, the final acceptance command fails closed

### Phase 8.1 Commit Stage
- `test(perf): record acceptance-gate evidence for production profiles`

### Phase 8.2 Tasks (mandatory)
1. Verify hash sync:
   - dev == GitHub == prod
2. Close stash state safely:
   - inspect each stash as patch
   - archive useful patch under `backend/artifacts/stash_archive/`
   - drop stash after archival or explicit rejection
3. Confirm clean status in all environments.

### Phase 8.2 Tests
```powershell
git rev-parse HEAD
git rev-parse origin/009-parallel-pose-inference
git status --short
git stash list
```

### Phase 8.2 Commit Stage
- `chore(release): finalize cross-environment sync and close stash backlog`

## Sync and Stash Closure Runbook (dev/prod/GitHub)

1. Dev push:
```powershell
git add -A
git commit -m "..." 
git push origin 009-parallel-pose-inference
```
2. Prod pull:
```bash
cd /home/bamby/grad_project
git fetch --all --prune
git pull --rebase --autostash origin 009-parallel-pose-inference
```
3. Stash inspect:
```bash
git stash list
git stash show -p stash@{0} > /tmp/stash0.patch
```
4. Archive or drop:
```bash
mkdir -p backend/artifacts/stash_archive
cp /tmp/stash0.patch backend/artifacts/stash_archive/$(date +%Y%m%d_%H%M%S)_stash0.patch
git stash drop stash@{0}
```
5. Final parity check:
```bash
git rev-parse HEAD
git status --short
git stash list
```

## Mandatory Phase Completion Checklist

A phase is complete only when all are true:
1. Phase tasks are executed.
2. Phase tests pass.
3. Commit is created with phase-scoped message.
4. Commit is pushed to GitHub.
5. Production is synced to same commit hash.
6. Evidence paths are recorded in phase notes.
