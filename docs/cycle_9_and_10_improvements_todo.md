# Cycle 9 + Cycle 10 — Remaining Improvements (TODO)

**Status:** Consolidated TODO list. Nothing in this document is accepted or
flagged "done" until each improvement has its own production benchmark on
the Linux RTX 5090 server that demonstrates (a) the targeted metric
improvement and (b) zero correctness regression vs. the prior accepted
baseline (Cycle 8, job `d2de80a0`). See `AGENTS.md` re-affirmation and the
Cycle 9 precedent for what "not accepted" means in practice.

> **For the next agent reading this file first:** this TODO covers **only**
> Cycle 9b and Cycle 10. The full map of every cycle (accepted, not
> accepted, staged, planned, deferred) is in **§ Z below**. Before starting
> any item here, scan § Z to see whether a more-recent cycle has changed
> the baseline these improvements are computed against.

## File-completion checklist (read this first)

This file is "done" only when **every** unchecked box below has been ticked
in a follow-up commit. Until then, future agents must assume the file is
in-progress and use the per-item acceptance gates inside it.

- [ ] B.1 Compact-postprocessing backend implemented (one of the approaches in §B.1) and prod-benchmarked
- [ ] B.2 Output-fusion approach selected from the comparison matrix in §B.2 and prod-benchmarked
- [ ] B.3 Child critical-path measurement landed and the dominant-child optimization landed
- [ ] B.4 Larger-batch knob measured (accept or reject)
- [ ] C.2.1 LPM hook wired into `_run_crop_behaviour_for_items`
- [ ] C.2.2 `telemetry_lpm_events` table + migration + writer landed
- [ ] C.2.3 LPM Phase 1 prod benchmark passes all §10 gates → ACCEPTED
- [ ] C.2.4 LPM Phase 2 (BLS migration) landed and benchmarked
- [ ] C.2.5 Open spec issues resolved

Each item below lists its own acceptance gate; flip the checkbox above
**only** when that gate is met with prod evidence.

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

### B.1 Server-side compact postprocessing (HIGHEST IMPACT — multi-approach)

**What:** Move the post-NMS step *inside* Triton so the dense YOLO grids
never cross gRPC. Same problem, two implementation strategies — both must
be built and benchmarked side-by-side before one is selected.

**Why this matters most:** every frame currently moves ~17.1 MB of dense
YOLO grid bytes from GPU → CPU → gRPC → Python, runs NMS 68 times
(17 crops × 4 models), then discards 99.99 % of it. See
`docs/triton_models_and_tensor_anatomy.md` for the byte math.

#### B.1 approaches to test

| ID | Approach | Backend type | Pros | Cons |
|---|---|---|---|---|
| **B.1.a** | **BLS Python backend** at `triton_repository_cuda12/behavior_postproc/` whose final ensemble step runs the existing `_decode_yolo_output0` math server-side | Triton 2.55 bundled `python_backend` | Lowest engineering cost — reuse Python NMS verbatim; debuggable in plain Python; first foothold for the Cycle 10 LPM Phase 2 migration | Adds a Python-interpreter hop inside Triton per call (~5–15 ms/call overhead); harder to tune at high throughput |
| **B.1.b** | **C++ / custom-backend NMS** compiled as a Triton backend shared object | Triton `custom_backend` | Lower per-call overhead (~0.1–0.5 ms); no GIL; closer to TRT NMS plugin performance | High engineering cost — C++ build pipeline + per-engine signatures; not reusable for LPM |
| **B.1.c** | **TRT EfficientNMS plugin** baked into each behavior engine via a re-export step (NMS happens inside the engine, not as a separate Triton step) | TensorRT plugin | Fastest path — NMS runs on GPU with the rest of the engine; no extra Triton scheduling | Each behavior engine must be re-exported with the plugin; output contract changes shape (compact `[num_dets, 6]` instead of `[C, 2100]`); accuracy parity must be re-validated per engine |

#### B.1 selection mechanism (.env-controlled)

