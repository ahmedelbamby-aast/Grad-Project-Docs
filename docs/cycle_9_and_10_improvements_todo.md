# Cycle 9 + Cycle 10 — Remaining Improvements (TODO)

**Status:** Consolidated TODO list. Nothing in this document is accepted or
flagged "done" until each improvement has its own production benchmark on
the Linux RTX 5090 server that demonstrates (a) the targeted metric
improvement and (b) zero correctness regression vs. the prior accepted
baseline (Cycle 8, job `d2de80a0`). See `AGENTS.md` re-affirmation and the
Cycle 9 precedent for what "not accepted" means in practice.

**Source documents this consolidates:**

- `docs/cycle_9_results.md` — Cycle 9 production outcome + post-mortem + the 5 continuation options
- `docs/logical_path_matrix_spec.md` — Cycle 10 LPM formal spec
- `docs/cycles_9_to_12_implementation_playbook.md` — original Cycle 9–12 plan
- `docs/triton_models_and_tensor_anatomy.md` — dense tensor inefficiency analysis
- `docs/runtime_sla_video_plus_5min.md` — SLA target (`total_wall ≤ video_duration + 5 min`)
- `AGENTS.md` — constitutional discipline

---

## A. Where we stand (numbers from prod, not estimates)

| State | Job | Total wall | Overall FPS | Status |
|---|---|---:|---:|---|
| Baseline (ROI-320 only) | `77650001` | 3 469 s (57.8 min) | 1.309 | superseded |
| Cycles 1–5 | `74ec0432` | 2 186 s (36.4 min) | 2.077 | ACCEPTED |
| Cycle 6 (pose chunking) | `a1a448b9` | 1 633 s (27.2 min) | 2.780 | ACCEPTED |
| Cycle 7 (redis cache) | `515fe118` | 1 582 s (26.4 min) | 2.870 | ACCEPTED (caveat) |
| Cycle 8 (embedding stage) | `d2de80a0` | 1 312 s (21.9 min) | 3.460 | **ACCEPTED** (latest accepted baseline) |
| Cycle 9 (behavior ensemble) | `c1651663` | 1 111 s (18.5 min) | 4.090 | **NOT ACCEPTED** — Step 2 wall +0.6 % failed the ≥ 10 % gate |
| Cycle 10 (LPM) | — | — | — | **STAGED** — code + tests landed, prod benchmark pending |

**SLA target (`combined.mp4`, 4 541 frames, 2 m 31 s):** total wall ≤ 7 m 31 s
= 451 s = **≥ 10.07 FPS overall**. Current accepted baseline (Cycle 8) is
**21.9 min** — gap is **14.4 min**.

---

## B. Cycle 9b — Five concrete improvements (NONE STARTED)

Cycle 9 (plain ensemble) reduced app-side request fragmentation but did not
reduce GPU-side child compute or dense output transfer, so Step 2 wall did
not move. The next ensemble-adjacent change MUST attack one of the five
levers below. Each is a separate STAGED candidate; the production benchmark
before/after is the only acceptance evidence.

### B.1 Option 1 — Server-side compact postprocessing (HIGHEST IMPACT)

**What:** Add a BLS Python backend (or a C++ custom backend) that consumes
the four dense behavior outputs **inside Triton** and emits only the
post-NMS / argmax decisions (class ID, confidence, optional bbox).

**Why this matters most:** every frame currently moves ~17.1 MB of dense
YOLO grid bytes from GPU → CPU → gRPC → Python, runs NMS 68 times
(17 crops × 4 models), then discards 99.99 % of it. See
`docs/triton_models_and_tensor_anatomy.md` for the byte math.

**Mechanism placement:**

- New `behavior_compact_ensemble` whose final step is a BLS Python backend
  at `triton_repository_cuda12/behavior_postproc/`.
- Triton 2.55's bundled `python_backend` stub is already present in our install.

**Expected gain:**

- Per-frame response shrinks from `(14 + 84 + 14 + 14) × 2100 × 4 B ≈ 17 MB`
  to roughly 32 B / crop / model = ~2 KB total.
- Removes the 68 dense-grid Python decodes from the per-frame hot path
  (`_decode_yolo_output0` calls).
