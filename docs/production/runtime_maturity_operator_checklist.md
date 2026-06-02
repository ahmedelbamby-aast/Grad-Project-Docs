# Runtime Maturity Operator Checklist

**Last updated:** 2026-05-27

Use this checklist with [docs/heterogeneous_production_runtime_maturity_plan.md](../heterogeneous_production_runtime_maturity_plan.md) and [docs/heterogeneous_production_runtime_maturity/tasks.md](../heterogeneous_production_runtime_maturity/tasks.md).

## Before Production Execution

- Confirm local branch is `release/prod-runtime-stabilization`.
- Confirm production repo is `/home/bamby/grad_project`.
- Confirm production backend env authority is `/home/bamby/grad_project/backend/.env`.
- Confirm repo-root `.env` is not used for backend startup.
- Confirm PostgreSQL is the only relational backend.
- Confirm production inference strategy is `INFERENCE_STRATEGY=triton_only`.
- Confirm exactly one Triton execution mode is active: `TRITON_EXECUTION_MODE=live` or `TRITON_EXECUTION_MODE=offline`.
- Confirm inactive Triton endpoint is unreachable.
- Confirm no production command prints raw secrets or GitHub tokens.

## Evidence Capture Order

1. Capture hash parity with `tools/prod/prod-hash-parity.ps1`.
2. Capture stash hygiene with `tools/prod/prod-stash-hygiene.ps1`.
3. Capture runtime preflight with `tools/prod/prod-runtime-preflight.sh`.
4. Capture endpoint policy with `tools/prod/prod_triton_endpoint_policy.sh`.
5. Capture worker status with `tools/prod/prod-workers.ps1 status`.
6. Run a fresh lifecycle job with explicit replay policy.
7. Capture GPU telemetry during the accepted job.
8. Export causality for the exact accepted job ID.
9. Generate the evidence manifest.
10. Verify final hash parity and production hygiene.

## Closure Criteria

- Local HEAD, origin HEAD, and production HEAD match.
- Production tracked tree has no unexplained modifications.
- Production stash entries are reviewed and archived or dropped only when understood.
- Runtime preflight passes on production.
- Active Triton endpoint returns ready and inactive endpoint is unreachable.
- Fresh lifecycle job succeeds through canonical API admission, Celery, Triton, PostgreSQL, and evidence export.
- Replayed failed lineage is rejected for acceptance.
- Final evidence package references one accepted job ID, Git SHA, env fingerprint, runtime profile, model artifact hashes, queue topology, GPU trace, and causality export.

## Final Evidence Package Validation

- `phase4_evidence_manifest.json` exists and uses `schema_version=production-wave-evidence.v1`.
- `runtime_maturity.job.job_id` matches the accepted lifecycle job.
- `runtime_maturity.job.replay_policy` is `reuse-success` for successful replay validation or `new-attempt` for a fresh linked attempt.
- `runtime_maturity.artifacts.gpu_trace`, `queue_topology`, `causality_export`, and `model_artifact_hashes` all include SHA-256 references.
- `runtime_maturity.branch_parity.local_head`, `origin_head`, and `production_head` match before final merge.
- `final/hash_parity.md`, `final/prod_hygiene.md`, and `final/production_acceptance_summary.md` are captured from production, not local simulation.