```env
# pick exactly one — empty / unset = legacy dense path
BEHAVIOR_COMPACT_BACKEND=          # "" | "bls_python" | "cpp_custom" | "trt_plugin"
BEHAVIOR_COMPACT_SCORE_THRESHOLD=0.25
BEHAVIOR_COMPACT_IOU_THRESHOLD=0.45
BEHAVIOR_COMPACT_MAX_DETECTIONS=100
```

The client-side `_decode_yolo_output0` reads `BEHAVIOR_COMPACT_BACKEND` and
either parses dense grids (when empty) or compact `[num_dets, 6]` tuples
(when one of the three approaches is enabled). Both paths must coexist so
the rollback to legacy is a single env flip.

#### B.1 measurement matrix (one prod bench per approach, same `combined.mp4`)

Fill the table in a new file `docs/cycle_9b_compact_postproc_results.md`.
Numbers must come from prod, not local — see § E hard rule #1.

| Metric | Legacy (Cycle 9 baseline) | B.1.a BLS Python | B.1.b C++ custom | B.1.c TRT plugin |
|---|---:|---:|---:|---:|
| Step 2 wall (s) | 858.1 | ? | ? | ? |
| Step 2 wall Δ vs legacy | — | ? % | ? % | ? % |
| Total DB-completed elapsed (s) | 1 110.7 | ? | ? | ? |
| Overall FPS (DB completed) | 4.09 | ? | ? | ? |
| Avg GPU util (%) | 9.36 | ? | ? | ? |
| Behavior dense bytes / frame | 17.1 MB | ~2 KB | ~2 KB | ~2 KB |
| Avg behavior RTT (ms) | 107.9 | ? | ? | ? |
| p95 behavior RTT (ms) | 173.9 | ? | ? | ? |
| Per-class bbox parity (Δ %) | 0 | ≤ 0.05 ✓ | ≤ 0.05 ✓ | ≤ 0.05 ✓ |
| Engineering effort (person-days) | — | ~1 | ~5 | ~3 |
| Production deploy risk | — | medium | high | medium-high |
| Rollback complexity | trivial | env flip | env flip + binary swap | env flip + engine swap |

**Selection rule:** pick the approach that delivers ≥ 10 % Step 2 wall
reduction with the **lowest** engineering effort and rollback complexity.
Ties go to the approach that also unlocks Cycle 10 LPM Phase 2 (B.1.a).

**Risk:** see per-approach Cons column above.

**Rollback (all three):** `BEHAVIOR_COMPACT_BACKEND=` (empty). The legacy
dense-grid path stays compiled in.

**Acceptance gate (any approach):**

1. Tensor parity probe: ≤ 1e-6 max abs diff between compact-output
   detections and the existing Python-NMS detections on the same crop
   batch.
2. Per-class bbox count parity within 0.05 % of Cycle 8 baseline.
3. Step 2 wall reduction ≥ 10 % on `combined.mp4`.
4. All three approaches' bench numbers recorded in
   `docs/cycle_9b_compact_postproc_results.md` even if only one ships —
   so future agents have the evidence for the tradeoff.

### B.2 Output fusion — class-trim and top-K (multi-approach, both built)

**What:** Lighter-weight alternative to B.1 — keep dense YOLO format but
trim the response to only the *needed* logits. Smaller engineering
footprint than the full BLS path.

#### B.2 approaches to test

| ID | Approach | What changes | Bytes saved per crop |
|---|---|---|---:|
| **B.2.a** | **Top-K anchor packing** — extra ensemble step (or tiny TensorRT op) returns only the top-K anchors per crop per model with `K = TRITON_YOLO_MAX_DECODE_CANDIDATES = 100` | Output shape per model becomes `[C, K]` instead of `[C, 2100]` — `100/2100 ≈ 4.8 %` of original | ~95 % per model |
| **B.2.b** | **Class-channel trim for `gaze_horizontal_model`** — re-export ONNX with a 2-class head instead of the inherited COCO-80 head | Output shape changes from `[84, 2100]` to `[6, 2100]` | ~93 % on this one model (689 KB → ~50 KB per crop = **~11 MB / frame** saved) |
| **B.2.c** | **Combine B.2.a + B.2.b** — top-K packing AND a 2-class re-export of `gaze_horizontal_model` (the other three behavior models keep their 10-class head) | Stacked savings | ~99 % per crop on `gaze_horizontal_model`, ~95 % on the others |