- Step 2 wall reduction projected at ≥ 10 % — this is the first candidate
  with a realistic path to the SLA acceptance gate.

**Risk:** **High.** First BLS Python backend in this repo. Accuracy contract
is bit-identical *only if* server-side NMS settings match the Python-side
`_decode_yolo_output0` (score threshold, IoU threshold,
`TRITON_YOLO_MAX_DECODE_CANDIDATES`).

**Rollback:** `TRITON_BEHAVIOR_ENSEMBLE_COMPACT=0`. Falls back to the
existing flat ensemble or to the standalone 4-model path.

**Acceptance gate:**

1. Tensor parity probe: ≤ 1e-6 max abs diff vs. the existing Python NMS
   path on the same crop batch.
2. Per-class bbox count parity within 0.05 % of Cycle 8 baseline.
3. Step 2 wall reduction ≥ 10 % on `combined.mp4`.

### B.2 Option 2 — Fuse outputs before returning (top-K or class-trim)

**What:** Lighter-weight alternative to Option 1 — keep dense YOLO format
but trim the response to only the *needed* logits.

**Two sub-options:**

- **B.2a — Top-K anchor packing.** Add an ensemble step or tiny TensorRT op
  that returns only the top-K anchors per crop per model (K matches
  `TRITON_YOLO_MAX_DECODE_CANDIDATES=100`, set in Cycle 1–5). Output bytes
  drop ~21× per crop.
- **B.2b — Class-channel trim for `gaze_horizontal_model` specifically.**
  This one model emits `[84, 2100]` because it was exported with a
  COCO-80 head — the pipeline only consumes the gaze classes. Re-export
  with a 2-class head: `[6, 2100]` = 50 KB instead of 689 KB per crop, a
  ~14× shrink on the single largest output we have.

**Expected gain:** Less than Option 1 but with a much smaller engineering
footprint. B.2b alone could trim ~11 MB / frame of network bytes.

**Risk:** Medium. Top-K choice must match Python-side
`TRITON_YOLO_MAX_DECODE_CANDIDATES` exactly. B.2b is just a re-export of one
ONNX model with a narrower head — but accuracy parity probe is mandatory.

**Acceptance gate:** identical to B.1 (parity, bbox parity, Step 2 wall
reduction).

### B.3 Option 3 — Reduce the child model critical path

**What:** The ensemble's wall time is `max(posture, gaze_h, gaze_v, gaze_d)`.
Cycle 9 ensemble RTT averaged 107.9 ms vs. the per-model spread of 143–168 ms
in Cycle 8 standalone — but those numbers tell us nothing about *which*
child dominates inside the ensemble.

**Steps:**

1. Add server-side per-child timing telemetry to the ensemble (Triton
   exposes `inference_stats.compute_infer.ns` per model — extract per-child
   ns from a side-by-side run).
2. Whichever sub-model dominates `max(…)`, optimize that one specifically:
   - TensorRT precision tuning (FP16 → INT8 on the dominant child only).
   - Engine batch-profile mismatch (each sub-model has its own preferred
     batch shape; align with `behavior_ensemble.max_batch_size=32`).
   - Output channel pruning if any single sub-model output is wider than
     needed (overlaps with B.2b).

**Expected gain:** Lifts the floor of `max(…)` for behavior dispatch. The
prize is bounded by how skewed the children are today.

**Risk:** Medium. Precision change (INT8 calibration) requires an accuracy
parity gate.

**Acceptance gate:** parity probe + Step 2 wall reduction ≥ 5 % vs. Cycle 9.

### B.4 Option 4 — Batch larger at ensemble level

**What:** Cycle 9 batch histograms already centered at 32 (the engine cap)
for behavior crops in single frames. Behavior crops could be coalesced
across **2 frames at once** so the ensemble sees ~34 crops / call instead
of 17.

**Steps:**

1. Raise `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES` from 2 → 4 (Cycle 5 set it
   to 2 to avoid the 100-GiB RSS spike from the earlier all-frames path —
   Cycle 8 trim + `TRITON_NUMPY_OUTPUTS=1` together bound RSS at ~2 GiB
   today).
