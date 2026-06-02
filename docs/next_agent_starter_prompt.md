# Starter Prompt — Next Agent on the Crop-Frame Optimization Plan

**Purpose:** Paste this into a fresh agent session at session start. It is
self-contained — read it once, then act. Every link in here points to a
file that already exists in this repository (verify with `git ls-files`).

**Last updated:** 2026-06-02 (after Cycle 9b exact slice + Top-K was accepted
with caveat).

---

## 0. Who you are and what you are doing

You are continuing the **crop-frame inference pipeline optimization plan**
on a Django + Celery + Triton stack that processes classroom videos
end-to-end on a production Linux server with an RTX 5090.

The goal of the plan is to hit a hard SLA:

> **`total_wall_clock(upload → DB status=completed) ≤ duration(video) + 5 min`**

For the canonical benchmark video `combined.mp4` (4 541 frames, 2 m 31 s
@ 30 fps), that means **total wall ≤ 7 m 31 s = 451 s = ≥ 10.07 FPS
overall**.

**Current accepted baseline** (Cycle 9b exact slice + Top-K, job `be4ba9ee`):
- Total wall: **17.0 min** (`1022.952 s`)
- Overall FPS (DB completed basis): **4.429**
- **Gap to SLA: ~9.5 min**

You are not at the SLA. The plan is in mid-flight. Cycles 1–8 are ACCEPTED;
Cycle 9 ran on prod but FAILED its Step 2 gate; Cycle 9b B.2.b exact slice is
ACCEPTED; Cycle 9b B.2.c exact slice + Top-K is ACCEPTED WITH CAVEAT; Cycle 10
LPM ran on prod but is NOT ACCEPTED and remains rolled back with
`LPM_ENABLED=0`. Future cycles (9b remaining, 10b, 11, 12, 13a, 13b) are
planned.

You will not silently expand scope. You will not declare anything
ACCEPTED without prod evidence on the Linux RTX 5090 server.

---

## 1. Mandatory reading list — read these in this order before you touch any code

| Step | File | Why |
|---|---|---|
| 1 | [`docs/cycle_9_and_10_improvements_todo.md`](cycle_9_and_10_improvements_todo.md) | The active TODO. § Z is the **single entry point** that maps every cycle (accepted, not accepted, staged, planned, deferred) and points to each one's docs. Start at § Z; come back to § A; then read whichever §B / §C item you intend to work on. |
| 2 | [`docs/runtime_sla_video_plus_5min.md`](runtime_sla_video_plus_5min.md) | The SLA contract. Memorise the math: 4 541 frames / 451 s = 10.07 FPS. |
| 3 | [`AGENTS.md`](../AGENTS.md) | Production server details, repo path, Triton paths, all prior cycle outcomes registered, the constitutional re-affirmation. |
| 4 | [`docs/cycle_9_results.md`](cycle_9_results.md) | The post-mortem for why Cycle 9 failed its Step 2 wall gate even though it was working code with measurable improvement. This is the precedent for "NOT ACCEPTED" and the canonical example of why you must name the lever you pull. |
| 5 | [`docs/triton_models_and_tensor_anatomy.md`](triton_models_and_tensor_anatomy.md) | What is loaded in Triton, what every input / output tensor contains, byte math for the dense-output inefficiency that drives Cycle 9b. |
| 6 | [`docs/logical_path_matrix_spec.md`](logical_path_matrix_spec.md) | The Cycle 10 LPM formal spec. § 2 (scope) and § 10 (acceptance gates) are the parts you cannot skip. |
| 7 | [`docs/cycles_9_to_12_implementation_playbook.md`](cycles_9_to_12_implementation_playbook.md) | The original Cycle 9 → 12 plan plus the Cycle 9 prod-outcome appendix. |

If you have not read all 7 you are not ready to write code.

---

## 2. The constitutional rules (NON-NEGOTIABLE)

These are gating conditions, not guidelines. Any commit that violates one
must be reverted and re-done.

1. **No "accepted" without a production benchmark on the Linux RTX 5090
   server** that demonstrates the targeted metric improvement AND zero
   correctness regression vs. the prior accepted baseline. Code review,
   parity probes, local pytest passes, and PR approvals are necessary but
   never sufficient.

