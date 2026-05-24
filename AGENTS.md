# Agents Execution Guide

## Purpose
This file defines how agents should execute tests quickly and safely in this repository.

<!-- SPECKIT START -->
## Active Spec Kit Plan
- Active feature plan: [specs/009-parallel-pose-inference/plan.md](specs/009-parallel-pose-inference/plan.md)
<!-- SPECKIT END -->

## Current Active Optimization Plan
- **Active plan name:** `Linux Production Optimization Execution Phases (RTX 5090)`
- Plan file: [docs/linux_production_optimization_execution_phases.md](docs/linux_production_optimization_execution_phases.md)
- **Production policy update:** Triton is **single-endpoint mode only** (never run live/offline Triton endpoints at the same time).

## Current Phase Status (handover snapshot)
- M0: partially completed (sync/hygiene repeatedly applied; local untracked `tmp/*` still exists on dev laptop).
- M1: partially completed (profile/cadence env contract documented, continue hardening env examples).
- M2: largely completed (queue routing + worker scripts added).
- M3:
  - Phase 3.1: in progress/partially implemented (sparse detect + reuse present).
  - Phase 3.2: not started (bbox interpolation/clamp/continuity guards).
- M4: partially touched, not accepted (live cadence/reuse/telemetry still needs full validation).
- M5: policy changed to single-endpoint mode; implement/verify active endpoint binding only.
- M6: not completed (dynamic-batch TensorRT regeneration and tuned Triton configs still pending full sign-off).
- M7:
  - Phase 7.1: started (benchmark exporter/test scaffolding added).
  - Phase 7.2: not started (full profile matrix execution/report ranking).
- M8: not completed (final acceptance gates + full stash/sync closure).

## Production Server Details (Linux, no Docker, no sudo)
- Host: `0.tcp.eu.ngrok.io`
- SSH port: `27681`
- Username: `bamby`
- Repo path: `/home/bamby/grad_project`
- Runtime constraints:
  - no Docker
  - no sudo
  - Triton-only inference on NVIDIA GPU
  - single active Triton endpoint at a time
- Current commonly used app ports:
  - Backend: `127.0.0.1:8011`
  - Frontend: `127.0.0.1:5174`
  - Triton (offline profile mode): `39100/39101/39102`
  - Triton (live profile mode, only when selected): `39000/39001/39002`
  - Redis: `6379`
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
4. Confirm ports:
   - `ss -ltnp | egrep '39100|39000|8011|5174|6379|5432|55432'`
5. Run focused tests before full suites:
   - `backend/tests/unit/video_analysis/test_tasks_detection_cadence.py`
   - `backend/tests/unit/scripts/test_benchmark_export_csv.py`
   - scaffold integration/system tests (xfail markers are expected until converted)

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
