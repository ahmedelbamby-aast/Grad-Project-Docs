# scripts/ci/triton_memory_pool_sweep.py

## Purpose

Phase 4 memory-pool sweep harness for Triton runtime lock recommendations.

The harness sweeps combinations of:
- `TRITON_PINNED_MEMORY_POOL_BYTES`
- `TRITON_CUDA_MEMORY_POOL_BYTES`

It captures markers from command output and/or provided metrics artifacts, then emits:
- JSON report with full case records and lock recommendation
- Markdown summary report

Default mode is non-destructive dry run. Use `--execute` to run commands.

For CI-oriented orchestration and standardized artifact folders, use:
- `scripts/ci/run_triton_memory_pool_sweep.py`

## Spec Shape

```yaml
command_template: >
  powershell -NoProfile -Command "
  $env:TRITON_PINNED_MEMORY_POOL_BYTES={TRITON_PINNED_MEMORY_POOL_BYTES};
  $env:TRITON_CUDA_MEMORY_POOL_BYTES={TRITON_CUDA_MEMORY_POOL_BYTES};
  python scripts/ci/run_phase4_probe.py"

pools:
  pinned_memory_pool_bytes: [134217728, 268435456, 536870912]
  cuda_memory_pool_bytes: [67108864, 134217728, 268435456]
```

Notes:
- Placeholders in `command_template` are required exactly as shown:
  - `{TRITON_PINNED_MEMORY_POOL_BYTES}`
  - `{TRITON_CUDA_MEMORY_POOL_BYTES}`
- Missing pool arrays fall back to conservative defaults.

## Commands

Dry run (default):

```powershell
python scripts/ci/triton_memory_pool_sweep.py path/to/spec.yaml
```

Execute sweep and collect stdout/stderr markers:

```powershell
python scripts/ci/triton_memory_pool_sweep.py path/to/spec.yaml --execute
```

Execute and merge external metrics artifacts:

```powershell
python scripts/ci/triton_memory_pool_sweep.py path/to/spec.yaml `
  --execute `
  --metrics-artifact path/to/metrics.json `
  --metrics-artifact path/to/metrics.yaml
```

Custom report output paths:

```powershell
python scripts/ci/triton_memory_pool_sweep.py path/to/spec.yaml `
  --report-json scripts/ci/artifacts/triton_phase4_memory_pool/lock_report.json `
  --report-md docs/triton_phase4_memory_pool_lock_report.md
```

## Metrics Artifact Contract

Each artifact may be JSON or YAML and can be:
- top-level list, or
- mapping with `results` list.

Each row should include:
- `pinned_bytes`
- `cuda_bytes`
- optional `status` (`success`/`oom`/`unknown`)
- optional `latency_ms`
- optional `throughput`

Artifact values override parsed command markers for the same pool tuple.

## Output Contract

JSON payload includes:
- `cases[]` (command, tuple, status, throughput, latency, mode)
- `recommendation.recommended`
- `recommendation.reason`

Recommendation policy:
- only `success` rows are eligible
- scoring is `throughput - 0.1 * latency_ms`
- if no successful rows exist, recommendation is `null`
