# Agents Execution Guide

## Purpose
This file defines how agents should execute tests quickly and safely in this repository.

## Database Authority
- PostgreSQL is the only authoritative relational database for this repository.
- SQLite is not an accepted runtime, test, migration, benchmark, or acceptance backend.
- Agents must not switch tests or scripts to SQLite for convenience, speed, or local fallback.
- Any change touching persistence, migrations, ORM queries, constraints, transactions, or evidence storage must be validated against PostgreSQL semantics.

### Production PostgreSQL Test-Role Prerequisite
- Full backend pytest suites require a PostgreSQL role that can create test databases (`CREATEDB`).
- Current production app role (`exam_user`) may not have enough privileges to create roles or databases.
- If tests fail with `permission denied to create role` or `permission denied to create database`, agents must not attempt SQLite fallback.
- Required DBA/bootstrap action (performed by a privileged PostgreSQL role):
  - Create a dedicated CI/test role (example: `exam_ci_user`) with `LOGIN` + `CREATEDB`.
  - Grant ownership/privileges for runtime DB access as needed.
  - Update environment keys consistently (local/prod CI shells) to use the same PostgreSQL host/port and test role for pytest.

## CI Credential Handling
- Authenticated GitHub CI inspection uses `GITHUB_TOKEN` loaded from local ignored file `.local/secrets/github.env`.
- The credential file MUST remain untracked and MUST contain a rotated token, never a token exposed in chat, logs, evidence, or Markdown.
- Agents MUST NOT write raw CI tokens into tracked files, command transcripts, or production evidence packages.

<!-- SPECKIT START -->
## Active Spec Kit Plan
- Active feature plan: [specs/011-bsil-semantic-runtime/plan.md](specs/011-bsil-semantic-runtime/plan.md)
<!-- SPECKIT END -->

## Current Active Production Runtime Plan
- **Active plan name:** `Heterogeneous Production Runtime Maturity Plan` — **COMPLETED 2026-05-29**
- Plan file: [docs/heterogeneous_production_runtime_maturity_plan.md](docs/heterogeneous_production_runtime_maturity_plan.md)
- Tasks file: [docs/heterogeneous_production_runtime_maturity/tasks.md](docs/heterogeneous_production_runtime_maturity/tasks.md)
- Evidence root: [ci_evidence/production/runtime_maturity/](ci_evidence/production/runtime_maturity/)
- Accepted job: `b1d2311c-b0af-44a4-a551-61e58200eb11` | Final SHA: `af3fce3`

## Runtime Stability Remediation (Completed)
- **Plan:** [docs/runtime_stability_remediation/plan.md](docs/runtime_stability_remediation/plan.md)
- **Tasks:** [docs/runtime_stability_remediation/tasks.md](docs/runtime_stability_remediation/tasks.md)
- **Constitution:** Section 17 (§17.1–§17.4) — Runtime Job Lifecycle and Vector Integrity
- **Status:** All RS tasks completed and merged to master (SHA `375cc84`)
- Local workflow: [.github/workflows/prod-runtime-maturity.yml](.github/workflows/prod-runtime-maturity.yml)
- Production evidence helper: [tools/prod/prod-runtime-maturity-evidence.ps1](tools/prod/prod-runtime-maturity-evidence.ps1)
- **Status:** Active, not completed.
- **Branch target:** `release/prod-runtime-stabilization`
- This is the current controlling plan for production runtime work. Agents must treat it as the source of truth when it conflicts with older optimization notes.
- **Production policy update:** Development and production are intentionally heterogeneous. Local Windows/Docker/RTX 3050 Ti may validate code behavior and contracts, but only native Linux/no-Docker/RTX 5090 production validation can certify inference execution, GPU evidence, throughput claims, and final lifecycle acceptance.
- **Triton policy:** Production must use `INFERENCE_STRATEGY=triton_only`. Triton uses **dual configured endpoint profiles** (live and offline), but production runs **one active mode at a time** selected by `backend/.env` (`TRITON_EXECUTION_MODE=live` or `TRITON_EXECUTION_MODE=offline`). The inactive endpoint profile must not receive production inference traffic, scheduler requests, Celery routing, or production-ready health status.

## Superseded Optimization Context
- Previous plan name: `Linux Production Optimization Execution Phases (RTX 5090)`
- Previous plan file: [docs/linux_production_optimization_execution_phases.md](docs/linux_production_optimization_execution_phases.md)
- The previous plan remains useful for phase history and implementation details, but it is no longer the current top-level plan.