#### B.2 selection mechanism (.env-controlled)

```env
# Each approach independently toggleable
TRITON_BEHAVIOR_TOP_K_ENABLED=0           # B.2.a — top-K anchor packing
TRITON_BEHAVIOR_TOP_K_VALUE=100            # K must equal the client-side cap to preserve parity

GAZE_HORIZONTAL_HEAD_VARIANT=coco80        # B.2.b — "coco80" (legacy 84-channel) | "gaze2" (re-exported 6-channel)
```

Setting both to non-default values activates **B.2.c**. The client-side
`_decode_yolo_output0` must read `GAZE_HORIZONTAL_HEAD_VARIANT` to know
whether to expect 84 or 6 channels on that specific model.

#### B.2 measurement matrix (one prod bench per row)

Record in `docs/cycle_9b_output_fusion_results.md`. All three rows must be
benchmarked even if only one ships.

| Metric | Legacy (Cycle 9) | B.2.a top-K only | B.2.b 2-class gaze_h only | B.2.c both |
|---|---:|---:|---:|---:|
| Step 2 wall (s) | 858.1 | ? | ? | ? |
| Step 2 wall Δ vs legacy | — | ? % | ? % | ? % |
| Total DB-completed elapsed (s) | 1 110.7 | ? | ? | ? |
| Overall FPS (DB completed) | 4.09 | ? | ? | ? |
| Behavior dense bytes / frame | 17.1 MB | ? MB | ~6.1 MB | ? MB |
| Avg behavior RTT (ms) | 107.9 | ? | ? | ? |
| Per-class bbox parity (Δ %) | 0 | ≤ 0.05 ✓ | ≤ 0.05 ✓ | ≤ 0.05 ✓ |
| Engineering effort (person-days) | — | ~1.5 | ~2 (re-export + accuracy probe) | ~3 |
| Production deploy risk | — | low | medium (engine swap) | medium |
| Rollback complexity | trivial | env flip | env flip + engine swap | env flip + engine swap |

**Selection rule:** Pick the approach with the highest Step 2 wall
reduction at acceptable risk. If B.2.c materially beats B.2.b alone,
ship B.2.c; otherwise the simpler B.2.b is the default.

**B.2 vs B.1 — should both be built?** Yes. B.2 is a lower-risk fallback
if B.1 slips. They are not mutually exclusive — B.1 (server-side NMS) and
B.2.b (smaller `gaze_horizontal_model` output) compose: a BLS backend
operating on a 6-channel `gaze_horizontal` output is even cheaper than one
operating on 84-channel.

**Risk:** Medium. Top-K `K` value must match
`TRITON_YOLO_MAX_DECODE_CANDIDATES` exactly. B.2.b re-export must pass an
accuracy parity probe (same gates as Cycle 11 smaller-input change).

**Rollback:** flip both env flags back to defaults.

**Acceptance gate (any approach):** identical to B.1.

### B.3 Reduce the child model critical path (measure-first, multi-approach)

**What:** The ensemble's wall time is `max(posture, gaze_h, gaze_v, gaze_d)`.
Cycle 9 ensemble RTT averaged 107.9 ms vs. the per-model spread of 143–168 ms
in Cycle 8 standalone — but those numbers tell us nothing about *which*
child dominates inside the ensemble.

#### B.3 Step 1 — measurement (always do this first)

Pull server-side per-child timing from Triton stats:

```bash
ssh prod-grad 'for m in posture_model gaze_horizontal_model \
                     gaze_vertical_model gaze_depth_model; do
  curl -s http://127.0.0.1:39100/v2/models/$m/stats \
   | python3 -c "import json,sys;d=json.load(sys.stdin)[\"model_stats\"][0]; \
       s=d[\"inference_stats\"][\"success\"]; \
       print(f\"{d[\\\"name\\\"]:>25}  count={s[\\\"count\\\"]} \
       avg_ms={s[\\\"ns\\\"]/max(1,s[\\\"count\\\"])/1e6:.3f}\")"
done'
```