2. Keep pose chunking at 16 to avoid regressing Cycle 6.
3. RSS watch on prod is mandatory.

**Expected gain:** Marginal. **Secondary candidate** unless Option 3
analysis shows the engine is genuinely under-utilized.

**Risk:** Medium. The prior 100-GiB RSS spike was at unbounded
`MAX_FRAMES`; we have the Cycle 8 trim infrastructure to bound it now, but
it must be measured.

**Acceptance gate:** worker RSS stays under 4 GiB AND Step 2 wall
improvement.

### B.5 Option 5 — Discipline rule (not implementation, governance)

**What:** Cycle 9 proved that **call-count reduction without GPU-work
reduction is insufficient**. This is recorded as an explicit discipline rule
in `docs/cycle_9_results.md` and re-affirmed here:

> Any new optimization-cycle hypothesis must name **which** of these three
> levers it pulls before any code is written:
>
> 1. GPU-side child compute (Options 3, Cycle 11 smaller input),
> 2. dense output bytes returned to Python (Options 1, 2),
> 3. frame-level Python orchestration outside the single-process ordering
>    (pose parallelization Cycle 10b, video sharding for a future cycle).

This is **not** a code change — it's a constitution rule that gates future
hypotheses. Cycle 9b candidates B.1–B.4 each name their lever explicitly.

---

## C. Cycle 10 — Logical Path Matrix (LPM) (STAGED, NOT WIRED)

### C.1 What landed already

- `backend/apps/pipeline/services/logical_path_matrix.py` — pure-function
  constraint solver with `LpmConfig`, `LpmFrameInput`, `LpmState`,
  `LpmBatchMetrics`, `apply(...)`, `estimate_head_yaw_deg(...)`,
  `apply_to_detection_boxes(...)`.
- 28 unit tests passing covering all of C1–C4, the disabled-flag identity,
  the exclusion contract, the RTMPose head-yaw helper, batch metrics, and
  the integration helper.
- All settings registered in `backend/config/settings/base.py`
  (`LPM_ENABLED` default `0`).
- All prod env knobs registered in `tools/prod/prod_enable_parallel_flow.sh`
  at value `0`.
- CI workflow gates the new module + tests.
- Formal spec at `docs/logical_path_matrix_spec.md`.

### C.2 What is NOT done yet (this is the TODO list)

#### C.2.1 Wire the LPM hook into `tasks.py`

**Where:** `backend/apps/video_analysis/tasks.py` inside
`_run_crop_behaviour_for_items()`, **after** the 4-model dispatch returns
and `_decode_yolo_output0` has produced the per-crop DetectionBox list,
**before** the boxes are appended to `fd.boxes`.

**What:**

```python
if settings.LPM_ENABLED:
    from apps.pipeline.services.logical_path_matrix import (
        apply_to_detection_boxes,
        LpmConfig,
    )
    new_boxes, lpm_priors_state, lpm_metrics = apply_to_detection_boxes(
        fd.boxes_pending_lpm,
        prior_states_by_track=lpm_state_cache,        # per-job dict
        pose_keypoints_by_track=pose_kpts_for_frame,  # populated when rtmpose ran
        cfg=LpmConfig.from_settings(),
    )
    fd.boxes_pending_lpm = new_boxes
    lpm_state_cache.update(lpm_priors_state)
    record_lpm_telemetry(job_id, frame_index, lpm_metrics)
```

**Constraints on this wiring:**

- `lpm_state_cache` is **per-job, per-Celery-task** — not module-level,
  not Redis. Survives only the lifetime of one offline job.
- `pose_kpts_for_frame` must be the *same-frame* RTMPose output. If pose
  for the current frame isn't available yet (race condition), pass `None`
  for that person — C4 then defaults to no-pose-coupling.
- The hook MUST NOT execute when `LPM_ENABLED=0` (disabled-flag identity
  property — already tested at the helper level; needs an integration
  test at the tasks.py call-site level).

**Acceptance:** integration unit test that exercises the wiring with a
fake `fd` object and verifies (a) disabled-flag identity, (b)
detection_count parity when LPM is enabled but inputs are unambiguous.

#### C.2.2 Telemetry table for LPM events

**Where:** new Django model + migration.

