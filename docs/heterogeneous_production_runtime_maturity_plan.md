# Heterogeneous Production Runtime Maturity Plan

**Last updated:** 2026-05-29

## Status
- **Plan state:** Active
- **Execution state:** Not completed
- **Current branch target:** `release/prod-runtime-stabilization`
- **Tasks file:** [docs/heterogeneous_production_runtime_maturity/tasks.md](heterogeneous_production_runtime_maturity/tasks.md)
- **Evidence root:** [ci_evidence/production/runtime_maturity/](../ci_evidence/production/runtime_maturity/)
- **Local workflow:** [.github/workflows/prod-runtime-maturity.yml](../.github/workflows/prod-runtime-maturity.yml)
- **Production evidence helper:** [tools/prod/prod-runtime-maturity-evidence.ps1](../tools/prod/prod-runtime-maturity-evidence.ps1)
- **Current authority:** Production runtime behavior is authoritative only when validated on the native Linux RTX 5090 server with PostgreSQL, Redis, Celery, Triton, and evidence tied to a committed Git SHA.
- **Supersedes:** `docs/linux_production_optimization_execution_phases.md` as the current top-level agent plan. The older optimization plan remains useful phase context, but agents must execute this maturity plan first when the two differ.

## Summary
Development and production are intentionally different and must not be forced into environment symmetry. The target is runtime behavioral consistency across heterogeneous systems: Windows/Docker/RTX 3050 Ti for development convenience, and native Linux/no-Docker/RTX 5090 for production authority.

The production system must be deterministic because its job is different from dev: it is the only authority for inference execution, GPU evidence, throughput claims, and final lifecycle validation. Dev may prove code behavior and contracts; prod proves runtime truth.

## Runtime Authority Model
- Production authority is isolated to:
  - committed Git SHA deployed on the production branch
  - `backend/.env` as the only backend runtime env authority
  - PostgreSQL for relational state
  - Redis for broker/cache behavior
  - Celery for governed orchestration
  - Triton for all production inference
  - production evidence under `ci_evidence/production/...`
- Development authority is limited to:
  - feature development
  - debugging
  - unit/contract tests
  - local Docker validation
  - non-authoritative experimentation
- Why: Windows, Docker, RTX 3050 Ti VRAM, and local CUDA/PyTorch stacks cannot prove production behavior on RTX 5090 Linux.
- Failure prevented: accepting a local pass as production truth when prod has different CUDA, TensorRT, GPU architecture, queue timing, process model, or throughput behavior.

## Production Branch and Sync Governance
- Use a short-lived `release/prod-runtime-stabilization` branch for this stabilization cycle.
- Do not create a permanent production-only fork. All production fixes must merge back into the active development branch after sign-off.
- Production deploy acceptance requires:
  - local HEAD equals origin HEAD
  - production HEAD equals origin HEAD
  - production working tree has no unexplained tracked changes
  - production stash entries are reviewed, archived, then dropped only when understood
- Why: the root problem is not that prod differs from dev; it is that prod changes become hidden, dirty, and unreviewable.
- Failure prevented: production-only hotfixes that disappear, stale code running on prod, and evidence that cannot be tied to a commit.

## Triton-Only Production Enforcement
- Production must run with `INFERENCE_STRATEGY=triton_only`.
- Any local Ultralytics, PyTorch CUDA, OpenCV DNN GPU, or direct model inference path in production must fail closed before processing.
- Triton is the compatibility boundary between application code and GPU execution.
- Production startup must reject:
  - local model loading in production
  - fallback from Triton to local CUDA
  - fallback from GPU to CPU for production claims
  - model-loaded or memory-allocated evidence without real inference traces
- Why: RTX 5090 `sm_120` exposed that local PyTorch kernels were incompatible. Triton isolates production inference from developer-local CUDA/PyTorch instability.
- Failure prevented: accidental local CUDA execution, kernel image mismatch, silent CPU fallback, false GPU evidence.

## Runtime Contract Preflight
Production must fail before processing any video if contract checks fail.