Record the per-child average and p95 in `docs/cycle_9b_child_critical_path_results.md`.
**Do NOT proceed to B.3 Step 2 until this measurement exists.**

#### B.3 Step 2 — approaches to test (only on the dominant child)

| ID | Approach | Mechanism | Risk |
|---|---|---|---|
| **B.3.a** | **FP16 → INT8 precision** | Re-export the dominant child as INT8 with calibration data from `combined.mp4` | Medium — needs accuracy parity gate |
| **B.3.b** | **Batch-profile alignment** | Each child engine has its own `min/opt/max` shape profile. Align dominant child's `opt` shape with `behavior_ensemble.max_batch_size=32` | Low — engine rebuild, no precision change |
| **B.3.c** | **Output-channel pruning on dominant child** | If the dominant child is `gaze_horizontal_model`, this overlaps with B.2.b. For other dominant children, prune the unused class channels in their head | Medium — re-export with narrower head |
| **B.3.d** | **Operator-level kernel selection** | TensorRT builder tactic tuning (kernel-selection hints, layer fusion overrides) for the dominant child | Low — pure builder-config change |

#### B.3 selection mechanism (.env-controlled)

```env
# Activated only when the dominant child has been measured.
# Each child has its own engine variant flag so future cycles can stack them.
POSTURE_ENGINE_VARIANT=fp16              # "fp16" (default) | "int8" | "fp16_pruned"
GAZE_HORIZONTAL_ENGINE_VARIANT=fp16
GAZE_VERTICAL_ENGINE_VARIANT=fp16
GAZE_DEPTH_ENGINE_VARIANT=fp16
```

A new helper `tools/prod/prod_enable_child_engine_variant.sh` ships the
matching `.plan` files and reloads Triton.

#### B.3 measurement matrix

Record in `docs/cycle_9b_child_critical_path_results.md`. Headers are
filled in after Step 1 identifies the dominant child — the table compares
the chosen approach against the FP16 baseline for that **specific** model.

| Metric | FP16 baseline (Cycle 9 child) | B.3.a INT8 | B.3.b batch-profile aligned | B.3.c output-pruned | B.3.d kernel-tuned |
|---|---:|---:|---:|---:|---:|
| Server compute_infer (ms) per call | ? | ? | ? | ? | ? |
| Ensemble `max(…)` RTT (ms) | ? | ? | ? | ? | ? |
| Step 2 wall (s) | 858.1 | ? | ? | ? | ? |
| Step 2 Δ vs FP16 | — | ? % | ? % | ? % | ? % |
| Per-class bbox parity Δ | 0 | ≤ 0.05 ✓ | 0 (no precision change) | ≤ 0.05 ✓ | 0 (no precision change) |
| Accuracy parity gate | n/a | **mandatory** | n/a | **mandatory** | n/a |
| Engineering effort (person-days) | — | ~3 (calibration set) | ~1 | ~2 | ~1.5 |

**Selection rule:** prefer the approach with the largest ensemble `max(…)`
reduction that passes accuracy parity. B.3.b and B.3.d are tried first
(no precision change → no accuracy risk). Move to B.3.a/c only if those
don't deliver ≥ 5 % Step 2 wall reduction.

**Risk:** B.3.a INT8 is the highest — calibration drift can introduce
silent accuracy regressions. The acceptance gate's per-class bbox parity
catches this only at the offline-pipeline level; an explicit per-image
calibration set comparison is mandatory before any INT8 child ships.

**Acceptance gate:** Step 2 wall reduction ≥ 5 % vs. Cycle 9 AND per-class
bbox parity within 0.05 % AND, for B.3.a / B.3.c, an explicit accuracy
parity probe on a held-out calibration set.

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

---

## Z. Map of ALL cycles — past, present, future

> **For agents finding this file first:** this section is the single
> entry point for *every* optimization cycle. If you don't see an item
> in this table, it does not exist in this repo. If you see an item
> marked ACCEPTED you may treat its measured numbers as the baseline
> against which your next cycle is measured. If you see an item marked
> NOT ACCEPTED / STAGED / PLANNED you must read its doc before assuming
> anything about its behavior.