**What:** the table spec in `docs/logical_path_matrix_spec.md` §8:

```sql
CREATE TABLE telemetry_lpm_events (
    id BIGSERIAL PRIMARY KEY,
    session_id UUID REFERENCES telemetry_sessions(id),
    frame_index INTEGER NOT NULL,
    person_count INTEGER NOT NULL,
    eliminated_contradictions INTEGER NOT NULL,
    violations_c1 INTEGER NOT NULL,
    violations_c2 INTEGER NOT NULL,
    violations_c3 INTEGER NOT NULL,
    violations_c4 INTEGER NOT NULL,
    latency_ms FLOAT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
CREATE INDEX telemetry_lpm_sess_frame_idx
    ON telemetry_lpm_events (session_id, frame_index);
```

**Why:** without this, the §10.4 acceptance gate (contradiction reduction
rate, gaze-flip-rate drop) can only be computed by log scraping. The table
is the canonical source for the before/after benchmark math.

**Implementation:** model in `backend/apps/telemetry/models.py`, write
helper in `backend/apps/telemetry/writer.py` (mirrors
`_bulk_insert_model_calls` from Cycle 7), bulk-insert from
`record_lpm_telemetry`.

#### C.2.3 Production benchmark — the only thing that turns STAGED into ACCEPTED

**Run:**

```bash
# Flip flag for one bench run
ssh prod-grad 'cd /home/bamby/grad_project && \
  LPM_ENABLED=1 bash tools/prod/prod_run_parallel_flow_benchmark.sh \
    --replay-key cycle10-lpm-crop-frame-$(date -u +%Y%m%dT%H%M%S) \
    --roi-behavior-input-size 320 --timeout 7200'
```

**Acceptance gates (all must pass — copy of `docs/logical_path_matrix_spec.md` §10):**

| Gate | Threshold | Source |
|---|---|---|
| Frame rows count | unchanged vs. Cycle 8 baseline | DB |
| Detection rows | ±0.05 % | DB |
| Per-class bbox counts (`attention/hand/person/sitting`) | ±0.05 % | DB |
| RTMPose `pose_record_count` | ±0.1 % | inference_audit |
| StudentTrack count | unchanged | DB |
| FrameEmbedding count | ±0.1 % | DB |
| **C3 violation count post-LPM** | **= 0** | `telemetry_lpm_events` |
| C1 violations + `eliminated_contradictions` | non-zero | `telemetry_lpm_events` |
| DB-completed FPS | ≥ 4.09 (Cycle 9 baseline) | bench summary |
| Step 2 wall | ≤ 858.1 s × 1.01 | inference_audit |
| Per-track gaze-flip rate | **drop ≥ 25 %** vs. LPM-off rerun | derived from Detection + StudentTrack |
| LPM identity property | LPM-off rerun on same SHA matches Cycle 9 numbers within noise | bench summary |

If any gate fails, LPM is **REJECTED** or **NEEDS-MORE-DATA**, and
`LPM_ENABLED` stays `0` in production.

#### C.2.4 Phase 2 — migrate the math to a Triton BLS Python backend

**When:** after C.2.3 lands and Phase 1 (Python-side) is ACCEPTED. Not
before — we need production evidence the math actually reduces
contradictions before paying the BLS deployment cost.

**What:**

- Move the pure `apply(...)` function into a Triton Python backend at
  `triton_repository_cuda12/logical_path_matrix/1/model.py`.
- The backend takes the 4 behavior outputs + an optional pose-keypoint
  input and returns compact `(h, v, d, confidence)` tuples per person.
- Combines naturally with Cycle 9b Option 1 (compact postprocessing) — the
  LPM step is the final ensemble step that produces the compact tuple.

**Why deferred:** BLS Python backend is a first-of-its-kind deployment in
this repo. Phase 1 lets us measure the math correctness using existing
deployment paths; Phase 2 lets us measure the network/CPU savings.

**Acceptance:** identical correctness gates to C.2.3 + Step 2 wall
reduction ≥ 5 % vs. Phase 1 (the wall improvement must come from
eliminated dense output transfer, not the LPM math which is essentially
free).

#### C.2.5 Open spec issues to resolve before C.2.3