Required preflight evidence:
- OS/runtime: native Linux, no Docker assumption, no sudo dependency
- Python: backend venv path resolves consistently
- GPU: RTX 5090 identity, VRAM, driver, CUDA compatibility, architecture fingerprint
- TensorRT: runtime version, engine compatibility manifest, artifact digests
- Triton: pinned binary, active endpoint readiness, inactive endpoint unreachable
- Models: required model configs, artifact hashes, active profile binding
- Env: `backend/.env` fingerprint, required keys, no repo-root `.env` backend authority
- Queues: Celery routes, worker queues, concurrency, prefetch policy
- Storage/state: PostgreSQL authority, Redis reachability, evidence output paths

Why: production must reject invalid runtime shape before doing expensive work.

Failure prevented: starting workers against wrong endpoints, stale TensorRT engines, wrong env file, hidden queue route drift, or invalid model artifacts.

## GPU Compatibility Strategy
- Required to match across dev/prod:
  - committed code contracts
  - inference request/response schemas
  - model names and semantic outputs
  - runtime mode semantics
  - telemetry schema
  - replay and evidence contracts
  - PostgreSQL-backed persistence behavior
- Allowed to differ:
  - OS
  - Docker usage
  - GPU architecture
  - VRAM capacity
  - CUDA/PyTorch local stack
  - throughput and latency numbers
  - TensorRT engine binaries, if production lineage records the authoritative build
- Dev RTX 3050 Ti validates behavior only within low-VRAM limits.
- Prod RTX 5090 validates throughput, GPU utilization, batching, thermal stability, and final inference authority.
- Why: behavioral correctness can be portable; performance truth is hardware-specific.
- Failure prevented: chasing identical throughput across unlike GPUs, or blocking development because local hardware cannot mirror production.

## Validation Hierarchy
- Local Windows validation may include:
  - unit tests
  - schema and contract tests
  - replay logic tests
  - queue routing logic tests
  - env parser tests
  - non-GPU causality export tests
  - Docker-based local smoke checks
- Production Linux validation must include:
  - real upload lifecycle through canonical API admission, Celery, Triton, PostgreSQL, and evidence export
  - GPU utilization and saturation checks
  - TensorRT/Triton readiness
  - live/offline endpoint isolation
  - queue and worker telemetry
  - dynamic batching evidence
  - long-running runtime stability evidence
- CI may gate correctness, but cannot certify production GPU behavior unless running on the production-equivalent self-hosted GPU lane with production evidence.
- Why: validation should match the risk being validated.
- Failure prevented: using a dev smoke test as a production GPU acceptance gate.

## Replay and Evidence Integrity
- `runtime_ingest_video` must stop silently accepting failed replay lineage as valid evidence.
- Add explicit replay policies:
  - `reuse-success`: only reuses a previous completed job
  - `fail-on-existing`: rejects any existing replay key
  - `new-attempt`: creates a fresh job linked to the replay lineage
- Benchmark and acceptance runs must default to fresh production validation or successful replay only.
- A replayed failed job must be marked invalid for production acceptance.
- Evidence packages must include:
  - job ID
  - replay key and replay policy
  - source hash
  - Git SHA
  - env fingerprint
  - runtime profile
  - model artifact hashes
  - queue topology
  - GPU trace
  - causality export
- Why: a failed command returned `"replayed": true` and reused failed lineage.
- Failure prevented: old failures being mistaken for fresh validation, or old successes hiding current drift.

## Deterministic Runtime Behavior
- Runtime mode selection must be explicit: `live` or `offline`.
- Only one Triton endpoint profile may be production-active at a time.
- Inactive endpoint must be unreachable and must not receive production traffic.
- No environment-dependent inference switching is allowed.
- Queue ownership must be explicit; no implicit queue route may process production inference.
- Runtime profile must declare batching, overlay, persistence, telemetry, throttling, and degradation behavior.
- Why: the runtime is distributed; determinism requires every boundary to be named and checked.
- Failure prevented: live/offline cross-talk, fake health, hidden local fallback, queue starvation, and profile-dependent surprises.