## Current Phase Status (handover snapshot)
- M0: partially completed (sync/hygiene repeatedly applied; local untracked `tmp/*` still exists on dev laptop).
- M1: partially completed (profile/cadence env contract documented, continue hardening env examples).
- M2: largely completed (queue routing + worker scripts added).
- M3:
  - Phase 3.1: in progress/partially implemented (sparse detect + reuse present).
  - Phase 3.2: not started (bbox interpolation/clamp/continuity guards).
- M4: partially touched, not accepted (live cadence/reuse/telemetry still needs full validation).
- M5: COMPLETED — single active endpoint policy enforced; offline Triton (39100) active, live (39000) isolated.
- M6: not completed (dynamic-batch TensorRT regeneration and tuned Triton configs still pending full sign-off).
- M7:
  - Phase 7.1: started (benchmark exporter/test scaffolding added).
  - Phase 7.2: not started (full profile matrix execution/report ranking).
- M8: COMPLETED — final acceptance gates passed; evidence packaged; branch merged to master.

## Heterogeneous Production Runtime Maturity — COMPLETED (2026-05-29)
- **Accepted job:** `b1d2311c-b0af-44a4-a551-61e58200eb11`
- **Final git SHA:** `af3fce3` (all locations in parity)
- **Runtime:** native Linux, no Docker, no sudo, RTX 5090, Triton offline (39100)
- **Evidence root:** `ci_evidence/production/runtime_maturity/`
- **Package index:** `ci_evidence/production/runtime_maturity/final/evidence_package_index.md`
- **Acceptance summary:** `ci_evidence/production/runtime_maturity/final/production_acceptance_summary.md`
- **Key runtime fixes applied during this cycle:**
  - `TRITON_FORCE_DOCKER=False` in backend/.env — was selecting non-existent Docker authority
  - `runtime_used` label corrected from `"hybrid"` to `"triton"` for triton-only paths (commit 12c291c)
  - `selected_authority` renamed from `triton_docker` to `triton_native` for native Linux
  - Embedding vector dimension coerced at DB write boundary (commit 441c28e, f689a4d)
  - Stale-job reconciler registered in Celery beat_schedule — no more stuck non-terminal jobs
- **Constitution:** bumped to v2.4.0 (Sections 17 + 18 added)

## Production Server Details (Linux, no Docker, no sudo)
- Host: `0.tcp.eu.ngrok.io`
- SSH port: `27681`
- Username: `bamby`
- Repo path: `/home/bamby/grad_project`
- Runtime constraints:
  - no Docker
  - no sudo
  - CUDA runtime authority: **CUDA 12.8**
  - TensorRT runtime authority must be pinned and rebuilt against CUDA 12.8-compatible stack
  - Triton-only inference on NVIDIA GPU
  - dual Triton endpoint profiles configured, one active production mode at a time
  - backend runtime env authority is `backend/.env`; repo-root `.env` must not participate in backend startup
  - PostgreSQL-backed persistence only; SQLite-backed execution is invalid in prod and invalid for production evidence

## Production Toolchain Pinning (Mandatory)
- Triton production default binary MUST be pinned to:
  - `/home/bamby/services/triton_build_r2502/tritonserver/install/bin/tritonserver`
- Python command in production shells MUST resolve consistently to backend venv:
  - `/home/bamby/grad_project/backend/.venv/bin/python`
- Agents must ensure both `~/.bashrc` and `~/.profile` export stable user-path resolution for:
  - Python (`PATH` includes backend venv `bin`)
  - Triton (`TRITON_SERVER_BIN` and optional `PATH` prepend)
  - pip user cache (`PIP_CACHE_DIR=/home/bamby/.cache/pip`)
- Duplicate or conflicting shell-export lines in `~/.bashrc` / `~/.profile` are considered drift and must be cleaned.

### Required Prod Shell Exports
- `export PATH="/home/bamby/grad_project/backend/.venv/bin:$PATH"`
- `export TRITON_SERVER_BIN="/home/bamby/services/triton_build_r2502/tritonserver/install/bin/tritonserver"`
- `export PIP_CACHE_DIR="/home/bamby/.cache/pip"`
- Optional convenience alias:
  - `alias python="/home/bamby/grad_project/backend/.venv/bin/python"`