- **C4 head-yaw sign convention.** Spec says "positive = head turned to its
  RIGHT (toward the right side of the frame from the viewer)". The
  estimator in `logical_path_matrix.py:estimate_head_yaw_deg` follows that
  convention. The wiring in C.2.1 must pass keypoints from RTMPose in
  pixel-coordinate space (not normalized). If RTMPose output is already
  normalized in the offline pipeline, the wiring must denormalize first OR
  the estimator must be made scale-invariant (it already is — it works on
  ratios, not absolute pixel distances; this just needs a confirmation
  test on real prod data).
- **Per-track state cache lifecycle.** Spec assumes `prior_state` is the
  *immediately previous frame*. If frames are processed concurrently (after
  Cycle 10b pose parallelization lands), the prior-state cache needs a
  per-track sequencing guarantee. For now, the offline pipeline processes
  frames in order so this is satisfied.
- **C4 yaw threshold tuning.** Default `LPM_THETA_POSE_THRESHOLD_DEG=35.0`
  was chosen as a sane initial value. The acceptance run should record yaw
  histograms so we can re-tune in a follow-up cycle.

---

## D. Sequencing recommendation

| Order | Improvement | Cycle | Notes |
|---|---|---|---|
| 1 | **C.2.1 LPM wiring** | 10 (Phase 1) | low risk, code already exists, prerequisite for C.2.3 |
| 2 | **C.2.2 LPM telemetry table** | 10 (Phase 1) | small DB migration, prerequisite for C.2.3 |
| 3 | **C.2.3 LPM Phase 1 prod benchmark** | 10 (Phase 1) | flip `LPM_ENABLED=1` for one bench, gate decides ACCEPTED / REJECTED |
| 4 | **B.2b `gaze_horizontal` re-export** | 9b | independent of LPM, smallest engineering footprint, single largest output trim |
| 5 | **B.3 child critical-path analysis** | 9b | server-side timing first, then targeted optimization of the dominant child |
| 6 | **B.1 server-side compact postprocessing (BLS)** | 9b | highest impact but highest risk — depends on B.2b learnings |
| 7 | **C.2.4 LPM Phase 2 (move math into BLS)** | 10 (Phase 2) | combines with B.1 — both land in the same BLS Python backend |
| 8 | **B.2a top-K anchor packing** | 9b | only if B.1 timeline slips |
| 9 | **B.4 larger ensemble batches** | 9b | secondary — measure after B.1 / B.3 |

Steps 1–3 unblock the LPM acceptance gate. Steps 4–6 are the path to the
≥ 10 % Step 2 wall reduction that Cycle 9 failed to achieve. Step 7 is the
architectural cleanup that fuses LPM into the same BLS layer as compact
postprocessing.

---

## E. Hard rules (re-stated)

1. **No "accepted" without a production benchmark on the Linux RTX 5090
   server.** Code review, parity probes, and local tests are necessary but
   never sufficient. Cycle 9 is the precedent.
2. **No optimization-cycle hypothesis without naming the lever it pulls**
   (Option 5 / B.5). Future hypotheses must say "this attacks GPU child
   compute" OR "this attacks dense output bytes" OR "this attacks
   single-process Python orchestration" — and the proposed change must
   actually move that lever.
3. **LPM scope is HARD-RESTRICTED.** Code that mutates RTMPose / person_detector
   / sitting_standing outputs must fail loudly via the structural
   exclusion tests in `test_logical_path_matrix.py`.

---

## F. Reference

- `AGENTS.md` — registered status of every cycle
- `docs/cycle_9_results.md` — full Cycle 9 post-mortem
- `docs/logical_path_matrix_spec.md` — full LPM formal spec including §10 acceptance gates
- `docs/triton_models_and_tensor_anatomy.md` — why "dense tensors" are the bottleneck
- `docs/runtime_sla_video_plus_5min.md` — SLA contract
- `docs/cycles_9_to_12_implementation_playbook.md` — full Cycle 9–12 roadmap
- `backend/apps/pipeline/services/logical_path_matrix.py` — LPM implementation
- `backend/tests/unit/pipeline/test_logical_path_matrix.py` — 28 LPM unit tests
