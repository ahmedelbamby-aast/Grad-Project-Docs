# Triton Stabilization Spec Templates

**Last updated:** 2026-05-22

This directory contains concrete sample inputs to run the Triton stabilization orchestrator end-to-end in dry-run or execute mode.

## Phase 3

- Spec template: `triton_phase3_profile_jobs.yaml`
- Example measured results input for filtering: `triton_phase3_results_in_example.yaml`

Phase 3 standalone example:

```powershell
python scripts/ci/triton_profile_jobs.py scripts/ci/specs/triton_phase3_profile_jobs.yaml `
  --results-in scripts/ci/specs/triton_phase3_results_in_example.yaml `
  --results-out scripts/ci/artifacts/triton_profile_filtered_results_example.json
```

## Phase 5

- HTTP artifact template: `triton_phase5_http_artifact_example.json`
- gRPC artifact template: `triton_phase5_grpc_artifact_example.json`

Phase 5 standalone example:

```powershell
python scripts/ci/protocol_decision_harness.py `
  --http-artifact scripts/ci/specs/triton_phase5_http_artifact_example.json `
  --grpc-artifact scripts/ci/specs/triton_phase5_grpc_artifact_example.json `
  --decision-report scripts/ci/artifacts/protocol_decision_report_example.json `
  --summary-report docs/protocol_decision_summary_example.md
```