### TensorRT Engine Compatibility Rule
- If runtime error contains serialization/version mismatch (for example engine version tag mismatch), treat all affected `.engine`/`model.plan` artifacts as incompatible.
- Rebuild required engines with the active production TensorRT stack, then refresh Triton model repository artifacts before reloading Triton.
- Rebuild success is not enough unless Triton model load status is `READY` for required models.
- Current commonly used app ports:
  - Backend: `127.0.0.1:8011`
  - Frontend: `127.0.0.1:5174`
  - Triton (offline profile mode): `39100/39101/39102`
  - Triton (live profile mode, only when selected): `39000/39001/39002`
  - App Redis/Celery broker: `6380` (user-space Redis)
  - System Redis: `6379` may require auth and must not be assumed usable for app workers
  - PostgreSQL may run non-default in this environment (check with `ss -ltnp` and `.env`)

## Windows Connection + Tooling Workflow
- Preferred client tools on Windows:
  - OpenSSH `ssh` / `scp` (built into modern Windows)
  - PowerShell scripts in `tools/prod`
- Optional/fallback:
  - PuTTY `pscp` if OpenSSH transfer has issues

### Recommended helper scripts
- `tools/prod/prod-ssh.ps1`:
  - `.\tools\prod\prod-ssh.ps1`
  - `.\tools\prod\prod-ssh.ps1 -Cmd "hostname"`
- `tools/prod/prod-copy-to.ps1`:
  - `.\tools\prod\prod-copy-to.ps1 -LocalPath "E:\grad_project\Raw Data\Video Project 1.mp4" -RemotePath "/home/bamby/grad_project/raw_test_videos/"`
- `tools/prod/prod-copy-from.ps1`
- `tools/prod/prod-workers.ps1` (`preflight|start|status|stop`)
- `tools/prod/prod-health-snapshot.ps1`
- `tools/prod/prod-hash-parity.ps1`
- `tools/prod/prod-stash-hygiene.ps1`
- `tools/prod/prod-sync-env-keys.ps1`

## Production Startup/Validation Checklist (for next agents)
1. Confirm branch and hash parity:
   - local: `git rev-parse HEAD`
   - remote: `ssh prod-grad "cd /home/bamby/grad_project && git rev-parse HEAD"`
2. Confirm single Triton endpoint policy:
   - active endpoint health returns `200`
   - inactive endpoint returns `000`/unreachable
3. Confirm backend/model-serving health:
   - `http://127.0.0.1:8011/api/v1/health/`
   - `http://127.0.0.1:8011/api/v1/health/model-serving/`
   - interpret `redis=degraded` separately from `redis=unhealthy`; auth-challenge is a degraded state, not a process-down state
   - if runtime values disagree with `backend/.env`, treat repo-root `.env` as drift to be removed from prod startup assumptions
4. Confirm ports:
   - `ss -ltnp | egrep '39100|39000|8011|5174|6379|6380|5432|55432'`
   - confirm backend DB settings point to PostgreSQL, not SQLite
5. Run focused tests before full suites:
   - `backend/tests/unit/video_analysis/test_tasks_detection_cadence.py`
   - `backend/tests/unit/scripts/test_benchmark_export_csv.py`
   - `backend/tests/unit/pipeline/test_runtime_mode_authority.py`
   - `backend/tests/contract/test_runtime_mode_contract.py`
   - `backend/tests/system/test_wave1_runtime_policy.py`
   - scaffold integration/system tests (xfail markers are expected until converted)
6. Validate toolchain pinning:
   - `which python` resolves to backend venv python
   - `echo $TRITON_SERVER_BIN` points to pinned binary
   - `python -c "import tensorrt as trt; print(trt.__version__)"` runs in venv
7. Reload Triton and verify model readiness:
   - start selected mode only (`live` or `offline`)
   - `curl` active `/v2/health/ready` returns `200`
   - inactive mode endpoint remains unreachable

## Triton Startup Guide — Prod Linux Server (RTX 5090, No Docker, No sudo)

This section documents every issue encountered getting Triton running on
`/home/bamby/grad_project` and the permanent fixes applied. Read this before
touching Triton on prod.

### Hardware & Software Constraints

