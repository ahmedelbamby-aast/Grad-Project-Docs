# Runtime Stability Remediation Plan

**Last updated:** 2026-05-29

## Status
- **Plan state:** Active
- **Execution state:** Not completed
- **Branch target:** `release/prod-runtime-stabilization`
- **Tasks file:** [docs/runtime_stability_remediation/tasks.md](tasks.md)
- **Evidence root:** [ci_evidence/production/runtime_maturity/](../../ci_evidence/production/runtime_maturity/)
- **Controlling plan:** [Heterogeneous Production Runtime Maturity Plan](../heterogeneous_production_runtime_maturity_plan.md)
- **Governing authority:** [Constitution v2.3.0](../../.specify/memory/constitution.md), Section 17 (Runtime Job Lifecycle and Vector Integrity Constitution)

## Why this plan exists

The maturity closure stalled on a single concrete failure: a fresh production
lifecycle acceptance job (`c375ea84-0407-4007-8ae8-d2adaf14d9e5`) never reached
a terminal state. The ingest command timed out with `status=processing`
(`ci_evidence/production/runtime_maturity/phase3_fresh_lifecycle_job.md`), which
blocked tasks **T038, T046, T047, T048, T051–T055, T060** of the controlling
maturity plan.

Commit `441c28e` ("enhance embedding generation and reconciliation for stale
jobs") patched the immediate symptoms. This plan converts those point fixes into
**enforced, non-bypassable runtime invariants** and closes the gaps the point
fixes left open, so the same class of failure cannot recur and cannot silently
re-enter the codebase.

This plan is scoped to **runtime stability and evidence integrity**. It does not
introduce new behavioral-intelligence capability. Production GPU/lifecycle truth
remains certifiable only on the native Linux RTX 5090 server per the controlling
plan and Constitution Section 16.

## Root-cause issue catalog

Each issue below is a *class*, not a single bug. The remediation makes each
class fail-closed and adds a regression gate so it cannot reappear.

### ISSUE-1 — Unbounded non-terminal job lifecycle
- **Symptom:** Job `c375ea84` hung in `processing`; the lifecycle command timed
  out instead of the job reaching a terminal status.
- **Root cause:** No stage had a wall-clock deadline, and the stale-job
  reconciler (`reconcile_runtime_workflows`) exists only as a **manual**
  management command — it is **not** in `beat_schedule`
  ([backend/config/celery.py:86](../../backend/config/celery.py)), so nothing
  automatically finalizes a stuck job in production.
- **Why it recurs without a guard:** A hang in any sub-stage (decode, inference,
  embedding, persistence) leaves the job indefinitely `processing`/`embedding`
  with no actor responsible for terminal resolution.
- **Constitution pillar:** §17.1 Bounded Job Lifecycle and Terminal-State Guarantee.

### ISSUE-2 — Embedding/feature vector dimension explosion
- **Symptom:** Oversized embedding payloads strained PostgreSQL/Redis; the cv2
  fallback produced 128×256×3 = 98,304 values and backbone output produced
  large variable-length vectors.
- **Root cause:** `persist_embedding` writes via `FrameEmbedding.objects.create()`
  ([backend/apps/tracking/embeddings.py:239](../../backend/apps/tracking/embeddings.py)),
  which does **not** invoke the `_validate_embedding_vector` 768-dim check on the
  model ([backend/apps/video_analysis/models.py:86](../../backend/apps/video_analysis/models.py)).
  The dimension contract holds only if the upstream coercion helper is called —
  the **write boundary itself is unguarded**.
- **Why it recurs without a guard:** Any new producer path (a new model, a new
  fallback, a refactor) that bypasses `_coerce_embedding_dimension` can write an
  arbitrary-size vector straight into the JSONField.
- **Constitution pillar:** §17.2 Fixed-Dimension Vector Contract.

### ISSUE-3 — Silently swallowed stage errors
- **Symptom:** Embedding persistence exceptions were logged and swallowed; the
  stage could "finish" having produced zero embeddings while the job advanced.
- **Root cause:** `generate_embeddings` counts errors (added in `441c28e`) but
  does **not** fail closed on a high error ratio or zero-output condition.
- **Why it recurs without a guard:** A per-item `except: log` pattern lets a
  totally failed stage report success.
- **Constitution pillar:** §17.3 Stage Outcome Accounting and Fail-Closed Thresholds.

### ISSUE-4 — Non-idempotent stage re-entry
- **Symptom:** Re-running embedding generation re-created embeddings for
  detections that already had them.
- **Root cause:** No existence guard before durable write (the
  `detection.embeddings.exists()` skip was added in `441c28e` for embeddings but
  the invariant is not generalized or test-locked across stages).
- **Why it recurs without a guard:** Retries/replays multiply durable evidence
  and inflate telemetry.
- **Constitution pillar:** §17.4 Stage Re-Entry Idempotency.

### ISSUE-5 — Timeout treated as an inconclusive result instead of a failure
- **Symptom:** The acceptance evidence file recorded a `CommandError: Timed
  out ... status=processing` and the maturity run simply stalled.
- **Root cause:** A lifecycle job used as acceptance evidence is only valid when
  it reaches a terminal status; a timeout is neither pass nor a recorded fail.
- **Constitution pillar:** §17.1 (lifecycle proof) + §16.7 (replay/evidence integrity).

## What commit `441c28e` already did (baseline)

These are in place and must be **preserved and test-locked**, not re-implemented:
- `_coerce_embedding_dimension()` bucket-averages/repeats any vector to exactly
  768 values; cv2 fallback reduced to 16×16 RGB (768 values).
- `generate_embeddings` skips detections that already have an embedding and
  records `embedding_summary` counts in job metadata.
- `reconcile_runtime_workflows` marks stale `processing`/`embedding` jobs
  (older than `--stale-minutes`, default 30) as `FAILED` with reason and
  PostgreSQL evidence output, with unit tests in
  `backend/tests/unit/video_analysis/test_management_commands.py`.

## Remediation phases

### Phase R1 — Vector integrity at the write boundary (ISSUE-2)
Enforce the dimension/value/payload contract where the row is written, so no
producer path can bypass it.
- Validate (or coerce-then-validate) inside `persist_embedding` before commit.
- Add an explicit maximum payload-size guard before DB/Redis writes.
- Lock with regression tests proving an oversized/wrong-dimension vector is
  rejected or deterministically coerced at the boundary.

### Phase R2 — Bounded lifecycle + automatic reconciler (ISSUE-1, ISSUE-5)
Guarantee no job stays non-terminal and make finalization automatic.
- Add a per-stage wall-clock deadline to the processing/embedding tasks.
- Register `reconcile_runtime_workflows` (or an equivalent task) in
  `beat_schedule` so production finalizes stale jobs without manual runs.
- Lock with tests proving the periodic reconciler is scheduled and that a job
  past its deadline is forced terminal.

### Phase R3 — Stage outcome accounting + fail-closed (ISSUE-3)
- Make `generate_embeddings` (and equivalent loop stages) fail closed when the
  error ratio exceeds a declared threshold or when zero required outputs were
  produced.
- Persist created/skipped/error counts as the authoritative stage outcome.
- Lock with tests for all-fail, partial-fail, and zero-output conditions.

### Phase R4 — Idempotency generalization (ISSUE-4)
- Confirm/extend existence-guarded writes for embedding and any sibling
  durable-write stage; document the idempotency key per stage.
- Lock with re-run tests proving no duplicate durable rows.

### Phase R5 — Production lifecycle re-validation (unblocks maturity closure)
- Sync the fix SHA to production, reconcile the stuck `c375ea84` lineage, and
  run a fresh real-video lifecycle job to a terminal `completed` status.
- This directly unblocks controlling-plan tasks **T038, T046, T047, T048**.

### Phase R6 — Maturity closure handoff
- Hand the accepted job back to the controlling maturity plan for **T046–T048**
  (GPU trace, causality, manifest) and **T051–T055, T060** (closure, parity,
  hygiene, merge).
- Propagate the new Section 17 gates into Spec Kit templates and `AGENTS.md`.

## Acceptance criteria
- A fresh production lifecycle job reaches terminal `completed` (not a timeout)
  and its evidence is captured under
  `ci_evidence/production/runtime_maturity/phase3_fresh_lifecycle_job.md`.
- No `FrameEmbedding` row can be written with a vector that is not exactly 768
  numeric values; proven by a boundary regression test.
- A job that exceeds its stage deadline is automatically forced to a terminal
  status by a scheduled reconciler; proven by a scheduling test + behavior test.
- An embedding stage producing zero required outputs fails the job; proven by a
  fail-closed test.
- Re-running a completed embedding stage creates zero duplicate rows; proven by
  an idempotency test.
- The focused backend suite (env/strategy/runtime-mode/replay/lifecycle/vector)
  passes on PostgreSQL with no hidden xfails.

## Test plan
- **Local (Windows/PostgreSQL, contract authority only):**
  - vector boundary rejection/coercion tests
  - reconciler scheduling + forced-terminal tests
  - stage fail-closed + idempotency tests
  - existing env/strategy/runtime-mode/replay suites stay green
- **Production (RTX 5090 Linux, runtime authority):**
  - preflight, fresh lifecycle job to terminal `completed`, GPU telemetry,
    causality export, evidence manifest — per controlling-plan §Production Linux.

## Constitution mapping
| Issue | New pillar | Existing reinforced |
| --- | --- | --- |
| ISSUE-1 | §17.1 | §14.6 Runtime Reconciliation, §8.2 task deadline |
| ISSUE-2 | §17.2 | §9.1 contract versioning, §10.1 typed sequence |
| ISSUE-3 | §17.3 | §14.23 No Silent Failure |
| ISSUE-4 | §17.4 | §10.2 idempotency |
| ISSUE-5 | §17.1 | §16.7 replay/evidence integrity |

## Assumptions
- PostgreSQL is the only relational authority; SQLite paths are invalid.
- Production remains native Linux, no Docker, no sudo; GPU truth is RTX 5090 only.
- Triton-only inference and single-active-endpoint policy are unchanged.
- The 768 embedding dimension is the current governed value; a change is a
  schema + migration governance decision.
