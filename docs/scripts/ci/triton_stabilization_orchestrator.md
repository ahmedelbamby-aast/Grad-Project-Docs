# scripts/ci/triton_stabilization_orchestrator.py

Unified CI orchestrator for Triton stabilization phases:

- Phase 0 baseline lock (`triton_phase0_benchmark_lock.py`)
- Phase 3 profile jobs filtering (`triton_profile_jobs.py`)
- Phase 4 memory-pool sweep wrapper (`run_triton_memory_pool_sweep.py`)
- Phase 5 protocol decision harness (`protocol_decision_harness.py`)

The orchestrator reuses existing scripts through subprocess calls and does not duplicate phase logic.

## Defaults

- Mode defaults to dry-run (no phase execution).
- Use `--execute` to run enabled phases.
- Per-step toggles:
  - `--no-phase0`
  - `--no-phase3`
  - `--no-phase4`
  - `--no-phase5`

## Artifacts

Each run writes a summary bundle under:

- `scripts/ci/artifacts/triton_stabilization/<run_label>/triton_stabilization_summary.json`
- `scripts/ci/artifacts/triton_stabilization/<run_label>/triton_stabilization_summary.md`

`<run_label>` defaults to a UTC timestamp unless overridden with `--run-label`.

## Examples

Dry-run plan generation:

```powershell
python scripts/ci/triton_stabilization_orchestrator.py --run-label dryrun_example
```

Dry-run with concrete Phase 3 + Phase 5 template artifacts:

```powershell
python scripts/ci/triton_stabilization_orchestrator.py `
  --run-label dryrun_templates `
  --phase3-spec scripts/ci/specs/triton_phase3_profile_jobs.yaml `
  --phase3-results-in scripts/ci/specs/triton_phase3_results_in_example.yaml `
  --phase3-results-out scripts/ci/artifacts/triton_stabilization/dryrun_templates/phase3/triton_profile_filtered_results.json `
  --phase5-http-artifact scripts/ci/specs/triton_phase5_http_artifact_example.json `
  --phase5-grpc-artifact scripts/ci/specs/triton_phase5_grpc_artifact_example.json
```

Execute all phases (requires phase inputs, especially Phase 5 HTTP/gRPC sources):

```powershell
python scripts/ci/triton_stabilization_orchestrator.py `
  --execute `
  --phase3-spec scripts/ci/specs/triton_phase3_profile_jobs.yaml `
  --phase5-http-artifact scripts/ci/artifacts/http_results.json `
  --phase5-grpc-artifact scripts/ci/artifacts/grpc_results.json
```

Execute with selected phases only:

```powershell
python scripts/ci/triton_stabilization_orchestrator.py --execute --no-phase5
```