| Item | Value |
|------|-------|
| GPU | NVIDIA GeForce RTX 5090, 32 GB GDDR7 |
| CPU | 32 cores |
| CUDA | 12.8 |
| Triton binary | `/home/bamby/services/triton_build_r2502/tritonserver/install/bin/tritonserver` |
| TRT backends dir | `/home/bamby/services/triton_build_r2502/tritonserver/install/backends` |
| Model repository | `/home/bamby/grad_project/backend/models/triton_repository_cuda12` |
| Active TRT version | **10.8.0.43** (python3.11 venv) — must match `.plan` engine build |
| Serialization version | **239** (what the `.plan` engines were compiled with) |
| Postgres port | **55432** (non-standard — always check `.env`) |

---

### Issue 1 — `libnvinfer.so.10: cannot open shared object file`

**Symptom:**
```
load failed for model 'person_detector': unable to load shared library: libnvinfer.so.10
```

**Root cause:**
Non-interactive SSH sessions (`bash -l -s` or `ssh prod-grad "cmd"`) do **not**
automatically source `~/.bashrc`. The `LD_LIBRARY_PATH` set in `~/.bashrc` was
never exported into the process that launched Triton via `nohup`, so the
TensorRT backend plugin (`libtriton_tensorrt.so`) could not find `libnvinfer.so.10`.

The Triton binary RUNPATH is `${ORIGIN}/../lib` (= `install/lib`) and the
TRT backend RUNPATH is `${ORIGIN}` (= `backends/tensorrt/`). Neither location
contains `libnvinfer.so.10`. There is no system `ldconfig` entry for it either.
The library lives only in the Python venvs under `site-packages/tensorrt_libs/`.

**Fix applied (permanent):**

Both `~/.bashrc` and `~/.profile` now export the correct `LD_LIBRARY_PATH`:
```bash
export LD_LIBRARY_PATH="/home/bamby/grad_project/.venv/lib/python3.11/site-packages/tensorrt_libs:/home/bamby/services/triton/lib:${LD_LIBRARY_PATH:-}"
```

A dedicated start script `tools/prod/prod_start_triton.sh` was created that
`source ~/.bashrc` before calling the Triton binary, so the correct
`LD_LIBRARY_PATH` is always in scope regardless of how the script is invoked.

**Rule for future agents:**
Never start Triton with a bare `nohup tritonserver ...` in a non-login shell.
Always use:
```bash
bash -l /home/bamby/grad_project/tools/prod/prod_start_triton.sh
```

---

### Issue 2 — `Version tag does not match. Current Version: 240, Serialized Engine Version: 239`

**Symptom:**
```
IRuntime::deserializeCudaEngine: Error Code 1: Serialization
(Serialization assertion stdVersionRead == kSERIALIZATION_VERSION failed.
Version tag does not match. Note: Current Version: 240, Serialized Engine Version: 239)
load failed for model 'posture_model': unable to load plan file to auto complete config
```

**Root cause:**
There are **two Python venvs** with different TensorRT versions:

| Venv | Python | TRT version | Serialization tag |
|------|--------|-------------|-------------------|
| `/home/bamby/grad_project/.venv` | 3.11 | **10.8.0.43** | **239** ✅ |
| `/home/bamby/grad_project/backend/.venv` | 3.12 | **10.16.1.11** | **240** ❌ |

The `.plan` engine files in `triton_repository_cuda12` were compiled with
TRT 10.8.0.43 (tag 239). Using the backend venv's `libnvinfer.so.10` (tag 240)
causes a deserialization failure on every model.

**Fix applied (permanent):**
`LD_LIBRARY_PATH` points to the **python3.11** venv's tensorrt_libs, not the
python3.12 backend venv. This is now enforced in both shell profiles and in
`prod_start_triton.sh`.

**Rule for future agents:**
- Do NOT add `backend/.venv/lib/python3.12/site-packages/tensorrt_libs` to `LD_LIBRARY_PATH`.
- The correct path is `.venv/lib/python3.11/site-packages/tensorrt_libs`.
- To verify: `strings /path/to/libnvinfer.so.10 | grep -m1 kSERIALIZATION` or
  run `/home/bamby/grad_project/.venv/bin/python3 -c "import tensorrt; print(tensorrt.__version__)"`.
  Must print `10.8.0.43`.
- If engines are ever rebuilt with a newer TRT, update `LD_LIBRARY_PATH` to
  point to the matching venv's tensorrt_libs.

---

