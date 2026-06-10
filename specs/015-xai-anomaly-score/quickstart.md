# Quickstart: Implementing An Atomic XAI Cycle

**Purpose**: Operational sequence for future implementation agents. This is a
plan quickstart, not evidence that any XAI cycle is implemented or accepted.

## 1. Read Authority

Before each cycle:

```text
AGENTS.md
.specify/memory/constitution.md
specs/015-xai-anomaly-score/spec.md
specs/015-xai-anomaly-score/plan.md
specs/015-xai-anomaly-score/no-ground-truth-doctrine.md
specs/015-xai-anomaly-score/atomic-cycles.md
specs/015-xai-anomaly-score/signal-catalog.md
docs/xai_anomaly/cycle_015_throughput_remediation_investigation.md
the cycle investigation/results docs
docs/production_inference_benchmark.md
docs/inference_parallelization_plan.md
docs/cycle_9_and_10_improvements_todo.md
```

## 1a. Throughput Target Authority

Before additive XAI/anomaly work can be accepted on the production critical
path, the canonical `combined.mp4` stride-1 benchmark must reach `>=15 FPS`
DB-completed end-to-end throughput. The practical batch envelope is `32` frames
completing their full authoritative cycle in `<=2` seconds.

This target includes inference, pose/behavior postprocess, PostgreSQL writes,
embeddings/derived records, telemetry/artifacts, reconciliation, and terminal
lifecycle state. Inference-only, direct Triton, frame-loop-only, or
progress-percent FPS cannot satisfy the target.

## 2. Refresh Runtime Truth

Cycle work begins from a fresh point-in-time production inventory, not from
`production-inventory-20260608.md` alone.

Capture:

- branch/SHA and clean/dirty state;
- active environment fingerprint;
- active Triton profile/endpoint/models/routes/artifacts;
- PostgreSQL/Redis/Celery/process/port state;
- active jobs;
- relevant relational row counts;
- instrumentation readiness;
- accepted baseline and exact benchmark media.

## 3. Kick Off One Atomic Cycle

Create/update:

```text
docs/xai_anomaly/cycle_015_<id>_investigation.md
docs/xai_anomaly/cycle_015_<id>_results.md
```

The investigation names:

- one causal hypothesis;
- exact implementation/config delta;
- streaming compatibility;
- baseline and candidate profiles;
- acceptance and veto gates;
- rollback;
- exactly one Figure Planner;
- exactly one separate Figure Implementer;
- raw artifact and Markdown targets.

## 4. Implement Behind A Reversible Flag

- Reuse existing ownership boundaries.
- Add one interface/adapter only when it removes real duplication.
- Do not add a parallel anomaly/XAI service.
- Enforce schema, payload, vector, idempotency, deadline, and missingness at the
  boundary.
- Keep deep XAI outside critical inference.
- Keep live state bounded.
- Do not add anomaly-model training/fine-tuning or treat reviewer assessments,
  heuristic output, BSIL output, model agreement, or assumed-normal history as
  ground truth.
- Build anomaly review priority from bounded per-student multivariate
  observed-signal pattern comparison.

## 4a. Bootstrap Reproducible Model Packs

Before local work that depends on pretrained artifacts, validate the Cycle 015
registry and download the declared runtime packs:

```powershell
.\.venv\Scripts\python.exe scripts/ci/verify_pretrained_models.py
python scripts/models/download-xai-registry-models.py --manifest specs/015-xai-anomaly-score/pretrained-model-acquisition-manifest.json --releases-dir releases/model-packs --runtime all --receipt backend/logs/xai_model_download_receipt.json
```

Then install the packs into `backend/models` using the existing bootstrap
scripts when needed:

```powershell
./scripts/models/bootstrap-models.ps1 -Runtime all -ModelsDir ./backend/models -ReleasesDir ./releases/model-packs
```

## 5. Validate Locally

Run focused tests first:

```powershell
git diff --check
python scripts/ci/verify_mermaid_diagrams.py --paths specs/015-xai-anomaly-score
python scripts/ci/verify_doc_dates_and_reading_order.py
```

Then run the cycle's focused backend/frontend/contract/integration tests.
PostgreSQL is the only accepted relational backend.

Local checks cannot create production acceptance.

## 6. Commit, Push, Pull Production

The user requires every implementation update to be committed/pushed and then
pulled on production before production validation.

```powershell
git status --short
git add <cycle-owned-files>
git commit -m "<cycle message>"
git push
.\tools\prod\prod-ssh.ps1 -Cmd "cd /home/bamby/grad_project && git pull --ff-only && git rev-parse --short HEAD"
```

Never discard unrelated production artifacts or user changes.

## 7. Run Production Preflight

Verify:

- production SHA equals pushed SHA;
- PostgreSQL/Redis/Celery/Triton readiness;
- one active Triton profile and inactive-profile rejection;
- exact route snapshot and config delta;
- no duplicate workers;
- no active job that would be interrupted;
- instrumentation `status=ok`;
- rollback command is ready.

## 8. Run Stride-1 Baseline And Candidate

- Use independent `new-attempt` jobs.
- Use the exact same governed media and route except for the one candidate
  variable.
- Record all required performance, correctness, calibration, explanation,
  anomaly, identity, queue/storage, and WebGL metrics.
- For anomaly behavior, record exact reconstruction, controlled fixtures,
  metamorphic/invariant/sensitivity, cold-start/contamination/drift/quarantine,
  withholding, stability, and knowledge-limit evidence. Do not report
  behavioral accuracy, precision, recall, F1, AUROC, AUPRC, false-positive, or
  false-negative claims.
- Component probes remain probe-only.

## 9. Generate Figures And Decide

- Figure Implementer generates figures from the exact raw decision artifacts.
- Validate manifest/digests and unavailable reasons.
- Embed/link figures in the cycle results doc.
- Record every run and decision in `docs/BENCHMARK_RESULTS_LEDGER.md`.
- Decide only after all veto gates and rollback proof exist.

## 10. Roll Back And Reconcile

Execute rollback even for an accepted candidate when the cycle requires
rollback proof. Verify route, task, queue, DB, artifact, telemetry, frontend,
and user-visible state convergence.

## 11. Acceptance Vocabulary

Allowed cycle states:

```text
ACCEPTED
NOT ACCEPTED
NEEDS FURTHER ITERATION
PROBE_ONLY
HYPOTHESIS_ONLY
STAGED
ROLLBACK
```

No score or explanation is described as proof of cheating or intent.
`within_observed_pattern` is not proof of normal/non-cheating behavior, and
`pattern_deviation` is not proof of abnormal intent or cheating.
