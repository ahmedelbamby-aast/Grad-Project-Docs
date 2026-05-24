# scripts/ci/run_triton_memory_pool_sweep.py

## Purpose

CI-safe wrapper for Phase 4 Triton memory pool sweep runs.

It wraps `scripts/ci/triton_memory_pool_sweep.py` and standardizes artifact output under:

- `scripts/ci/artifacts/triton_phase4_memory_pool/<run_label>/`

Default behavior is safe dry-run. Commands from the spec execute only when `--execute` is passed.

## CI Workflow Wiring

GitHub Actions workflow: `.github/workflows/ci-bootstrap.yml` (`triton-stabilization` job)

- Pull requests run lint + dry-run only.
- Manual dispatch can optionally enable execute mode via `execute=true`.
- Each run uploads a stabilization artifact bundle from the resolved run label directory.

## Default Specs

- `scripts/ci/specs/triton_phase4_memory_pool_default.yaml`
- `scripts/ci/specs/triton_phase4_memory_pool_extended.yaml`

## Commands

Dry run with default spec:

```powershell
python scripts/ci/run_triton_memory_pool_sweep.py
```

Execute sweep commands:

```powershell
python scripts/ci/run_triton_memory_pool_sweep.py --execute
```

Run with explicit spec and stable artifact label:

```powershell
python scripts/ci/run_triton_memory_pool_sweep.py `
  --spec scripts/ci/specs/triton_phase4_memory_pool_extended.yaml `
  --run-label ci-main `
  --execute
```

Pass external metrics artifacts through to the sweep:

```powershell
python scripts/ci/run_triton_memory_pool_sweep.py `
  --metrics-artifact scripts/ci/artifacts/prior_run.json `
  --metrics-artifact scripts/ci/artifacts/prior_run.yaml
```

## Wrapper Artifacts

Each run directory includes:

- `wrapper_invocation.json` (mode/spec/command metadata)
- `memory_pool_sweep_report.json` (produced by sweep script)
- `memory_pool_sweep_report.md` (produced by sweep script)
- `case_*.stdout.txt`, `case_*.stderr.txt` (only in execute mode)