### Issue 3 — Duplicate Celery Workers Causing Job Stalls and Timeouts

**Symptom:**
- Video analysis jobs queue successfully but stall indefinitely or hit timeout.
- `ps aux | grep 'celery -A config'` shows 3× the expected number of processes
  (e.g., 3 sets of offline workers all consuming from the same queue).

**Root cause:**
The `prod_start_celery_workers.sh` script was run multiple times without stopping
the existing workers first. The PID file check (`kill -0 $OLD_PID`) only guards
against starting a second copy of a single named worker, but if the PID file was
stale or absent, a new set of workers launched alongside the existing ones.
Multiple workers competing on the same queue causes race conditions: one picks
up the task, the others time out waiting, creating apparent stalls.

**Fix applied (permanent):**
- `prod_stop_celery_workers.sh` must always be run before `prod_start_celery_workers.sh`.
- `prod_start_triton.sh` now also stops any existing Triton instance before
  launching a fresh one, preventing accumulation of stale processes.

**Rule for future agents:**
```bash
# Always stop before starting — never just start
bash tools/prod/prod_stop_celery_workers.sh
bash tools/prod/prod_start_celery_workers.sh
```
After starting, verify exactly one set of workers:
```bash
ps aux | grep 'celery -A config worker' | grep -v grep | wc -l
# Expected: 5 parent processes (one per queue) + their child forks
# If > 15 processes, there are duplicates — stop and restart
```

---

### Issue 4 — `PYRAMID_WORKER_COUNT=24` Fails Pydantic Validation

**Symptom:**
```
pydantic_core.ValidationError: 1 validation error for PipelineConfig
worker_count: Input should be less than or equal to 16
```
Celery workers crash immediately on startup; PID files are created but processes
exit within seconds.

**Root cause:**
`PipelineConfig` in `backend/apps/pipeline/config.py` had `le=16` on
`worker_count`. Setting `PYRAMID_WORKER_COUNT=24` (appropriate for a 32-core
machine) exceeded this limit.

**Fix applied (permanent):**
Raised the Pydantic limit from `le=16` to `le=32` in `config.py`. Committed
and pushed as `f878620`. Both prod and local now accept values up to 32.

**Rule for future agents:**
If you add `PYRAMID_WORKER_COUNT` to `.env` and workers crash immediately,
check the Celery log at `backend/logs/celery_*.log` for `ValidationError`
before assuming a runtime or import problem.

---

### Canonical Triton Start Procedure (after any restart or code change)

```bash
# 1. Stop all workers and Triton
bash /home/bamby/grad_project/tools/prod/prod_stop_celery_workers.sh

# 2. Start Triton (sources .bashrc for correct LD_LIBRARY_PATH)
bash -l /home/bamby/grad_project/tools/prod/prod_start_triton.sh

# 3. Wait for all 6 models to load (~60–90 seconds)
sleep 70

# 4. Verify Triton ready
curl -sf http://127.0.0.1:39100/v2/health/ready && echo READY || echo FAILED

# 5. Start Celery workers
cd /home/bamby/grad_project && bash tools/prod/prod_start_celery_workers.sh

# 6. Full health check
curl -sf http://127.0.0.1:8011/api/v1/health/ | python3 -m json.tool
```

### Quick Diagnostic Checklist

```bash
# Is Triton ready?
curl -sf http://127.0.0.1:39100/v2/health/ready && echo READY

# Which TRT version is loaded? (must print 10.8.0.43)
/home/bamby/grad_project/.venv/bin/python3 -c "import tensorrt; print(tensorrt.__version__)"

# Is LD_LIBRARY_PATH pointing to py3.11 venv?
echo $LD_LIBRARY_PATH | grep 'python3.11'

# How many Celery workers? (expect 5 parents + forks, no duplicates)
ps aux | grep 'celery -A config worker' | grep -v grep | wc -l

# GPU state
nvidia-smi --query-gpu=memory.used,memory.free,utilization.gpu --format=csv,noheader

# Are there duplicate workers?
ps aux | grep 'celery -A config worker' | grep -v grep | \
  awk '{print $12}' | sort | uniq -c | sort -rn | head -10
# If any queue name appears more than once → stop all workers and restart
```

### Key File Locations