## Operational Safety
- Developers may keep inconsistent local CUDA/PyTorch stacks without endangering production because production never trusts local inference paths.
- Weak local GPUs are acceptable because local tests are not production throughput gates.
- Windows/Linux differences are contained by contract tests locally and runtime preflight in production.
- No-sudo production remains viable by pinning user-space runtime paths, venv, Triton binary, env files, and evidence scripts.
- Production startup must prefer fail-closed over degraded inference when authority contracts are violated.
- Why: operational reliability comes from narrowing authority, not pretending every machine is equivalent.
- Failure prevented: production breakage caused by developer-local assumptions.

## Long-Term Maturity Direction
- Near-term target: governed native Linux runtime because production has no sudo and no Docker.
- Medium-term target: immutable user-space runtime package:
  - pinned Python wheel lock
  - pinned Triton binary path
  - TensorRT compatibility manifest
  - model artifact digest manifest
  - env contract fingerprint
  - reproducible production evidence bundle
- Long-term optional target: containerized production only if infrastructure constraints change and GPU/Triton/container support can be governed without sudo.
- Do not pursue Docker everywhere now; it conflicts with the current production boundary.
- Why: maturity means reproducibility under real constraints, not forcing a deployment model the server cannot support.
- Failure prevented: spending effort on an idealized container target while current native production remains dirty and fragile.

## Execution Phases
1. Commit and branch stabilization:
   - commit current production fixes
   - create `release/prod-runtime-stabilization`
   - sync prod to that exact branch
   - verify hash parity and stash hygiene
2. Contract lock:
   - implement production runtime doctor/preflight
   - update prod helper defaults to target the release branch
   - enforce `backend/.env` authority
   - block local inference in production
3. Replay and lifecycle hardening:
   - add explicit replay policy
   - reject failed replay as acceptance evidence
   - run a fresh real-video lifecycle job on production
4. GPU and causality certification:
   - capture GPU telemetry during the actual job
   - export causality for the exact job ID
   - prove endpoint isolation, queue stability, model calls, batching behavior, and evidence artifacts
5. Closure:
   - package evidence
   - merge release branch back
   - leave production clean and reproducible

## Phase Status (updated 2026-05-29)
- Phase 1, commit and branch stabilization: **COMPLETED** — branch created, hash parity verified, stash reviewed.
- Phase 2, contract lock: **COMPLETED** — preflight passing, Triton-only enforced, backend/.env authority, queue routes verified.
- Phase 3, replay and lifecycle hardening: **COMPLETED** — accepted job b1d2311c (status: completed, runtime_used: triton, 2936 detections, 0 errors); replay reuse-success verified.
- Phase 4, GPU and causality certification: **COMPLETED** — GPU trace captured (peak 11% SM at 100ms), causality exported (4 model calls, 0 failures), manifest generated.
- Phase 5, closure: **COMPLETED** — evidence packaged, hash parity final, stash reviewed, branch merged to master.

## Accepted Job
- **Job ID:** `b1d2311c-b0af-44a4-a551-61e58200eb11`
- **Final SHA:** `af3fce3` (local/origin/production in parity)
- **Evidence index:** `ci_evidence/production/runtime_maturity/final/evidence_package_index.md`

## Test Plan
- Local Windows:
  - env contract tests
  - runtime mode tests
  - local inference block tests
  - replay policy tests
  - queue route tests
  - causality schema tests
- Production Linux:
  - hash parity
  - runtime preflight
  - endpoint policy
  - worker queue inspection
  - fresh upload lifecycle run
  - GPU telemetry capture
  - causality export
  - successful replay validation
  - evidence package validation

## Assumptions
- Production remains native Linux, no Docker, no sudo.
- Production GPU tests must run on the RTX 5090 server.
- Development remains Windows/Docker/RTX 3050 Ti and is not production truth.
- Triton remains the only production inference authority.
- PostgreSQL remains the only relational authority.
- The production branch is temporary stabilization infrastructure, not a permanent fork.