### Z.1 Cycles that have completed (have prod-benchmark evidence)

| # | Title | Status | Job ID | Result | Primary docs |
|---|---|---|---|---|---|
| Cycles 1–5 | Bundle: telemetry writer fix + concurrency knobs + concat memo + trim amortization | **ACCEPTED 2026-06-01** | `74ec0432-995c-487e-9d77-1048ec109fb1` | Step 2 FPS 2.09 → 5.14, overall FPS 1.31 → 2.08 | `docs/production_inference_benchmark.md` §11, `docs/crop_frame_optimization_execution.md` Cycles 1–5, `AGENTS.md` |
| Cycle 6 | Pose dispatch chunking (rtmpose batch>16 fix) | **ACCEPTED 2026-06-01** | `a1a448b9-474f-4dea-942b-3288bcae6900` | Pose wall 12 m 13 s → 3 m 42 s (−69.7 %), overall FPS 2.08 → 2.78 | `docs/production_inference_benchmark.md` §12, Cycle 6 section in `docs/crop_frame_optimization_execution.md` |
| Cycle 7 | Redis client caching | **ACCEPTED 2026-06-01 (with caveat)** | `515fe118-6009-4776-916d-6473fbf31ed7` | Embedding wall 467 → 451 s (−3.6 %), overall FPS 2.78 → 2.87. Hypothesis projected −69 %; only −3.6 % delivered (redis-py 5.x lazy pool was much cheaper than assumed) | `docs/production_inference_benchmark.md` §13 |
| Cycle 8 | Embedding stage attack: track-level reuse + lazy cv2 + bulk_create | **ACCEPTED 2026-06-01** | `d2de80a0-31b7-4a47-b9f1-d2e2156ea3a8` | Embedding wall 451 → 174 s (−61.5 %), overall FPS 2.87 → 3.46 | `docs/production_inference_benchmark.md` §14, Cycle 8 section in `docs/crop_frame_optimization_execution.md` |
| Cycle 9 | Triton behavior ensemble (4 → 1 gRPC call per frame) | **NOT ACCEPTED 2026-06-01** | `c1651663-e08a-4e29-9ee3-fd0f09884b98` | App calls −75 %, behavior RTT 143–168 → 107.9 ms, overall FPS 3.46 → 4.09 (+18 %) — but **Step 2 wall 852.8 → 858.1 s failed the ≥ 10 % reduction gate**. Ensemble kept available under `TRITON_BEHAVIOR_ENSEMBLE=1` but not on the SLA path | `docs/cycle_9_results.md` (full post-mortem), `docs/cycle_9_investigation.md` |

### Z.2 Cycles staged or in progress (code exists, awaiting prod evidence)

| # | Title | Status | What is missing | Primary docs |
|---|---|---|---|---|
| **Cycle 9b** | Five concrete continuation options (compact postprocessing, output fusion, child critical-path, larger ensemble batches, discipline rule) | **PLANNED, NOT STARTED** | **This file, §§B.1–B.5.** No code yet; each option has its own multi-approach comparison matrix. | This file, `docs/cycle_9_results.md` |
| **Cycle 10** | Logical Path Matrix (LPM) — server-side behavioral constraint layer | **STAGED, NOT WIRED** | Math + 28 unit tests + CI gate + spec are landed. **Wiring into `tasks.py` (§C.2.1), telemetry table (§C.2.2), and the prod benchmark (§C.2.3) are NOT done.** | **This file, §§C.1–C.2.5**, `docs/logical_path_matrix_spec.md` |

### Z.3 Cycles planned but not yet started (from the 9–12 playbook)