| File | Purpose |
|------|---------|
| `tools/prod/prod_start_triton.sh` | Permanent Triton launcher (sets LD_LIBRARY_PATH correctly) |
| `tools/prod/prod_start_celery_workers.sh` | Starts all offline/default workers with tuned limits |
| `tools/prod/prod_stop_celery_workers.sh` | Stops all workers gracefully |
| `~/.bashrc` and `~/.profile` | Both export `LD_LIBRARY_PATH` with py3.11 TRT libs |
| `backend/.env` | Active runtime config authority for all services |
| `backend/logs/triton.log` | Triton startup and model load log |
| `backend/logs/celery_*.log` | Per-worker startup and task execution log |

---

## Notes About `xfail` Scaffold Tests
- Some tests are intentionally marked `xfail(strict=True)` as implementation scaffolds.
- `xfailed` is not a regression by itself.
- When behavior is fully implemented, remove `xfail` and convert to normal passing assertions in the same phase commit.

## Parallel Testing Policy
- Prefer framework-native parallelism first:
  - Frontend unit tests: Vitest workers.
  - Frontend E2E: Playwright workers.
  - Backend tests: `pytest-xdist` workers.
- Avoid nested parallel collisions:
  - Do not run multiple backend `pytest -n ...` invocations at the same time unless DB isolation is guaranteed per invocation.
- Use percentage-based worker tuning on shared machines.

## Standard Commands
- Frontend unit (parallel):
  - `npm run test:unit:parallel`
- Frontend E2E (parallel):
  - `npm run test:e2e:parallel`
- Frontend combined parallel suite:
  - `npm run test:all:parallel`
- Backend unit/integration/contract/system (parallel per suite):
  - `.\.venv\Scripts\python.exe -m pytest backend/tests/<suite> -n auto --dist=loadscope -q --tb=short`

## Automation Testing
- Use fast feedback first, then broaden scope:
  - PR gate: unit + API contract + smoke.
  - Pre-release: full E2E + accessibility + visual regression.
- Practical parallel commands:
  - Frontend automation (unit + E2E): `npm run test:all:parallel`
  - Backend automation by suite (run one suite at a time): `.\.venv\Scripts\python.exe -m pytest backend/tests/<suite> -n auto --dist=loadscope -q --tb=short`
  - API contract (parallel): `.\.venv\Scripts\python.exe -m pytest backend/tests/contract -n auto --dist=loadscope -q --tb=short`
  - Smoke/canary (parallel, tagged): `.\.venv\Scripts\python.exe -m pytest -m "smoke or canary" -n auto --dist=loadscope -q --tb=short`

## Load Testing
- Goal: validate throughput, latency, and error rate under concurrent traffic.
- Use staged runs:
  - Baseline: short warm-up + steady-state check.
  - Stress: ramp beyond expected peak.
  - Soak: longer duration for leak/drift detection.
- Practical parallel commands:
  - k6 local workers: `k6 run --vus 50 --duration 5m load-tests/baseline.js`
  - k6 scenario file: `k6 run load-tests/scenarios/stress.js`
  - Locust multi-process: `locust -f load-tests/locustfile.py --headless -u 200 -r 20 -t 10m --processes -1`

## Additional Test Types
- Performance regression:
  - `.\.venv\Scripts\python.exe -m pytest backend/tests/performance -n auto --dist=loadscope -q --tb=short`
- Chaos/resilience:
  - `.\.venv\Scripts\python.exe -m pytest backend/tests/resilience -n auto --dist=loadscope -q --tb=short`
- Security:
  - `npm audit --audit-level=high`
  - `.\.venv\Scripts\python.exe -m pip-audit`
- Accessibility:
  - `npx playwright test --grep @a11y --workers=50%`
- Visual regression:
  - `npx playwright test --grep @visual --workers=50%`
- API contract (consumer/provider):
  - `.\.venv\Scripts\python.exe -m pytest backend/tests/contract -n auto --dist=loadscope -q --tb=short`
- Smoke/canary:
  - `npx playwright test --grep @smoke --workers=50%`
  - `.\.venv\Scripts\python.exe -m pytest -m canary -n auto --dist=loadscope -q --tb=short`

## Best-Practice Rules
- Keep tests isolated: no shared mutable cross-test state.
- Mock external dependencies in E2E where practical to reduce flakiness.
- Keep coverage collection enabled, but enforce thresholds only through explicit configured gates.
- Do not introduce SQLite-only test shortcuts or fallback database settings; keep test and runtime behavior aligned with PostgreSQL.