2. **Every optimization-cycle hypothesis must name the lever it pulls**
   before any code is written. Exactly one of:
   - **GPU child compute** — reduce server-side TensorRT execution time
   - **Dense output bytes** — reduce bytes returned from GPU to Python
   - **Single-process Python orchestration** — reduce per-frame Python
     critical path that the GPU is waiting on

   Cycle 9 failed because it reduced gRPC call count (already absorbed
   by concurrent dispatch) without moving any of those three levers.
   Don't repeat that.

3. **LPM scope is HARD-RESTRICTED to the 3 gaze models.** Code that
   mutates RTMPose / person_detector / sitting_standing outputs must fail
   loudly via the structural exclusion tests in
   `backend/tests/unit/pipeline/test_logical_path_matrix.py`. The three
   gaze models are the entire LPM scope; nothing else.

4. **CI/CD workflow updates are MANDATORY, not optional.** Every code
   change must update `.github/workflows/inference-parallelization.yml`
   in the same commit:
   - New test files listed under both `on.push.paths` and
     `on.pull_request.paths` AND inside the `Run inference parallelization
     tests` pytest invocation.
   - New app modules listed under the same path filter.
   - New docs (results files) listed under the same path filter.
   - Every new `.env` flag exercised by at least one unit test that runs
     in CI.

5. **No option, optimization, or strategy is accepted until it has been
   really tested, measured, and compared to the SLA Baseline on the
   production Linux server.** Three required artefacts from the prod
   Linux RTX 5090:
   - `backend/logs/bench_summary_*.json` (from `prod_run_parallel_flow_benchmark.sh`)
   - `backend/data/videos/<job_id>/inference_audit.json`
   - `backend/logs/gpu_monitor_bench_*.csv`

   The metrics must be recorded in a row of the matching
   `docs/cycle_*_results.md` file alongside the latest accepted baseline
   and the SLA target row. No prod artefacts → STAGED, not ACCEPTED.

6. **Multi-approach items (TODO § B.1, § B.2, § B.3) must measure ALL
   their approaches on prod before one is selected.** A
   `docs/cycle_9b_*_results.md` file captures the comparison matrix and is
   the source of truth for the tradeoff. Selection is `.env`-controlled.

These rules also live in
[`docs/cycle_9_and_10_improvements_todo.md`](cycle_9_and_10_improvements_todo.md)
§ E and § Z.6. If you update them in one place, update both.

---

## 3. Workflow protocol — the 6-phase loop every cycle MUST follow

Every cycle you start goes through these six phases in order. You may
combine adjacent phases in one commit only when each has its own evidence.