| # | Title | Status | Projected gain | Primary docs |
|---|---|---|---:|---|
| **Cycle 10b** | Pose parallelization across frames (originally Cycle 10 in the playbook — slot taken by LPM; pose work moved here) | **PLANNED** | Pose wall 220 → ~80 s | `docs/cycles_9_to_12_implementation_playbook.md` §3 |
| **Cycle 11** | Behavior input 320 → 256 with engine rebuild + accuracy parity gate | **PLANNED** | Step 2 wall ~−13 %, overall FPS +10 % | `docs/cycles_9_to_12_implementation_playbook.md` §4 |
| **Cycle 12** | Parallel render writers + PostgreSQL `COPY FROM` for embeddings | **PLANNED** | ~20 s saved | `docs/cycles_9_to_12_implementation_playbook.md` §5 |
| **Cycle 13a** | BLS server-side fan-out (the architectural change — combines with §B.1 in this file) | **PLANNED** | 5 FPS → 8–10 FPS, lands at SLA boundary | `docs/cycles_9_to_12_implementation_playbook.md` §6 |
| **Cycle 13b** | Multi-process video sharding (4 Celery workers, each takes a quarter of the video, then a stitch step) | **PLANNED, RISKY** | 5 FPS → 15–18 FPS (tracking stitch is the hard part) | `docs/cycles_9_to_12_implementation_playbook.md` §6 |

### Z.4 Deferred decisions / out-of-scope here

| Topic | Status | Rationale | Doc |
|---|---|---|---|
| **YOLOE integration** | **DEFERRED until SLA reached or Cycle 13a lands** | Adding it now widens the SLA gap and would be redone after the BLS architecture change | `docs/new_models_yoloe_depth_anything_v2_timing_decision.md` |
| **Depth Anything v2 integration** | **DEFERRED** | Same reasoning as YOLOE; the smallest variant alone adds ~2.5 min of new GPU wall per `combined.mp4` | `docs/new_models_yoloe_depth_anything_v2_timing_decision.md` |

### Z.5 SLA contract recap

| Quantity | Value |
|---|---|
| Benchmark video | `combined.mp4` (4 541 frames, 2 m 31 s @ 30 fps) |
| SLA contract | `total_wall ≤ duration(video) + 5 min` |
| For `combined.mp4` | total wall ≤ **7 m 31 s** = 451 s |
| Required overall FPS | **≥ 10.07** |
| Current accepted baseline (Cycle 8) | 21.87 min = **3.46 FPS** (gap: 14.4 min) |
| Cycle 9 NOT ACCEPTED but produced | 18.51 min = 4.09 FPS (gap: 11.0 min) |

Document: `docs/runtime_sla_video_plus_5min.md`.

### Z.6 Constitutional hard rules (re-affirmed across all cycles)

1. **No "accepted" without a production benchmark on the Linux RTX 5090
   server** that demonstrates the targeted metric improvement AND zero
   correctness regression vs. the prior accepted baseline. Cycle 9 is the
   precedent — working code + FPS improvement + parity, but the wrong
   metric moved, so the cycle is NOT ACCEPTED.
2. **Every optimization-cycle hypothesis must name the lever it pulls**
   (GPU child compute / dense output bytes / single-process orchestration)
   before any code is written. See §B.5.
3. **LPM scope is HARD-RESTRICTED to the 3 gaze models.** Code that mutates
   RTMPose / person_detector / sitting_standing outputs must fail loudly
   via the structural exclusion tests in
   `backend/tests/unit/pipeline/test_logical_path_matrix.py`.
4. **Multi-approach items in this file (§B.1, §B.2, §B.3) must measure ALL
   their approaches on prod before one is selected.** A new
   `docs/cycle_9b_*_results.md` file captures the comparison matrix and is
   the source of truth for the tradeoff. Selection is .env-controlled so
   rollback is a single env flip.

### Z.7 How to read this file as the next agent

1. Read § Z.1–Z.4 to confirm the cycle landscape.
2. Tick the boxes at the top of this file ("File-completion checklist")
   based on actual prod evidence — not based on local tests, not based on
   PR review, not based on "the code looks right."
3. If a box is unchecked and you intend to work on it, follow the
   per-item acceptance gate in §B / §C.
4. If you ship something here, **update both this file's checklist and the
   matching cycle row in § Z.1 (move it from Z.2/Z.3 to Z.1).**
5. If you discover a new optimization cycle that does not fit here (e.g.
   a Cycle 14 idea), add it to § Z.3 with status `PLANNED` and a doc
   reference; do not silently start it.