| Phase | What you do | Done when |
|---|---|---|
| **Phase 1 — Investigate** | Measure the bottleneck on a prod baseline. Pull `nvidia-smi`, Triton `/v2/models/<m>/stats`, `inference_audit.json`, GPU monitor CSV. Identify the dominant cost. | You can point to a specific number (e.g. "embedding stage is 450.7 s of the 1 582 s total = 28.6 %") and the file that contains it. |
| **Phase 2 — Hypothesize** | Write the hypothesis in `docs/crop_frame_optimization_execution.md` BEFORE coding. State which of the 3 levers (rule #2) you are pulling, the expected delta, the risk, and the rollback. | The hypothesis section names the lever explicitly and the rollback is one env-flag flip or one `git revert`. |
| **Phase 3 — Implement + local tests** | One focused change. Unit tests cover the new behavior. CI workflow updated per rule #4 in the same commit. | Local pytest green AND CI YAML updated AND every new env flag exercised by a test. |
| **Phase 4 — Production benchmark** | Push, sync prod, run `tools/prod/prod_run_parallel_flow_benchmark.sh` against `combined.mp4`. Capture the three artefacts (rule #5). | Job reached DB `status=completed`, bench summary JSON exists, inference_audit exists, GPU monitor CSV exists. |
| **Phase 5 — Compare** | Add a row to the matching `docs/cycle_*_results.md` file with **before** (prior accepted baseline) and **after** (this cycle), every metric the cycle targeted, and every correctness counter (detections / bboxes / embeddings / per-class boxes). | The row exists, references the job ID + replay key, and explicitly states ACCEPTED or NOT ACCEPTED with reason. |
| **Phase 6 — Update the map** | If ACCEPTED: move the row in `docs/cycle_9_and_10_improvements_todo.md` § Z.2/Z.3 to § Z.1. Update `AGENTS.md` registered status. Tick the checkbox in the file-completion checklist at the top of the TODO. If NOT ACCEPTED: keep STAGED and document why. | The TODO's checklist reflects reality. The cycle map is consistent with `AGENTS.md`. |

**Never skip phases. Never claim acceptance without phases 4 + 5 + 6
artefacts.**

---

## 4. Production server access — exactly what you need

From `AGENTS.md`:

- Host: `0.tcp.eu.ngrok.io`, port `27681`, user `bamby`
- Repo path: `/home/bamby/grad_project`
- SSH alias (Windows OpenSSH config): `prod-grad`
- Triton offline endpoint: HTTP `http://127.0.0.1:39100`, gRPC `127.0.0.1:39101`, metrics `:39102`
- Pinned Triton binary: `/home/bamby/services/triton_build_r2502/tritonserver/install/bin/tritonserver`
- Pinned Python venv: `/home/bamby/grad_project/backend/.venv/bin/python`
- Backend env authority: `backend/.env` (NOT repo-root `.env`)
- Postgres: port `55432` (non-standard; check `backend/.env`)

Common commands you will use:

```bash
# Push to prod
git push origin feature/phase7a-crop-frame-mode
ssh prod-grad "cd /home/bamby/grad_project && git fetch origin <branch> -q && git reset --hard origin/<branch> -q && git rev-parse HEAD"

# Apply optimized env (registers all the LPM / ensemble / behavior flags)
ssh prod-grad "cd /home/bamby/grad_project && bash tools/prod/prod_enable_parallel_flow.sh"

# Verify Triton is loading what we expect
ssh prod-grad 'curl -s -X POST http://127.0.0.1:39100/v2/repository/index | python3 -m json.tool'

# Run the canonical prod benchmark
ssh prod-grad "cd /home/bamby/grad_project && bash tools/prod/prod_run_parallel_flow_benchmark.sh \
  --replay-key cycle<N>-<short-name>-crop-frame-\$(date -u +%Y%m%dT%H%M%S) \
  --roi-behavior-input-size 320 \
  --timeout 7200"

# Probe the latest job
ssh prod-grad "cd /home/bamby/grad_project && bash tools/prod/prod_parallel_flow_probe.sh"

# Verify per-frame signal contract is intact
ssh prod-grad "cd /home/bamby/grad_project && bash tools/prod/prod_verify_per_frame_signals.sh"
```

Run `prod_run_parallel_flow_benchmark.sh` with `run_in_background: true`
and set up a `Monitor` task that watches DB job status. The bench takes
~17 min on `combined.mp4` for the current accepted baseline.

---

## 5. CI/CD checklist (what to touch every time)

The single workflow that matters for the optimization plan is
[`.github/workflows/inference-parallelization.yml`](../.github/workflows/inference-parallelization.yml).
Every code-touching commit must update three blocks of this file:

```yaml
# Block 1 — push paths
on:
  push:
    paths:
      - "backend/apps/pipeline/services/<NEW_MODULE>.py"
      - "backend/tests/unit/pipeline/<NEW_TEST_FILE>.py"
      - "docs/<NEW_RESULTS_FILE>.md"

# Block 2 — pull_request paths
  pull_request:
    paths:
      - "backend/apps/pipeline/services/<NEW_MODULE>.py"
      - "backend/tests/unit/pipeline/<NEW_TEST_FILE>.py"
      - "docs/<NEW_RESULTS_FILE>.md"

# Block 3 — pytest invocation
- name: Run inference parallelization tests
  run: |
    python -m pytest \
      ... existing entries ... \
      backend/tests/unit/pipeline/<NEW_TEST_FILE>.py \
      -q --tb=short
```

For every new `.env` flag, at least one unit test must monkeypatch it and
assert the dispatch path changes accordingly. Example:

```python
def test_compact_backend_bls_python_path_is_used(monkeypatch):
    monkeypatch.setenv("BEHAVIOR_COMPACT_BACKEND", "bls_python")
    # assert the dispatch chooses the BLS path
```

If you cannot articulate the test that exercises the new flag, you cannot
ship the flag.

---

## 6. What to work on (sequenced TODO entry points)

Read [`docs/cycle_9_and_10_improvements_todo.md`](cycle_9_and_10_improvements_todo.md)
§ D for the full sequence. Short version:

| Order | Task | Where the spec lives |
|---|---|---|
| 1 | **B.1 compact postprocessing** — preferably BLS only after verifying the prod Triton backend exists or rebuilding it intentionally | TODO § B.1 |
| 2 | **B.3 Step 2** — remeasure the dominant child against the Top-K baseline, then optimize only the measured dominant child | TODO § B.3 |
| 3 | **B.4** — bump `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES` 2 → 4 with RSS watch | TODO § B.4 |
| 4 | **B.4** — bump `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES` 2 → 4 with RSS watch | TODO § B.4 |
| 5 | **Cycle 10 LPM redesign** — capture pre-decode gaze probabilities or move LPM into compact postprocessing/BLS | TODO § C.2.4 |
| 6 | **Cycle 10b pose parallelization** | `docs/cycles_9_to_12_implementation_playbook.md` |

B.2.b exact server-side slice and B.2.c exact slice + Top-K are already
accepted; Top-K is now the production baseline. Do not repeat the rejected
`gaze2` standalone output-slice plan. The remaining Step 2 work must reduce
server-side child compute, improve GPU occupancy, or remove single-process
Python orchestration. Standalone B.2.a Top-K-only was not separately benchmarked
and is lower priority than the GPU-occupancy/server-side execution work because
B.2.c already proved response-byte trimming alone is not enough.

If you are not picking one of the above, you are scope-creeping. Don't.

---

## 7. What NOT to do

- **Do not** add new models (YOLOE, Depth Anything v2, anything else)
  until the SLA is hit or Cycle 13a's BLS architecture lands. The
  reasoning is in `docs/new_models_yoloe_depth_anything_v2_timing_decision.md`.
- **Do not** modify Triton model configs for `rtmpose_model`,
  `person_detector`, or `posture_model` as part of LPM work. They are
  out of scope by design (TODO § C.1 / spec § 2.2).
- **Do not** "mark accepted" based on local tests, parity probes, or
  hypotheses. The acceptance bar is the three prod artefacts (§ 2 rule 5).
- **Do not** combine cycles in one commit. One change → one benchmark →
  one cycle. The §B.1.a / B.1.b / B.1.c approaches are siblings of the
  same cycle (Cycle 9b) — each gets its own benchmark and a row in the
  comparison matrix, but together they constitute one cycle's selection
  decision.
- **Do not** bypass CI by adding code without updating
  `.github/workflows/inference-parallelization.yml`. Rule #4.
- **Do not** silently change the LPM scope. If you find yourself adding
  a `_LPM_AXIS_FOR_LABEL` entry for a non-gaze label, stop and re-read
  the spec § 2.
- **Do not** edit `master` or any other branch. The optimization work
  lives on `feature/phase7a-crop-frame-mode`. Push there and sync prod
  from there.

---

## 8. Communication style expectations

- **Trust but verify.** When you write "I implemented X", show the diff.
  When you write "tests pass", show the pytest output.
- **Explicit STAGED vs ACCEPTED.** If you ship code without a prod
  benchmark, the right word is STAGED. If you ship a prod benchmark and
  it passed every gate, the right word is ACCEPTED. There is no third
  status. (NOT ACCEPTED is the negative form of ACCEPTED — also valid.)
- **Cite prod evidence.** Every claim like "FPS improved by X%" must
  include the job ID, the replay key, and the bench summary path.
- **Honest about failures.** If a cycle does not hit its gate, document
  the post-mortem (mechanism: why didn't it work; lesson: what discipline
  rule must be added). Cycle 7 (redis cache projected −69 %, delivered
  −3.6 %) and Cycle 9 (ensemble projected ≥ 10 % Step 2 reduction,
  delivered +0.6 %) are the precedents. Document like they did.
- **No happy-path-only docs.** Every results doc must show the failure
  modes too (RSS regression possibilities, parity issues, etc.).

---

## 9. Self-check before claiming a cycle complete

Before you write "Cycle <N> ACCEPTED" anywhere:

1. [ ] Hypothesis section in `docs/crop_frame_optimization_execution.md`
       was written **before** the code? (Phase 2)
2. [ ] The hypothesis named which of the 3 levers it pulls? (Rule #2)
3. [ ] Unit tests cover the new behavior? (Phase 3)
4. [ ] `.github/workflows/inference-parallelization.yml` updated in the
       same commit as the code? (Rule #4)
5. [ ] Every new `.env` flag exercised by at least one unit test? (Rule #4)
6. [ ] Code pushed and prod synced to the same SHA? (Phase 4)
7. [ ] `tools/prod/prod_run_parallel_flow_benchmark.sh` ran end-to-end
       on `combined.mp4`? (Phase 4)
8. [ ] DB `status=completed` reached? Bench summary JSON exists?
       inference_audit JSON exists? GPU monitor CSV exists? (Phase 4)
9. [ ] Comparison row added to `docs/cycle_<N>_results.md` (or the
       relevant existing results doc) with before/after numbers and
       every correctness counter? (Phase 5)
10. [ ] § Z.1 row in `docs/cycle_9_and_10_improvements_todo.md` updated
        to reflect ACCEPTED? (Phase 6)
11. [ ] `AGENTS.md` registered status updated? (Phase 6)
12. [ ] File-completion checklist in
        `docs/cycle_9_and_10_improvements_todo.md` checkbox ticked?
        (Phase 6)

**If any box above is unchecked, the cycle is not complete.**

---

## 10. The current state of the world (snapshot — verify with `git log`)

- **Branch:** `feature/phase7a-crop-frame-mode`
- **Latest accepted baseline:** Cycle 9b B.2.c exact slice + Top-K, job
  `be4ba9ee-4786-48e9-8334-28feb237a1fb`, replay key
  `cycle9b-topk-crop-frame-20260602T041900`, `1022.952 s` total,
  4.429 FPS overall (DB completed)
- **Cycle 9 (Triton behavior ensemble):** NOT ACCEPTED — Step 2 wall +0.6 %
  failed the ≥ 10 % reduction gate. Job `c1651663-e08a-4e29-9ee3-fd0f09884b98`.
  Flag `TRITON_BEHAVIOR_ENSEMBLE` exists and is part of the accepted slice route,
  but the plain dense ensemble by itself is not accepted.
- **Cycle 9b B.2.b exact slice:** ACCEPTED — post-benchmark parity
  `max_abs_diff=0.0`, Step 2 wall `858.1 → 573.927 s`, correctness parity held.
- **Cycle 9b B.2.c exact slice + Top-K:** ACCEPTED WITH CAVEAT — decoded parity
  `failed_count=0`, `max_score_diff=0.0`, `max_box_diff=0.0`, Step 2 frame wall
  `573.927 → 540.399 s`, behavior RTT mean `91.470 → 84.865 ms`, output bytes
  `~6.85 → ~0.33 MB/frame`, correctness parity held. Caveat: average GPU
  utilization did not improve (`9.595 % → 9.3 %`), so the next cycle must target
  GPU occupancy / server-side execution / orchestration.
- **Cycle 9b B.3 Step 1 REMEASURED 2026-06-02:** controlled probes against the
  Top-K topology show dominant child is still `gaze_horizontal_model` (`18.79 ms`
  server delta) but the gap to the next slowest shrank from `+33 %` to `+15 %`
  (absolute `3.93 → 2.47 ms`). Ensemble RTT `63.59 ms` mean, server `30.13 ms`,
  orchestration overhead `~8.7 ms / call`, per-crop compute constant at
  `~0.94 ms/crop`. Per-child Step 2 ceiling is `~4 %` Step 2 wall reduction.
  Next Step 2 must compare B.3.b/d kernel-tune (low risk, ≤ 4 %) against
  Cycle 11 (320 → 256, ~10-13 %) per §E.6 multi-approach rule. Full evidence:
  `docs/cycle_9b_child_critical_path_remeasure_topk_results.md`.
- **Cycle 10 (Logical Path Matrix):** NOT ACCEPTED. The hook and telemetry table
  ran on prod, but contradiction counters stayed zero and attention boxes
  regressed. Production remains `LPM_ENABLED=0`.
- **TODO file:** [`docs/cycle_9_and_10_improvements_todo.md`](cycle_9_and_10_improvements_todo.md)
  is the single source of truth for what is still open. Section Z is the
  map of every cycle past, present, and future.

---

## 11. One more thing — read this before you start

The user (and the prior agent) hold this work to a high evidence bar. Your
job is to be **boring**:

- One change at a time.
- Measure before, measure after.
- Update CI in the same commit.
- Write the result doc.
- Tick the checkbox.
- Move on.

You will be tempted to skip a step "just this once". Don't. The Cycle 7
post-mortem is the precedent for what happens when projections aren't
measured: a hypothesis predicted −69 %, the actual was −3.6 %, and the
recovery was a re-baselined doc that admits the mistake. The Cycle 9
post-mortem is the precedent for what happens when the wrong lever is
pulled: working code that doesn't move the targeted metric, held back as
NOT ACCEPTED.

Be the agent that ships ACCEPTED cycles. Read § 1 mandatory list. Pick
the next task from § 6. Follow § 3 phases. Don't violate § 2 rules.

Welcome to the team.
