# Cycle 9 + Cycle 10 — Remaining Improvements (TODO)

**Last updated:** 2026-06-06

**Status:** Consolidated TODO list. Nothing in this document is accepted or
flagged "done" until each improvement has its own production benchmark on
the Linux RTX 5090 server that demonstrates (a) the targeted metric
improvement and (b) zero correctness regression vs. the prior accepted
baseline for that cycle. The latest accepted baseline is now Cycle 12.C
single-inflight behavior overlap, job
`069a217f-fa43-48cc-bf18-c946d53bb3ee`. See
`AGENTS.md` re-affirmation and the
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

**A box may be ticked ONLY when ALL of the following are true** (this is
the §E hard-rule restated as a checklist gate):

1. The change is implemented AND has unit tests.
2. `.github/workflows/inference-parallelization.yml` lists the new test
   file(s) under both `paths:` filters AND inside the pytest invocation.
3. The change has been deployed to the Linux RTX 5090 production server.
4. `tools/prod/prod_run_parallel_flow_benchmark.sh` was executed on
   `combined.mp4` and the bench summary JSON, inference_audit JSON, and
   GPU monitor CSV are captured.
5. The resulting metrics are recorded in the matching
   `docs/cycle_*_results.md` file alongside the latest accepted baseline
   and the SLA target.
6. The corresponding row in § Z.1 / Z.2 of this file has been moved /
   updated to reflect the new accepted state.

Items per §B / §C:

- [ ] **B.1** Compact-postprocessing: ALL three approaches (B.1.a BLS Python, B.1.b C++ custom, B.1.c TRT NMS plugin) measured on prod and recorded in `docs/cycle_9b_compact_postproc_results.md`; one approach selected via `BEHAVIOR_COMPACT_BACKEND`; CI gate updated for the new tests
- [ ] **B.2** Output-fusion: B.2.b exact gaze_h slice and B.2.c exact slice + Top-K are prod-measured and accepted; standalone B.2.a top-K-only was not separately benchmarked and must be formally measured or deferred before this whole matrix is closed
- [ ] **B.3** Child critical-path: Step 1 measurement landed in `docs/cycle_9b_child_critical_path_results.md` (`gaze_horizontal_model` is the dominant child); ALL applicable Step 2 approaches still need implementation, prod measurement, and CI gate updates
- [x] **B.4** Larger-batch knob: prod-benchmarked with RSS watch; rejected in `docs/cycle_9b_batch_window_results.md`
- [ ] **C.2.1** LPM hook wired into `_run_crop_behaviour_for_items`; integration unit test added; CI gate updated (**deployed and prod-proven, but Cycle 10 acceptance failed so checkbox remains open**)
- [ ] **C.2.2** `telemetry_lpm_events` table + migration + writer landed; migration test added; CI gate updated (**deployed and prod-proven, but Cycle 10 acceptance failed so checkbox remains open**)
- [ ] **C.2.3** LPM Phase 1 prod benchmark passes ALL §10 gates of `docs/logical_path_matrix_spec.md`; metrics recorded in a new `docs/cycle_10_lpm_phase1_results.md`; LPM row in § Z.2 moved to § Z.1
- [ ] **C.2.4** LPM Phase 2 (BLS migration) landed and prod-benchmarked; results in `docs/cycle_10_lpm_phase2_results.md`
- [ ] **C.2.5** Open spec issues resolved; resolution recorded in `docs/logical_path_matrix_spec.md`
- [ ] **CI/CD audit** of this file: confirm `.github/workflows/inference-parallelization.yml` lists every new test, every new module, and every new doc this file adds before any of B.1–B.4 / C.2.1–C.2.5 is marked complete

Each item below lists its own acceptance gate; flip the checkbox above
**only** when that gate is met with prod evidence on the Linux RTX 5090
server. **There is no path that bypasses § E rules 1, 4, and 5.**

**Source documents this consolidates:**

- `docs/cycle_9_results.md` — Cycle 9 production outcome + post-mortem + the 5 continuation options
- `docs/logical_path_matrix_spec.md` — Cycle 10 LPM formal spec
- `docs/cycles_9_to_12_implementation_playbook.md` — original Cycle 9–12 plan
- `docs/triton_models_and_tensor_anatomy.md` — dense tensor inefficiency analysis
- `docs/runtime_sla_video_plus_5min.md` — SLA target (`total_wall ≤ video_duration + 5 min`)
- `AGENTS.md` — constitutional discipline

---

## Quick-reference: what a "CI/CD update" looks like (§E.4 enforcement)

Every change in this file lands in three workflow blocks of
`.github/workflows/inference-parallelization.yml`. Skipping any of these
is a §E.4 violation.

```yaml
# Block 1 — path filter under `on.push.paths` AND `on.pull_request.paths`
on:
  push:
    paths:
      - "backend/apps/pipeline/services/<NEW_MODULE>.py"
      - "backend/tests/unit/pipeline/<NEW_TEST_FILE>.py"
      - "docs/<NEW_RESULTS_FILE>.md"
      - "backend/models/triton_repository_cuda12/**"   # already present
  pull_request:
    paths:
      - "backend/apps/pipeline/services/<NEW_MODULE>.py"
      - "backend/tests/unit/pipeline/<NEW_TEST_FILE>.py"
      - "docs/<NEW_RESULTS_FILE>.md"

# Block 2 — pytest invocation
- name: Run inference parallelization tests
  run: |
    python -m pytest \
      ... existing entries ... \
      backend/tests/unit/pipeline/<NEW_TEST_FILE>.py \
      -q --tb=short
```

For multi-approach items (§B.1, §B.2, §B.3) each approach must have at
least one unit test that exercises its `.env` flag value. Example for
B.1.a:

```python
def test_compact_backend_bls_python_path_is_used(monkeypatch):
    monkeypatch.setenv("BEHAVIOR_COMPACT_BACKEND", "bls_python")
    # assert the dispatch chooses the BLS path
```

If you cannot articulate the test that exercises the new flag, you cannot
ship the flag.

---

## A. Where we stand (numbers from prod, not estimates)

| State | Job | Total wall | Overall FPS | Status |
|---|---|---:|---:|---|
| Baseline (ROI-320 only) | `77650001` | 3 469 s (57.8 min) | 1.309 | superseded |
| Cycles 1–5 | `74ec0432` | 2 186 s (36.4 min) | 2.077 | ACCEPTED |
| Cycle 6 (pose chunking) | `a1a448b9` | 1 633 s (27.2 min) | 2.780 | ACCEPTED |
| Cycle 7 (redis cache) | `515fe118` | 1 582 s (26.4 min) | 2.870 | ACCEPTED (caveat) |
| Cycle 8 (embedding stage) | `d2de80a0` | 1 312 s (21.9 min) | 3.460 | **ACCEPTED** (superseded by Cycle 9b exact slice + Top-K) |
| Cycle 9 (behavior ensemble) | `c1651663` | 1 111 s (18.5 min) | 4.090 | **NOT ACCEPTED** — Step 2 wall +0.6 % failed the ≥ 10 % gate |
| Cycle 10 (LPM) | `17075418` | 1 076 s (17.9 min) | 4.219 | **NOT ACCEPTED** — telemetry worked, but C1/eliminated stayed 0 and `attention_tracking` boxes regressed 11 776 → 2 680 |
| Cycle 9b B.2.b exact slice | `7933c1e5` | ~1 054 s (17.6 min) | 4.307 | **ACCEPTED** — Step 2 wall 858.1 → 573.927 s and correctness parity held |
| Cycle 9b B.2.c exact slice + Top-K | `be4ba9ee` | 1 023 s (17.0 min) | 4.429 | **ACCEPTED WITH CAVEAT** — Step 2 frame wall 573.927 → 540.399 s, behavior RTT 91.470 → 84.865 ms, output bytes ~95 % lower, average GPU util did not improve |
| Cycle 9b B.4 batch window 2 → 4 | `416efe8c` | 1 016 s (16.9 min) | 4.471 | **NOT ACCEPTED** — Step 2 improved 5.17 %, but behavior RTT mean regressed 16.95 %, StudentTrack count dropped 53 → 47, and model-agreement F1 failed for three behavior outputs |
| Cycle 12.C single-inflight behavior overlap | `069a217f` | 936 s (15.6 min) | 4.854 | **ACCEPTED** — Step 2 wall 540.399 → 459.461 s, behavior RTT mean 84.865 → 83.936 ms, tracks unchanged, model-agreement F1 `>=99.716 %` |

**SLA target (`combined.mp4`, 4 541 frames, 2 m 31 s):** total wall ≤ 7 m 31 s
= 451 s = **≥ 10.07 FPS overall**. Current accepted baseline (Cycle 12.C
single-inflight behavior overlap) is **15.6 min** — gap is **~8.1 min**.

---

## B. Cycle 9b — Five concrete improvements (PARTIALLY PROVEN)

Cycle 9 (plain ensemble) reduced app-side request fragmentation but did not
reduce GPU-side child compute or dense output transfer, so Step 2 wall did
not move. The next ensemble-adjacent change MUST attack one of the five
levers below. Each is a separate STAGED candidate; the production benchmark
before/after is the only acceptance evidence.

### B.1 Server-side compact postprocessing (HIGHEST IMPACT — multi-approach)

**What:** Move the post-NMS step *inside* Triton so the dense YOLO grids
never cross gRPC. Same problem, two implementation strategies — both must
be built and benchmarked side-by-side before one is selected.

**Why this matters:** before Cycle 9b B.2.c, every frame moved ~17.1 MB of
dense behavior YOLO grid bytes from GPU → CPU → gRPC → Python, ran NMS 68
times (17 crops × 4 models), then discarded 99.99 % of it. The accepted
exact-slice + Top-K route already reduced behavior outputs to roughly
`~0.33 MB/frame`, so B.1 must now be measured as a smaller but still relevant
remaining-output + client decode/NMS lever. See
`docs/triton_models_and_tensor_anatomy.md` and
`docs/cycle_9b_compact_postproc_investigation.md` for the updated byte math.

#### B.1 approaches to test

| ID | Approach | Backend type | Pros | Cons |
|---|---|---|---|---|
| **B.1.a** | **BLS Python backend** at `triton_repository_cuda12/behavior_postproc/` whose final ensemble step runs the existing `_decode_yolo_output0` math server-side | Requires production Triton Python backend verification or rebuild | Lowest engineering cost once backend exists — reuse Python NMS verbatim; debuggable in plain Python; first foothold for the Cycle 10 LPM Phase 2 migration | The active prod build did not expose the Python backend during Cycle 9b; adds a Python-interpreter hop inside Triton per call (~5–15 ms/call overhead); harder to tune at high throughput |
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

| Metric | Current Top-K baseline | B.1.a BLS Python | B.1.b C++ custom | B.1.c TRT plugin |
|---|---:|---:|---:|---:|
| Step 2 wall (s) | 540.399 | ? | ? | ? |
| Step 2 wall Δ vs Top-K | — | ? % | ? % | ? % |
| Total DB-completed elapsed (s) | 1 022.952 | ? | ? | ? |
| Overall FPS (DB completed) | 4.439 | ? | ? | ? |
| Avg GPU util (%) | 9.344 | ? | ? | ? |
| Behavior output bytes / frame | ~0.33 MB | ~2 KB | ~2 KB | ~2 KB |
| Avg behavior RTT (ms) | 84.865 | ? | ? | ? |
| p95 behavior RTT (ms) | 128.056 | ? | ? | ? |
| Per-class bbox parity (Δ %) | 0 | ≤ 0.05 ✓ | ≤ 0.05 ✓ | ≤ 0.05 ✓ |
| Engineering effort (person-days) | — | ~1 | ~5 | ~3 |
| Production deploy risk | — | medium | high | medium-high |
| Rollback complexity | trivial | env flip | env flip + binary swap | env flip + engine swap |

**Phase A decode-cost probe (2026-06-02):** production probe
`backend/logs/cycle9b_b1_decode_cost_topk_20260602T185559Z.json` sampled `340`
real accepted-Top-K crops. Mean Python decode/NMS was `3.125 ms` per 17-crop
batch (`0.183823 ms/crop`), mean `as_numpy` parse was `0.114 ms`, and behavior
output bytes were already only `19,200 bytes/crop`. With `3597` accepted
baseline behavior calls, observed client decode/NMS is about `11.24 s`
(`~2.08 %` of Step 2 wall). B.1 remains open, but a candidate that only moves
Top-K decode/NMS out of Python is unlikely to satisfy the `>=10 %` Step 2 gate.

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
2. Per-class bbox count parity within 0.05 % of the latest accepted baseline
   for the cycle.
3. Step 2 wall reduction ≥ 10 % on `combined.mp4`.
4. All three approaches' bench numbers recorded in
   `docs/cycle_9b_compact_postproc_results.md` even if only one ships —
   so future agents have the evidence for the tradeoff.

### B.2 Output fusion — class-trim and top-K (multi-approach)

**What:** Lighter-weight alternative to B.1 — keep dense YOLO format but
trim the response to only the *needed* logits. Smaller engineering
footprint than the full BLS path.

#### B.2 approaches to test

| ID | Approach | What changes | Bytes saved per crop |
|---|---|---|---:|
| **B.2.a** | **Top-K anchor packing** — extra ensemble step (or tiny TensorRT op) returns only the top-K anchors per crop per model with `K = TRITON_YOLO_MAX_DECODE_CANDIDATES = 100` | Output shape per model becomes `[C, K]` instead of `[C, 2100]` — `100/2100 ≈ 4.8 %` of original | ~95 % per model |
| **B.2.b** | **Class-channel trim for `gaze_horizontal_model`** — staged as an ONNX output-slice variant, not a retrained head | Output shape changes from `[84, 2100]` to `[6, 2100]`; compact class IDs are remapped back to legacy IDs `4/5` | ~93 % on this one model (689 KB → ~50 KB per crop = **~11 MB / frame** saved) |
| **B.2.c** | **Combine B.2.a + B.2.b** — top-K packing AND the accepted exact slice for `gaze_horizontal_model` (the other three behavior models keep their 10-class head) | Stacked savings | ~99 % per crop on `gaze_horizontal_model`, ~95 % on the others |

#### B.2 selection mechanism (.env-controlled)

```env
# Each approach independently toggleable
TRITON_BEHAVIOR_TOP_K_ENABLED=0           # B.2.a — top-K anchor packing
TRITON_BEHAVIOR_TOP_K_VALUE=100            # K must equal the client-side cap to preserve parity

GAZE_HORIZONTAL_HEAD_VARIANT=coco80        # B.2.b — "coco80" (legacy 84-channel) | "gaze2" (rejected standalone 6-channel) | "slice" (exact server-side 6-channel)
```

Setting both to non-default values activates **B.2.c**. The client-side
`_decode_yolo_output0` must read `GAZE_HORIZONTAL_HEAD_VARIANT` to know
whether to expect 84 or 6 channels on that specific model.

**Current implementation state (2026-06-02):** B.2.b's first implementation,
the separate TensorRT output-slice variant behind
`GAZE_HORIZONTAL_HEAD_VARIANT=gaze2`, is **NOT ACCEPTED**. It created
`gaze_horizontal_gaze2_model` by gathering channels `[0,1,2,3,8,9]` from the
legacy horizontal ONNX output, routed `behavior_all` to
`behavior_ensemble_gaze2`, and remapped compact class IDs `0/1` back to legacy
DB IDs `4/5`. Local focused validation passed (`160 passed`), but production
raw tensor parity failed twice; the final rebuilt-engine probe
`backend/logs/gaze_horizontal_gaze2_parity_20260601T231503_postrebuild.json`
reported `max_abs_diff=9.5` at tolerance `1e-6`. No full benchmark was run.
Production was rolled back to `GAZE_HORIZONTAL_HEAD_VARIANT=coco80` and the
legacy `behavior_ensemble` route. Any future B.2 approach must preserve parity,
preferably by slicing or packing the already-executed legacy output server-side
or by passing an explicit decoded-detection parity gate.

**Accepted follow-up (2026-06-02):** exact server-side slicing is accepted behind
`GAZE_HORIZONTAL_HEAD_VARIANT=slice`. It keeps `gaze_horizontal_model`
unchanged, adds `gaze_horizontal_slice_model` to gather channels
`[0,1,2,3,8,9]` from the legacy dense output inside Triton, routes
`behavior_all` through `behavior_ensemble_gaze_slice`, and routes standalone
fallback through `gaze_horizontal_slice_adapter`. Production benchmark
`cycle9b-exactslice-crop-frame-20260601T233211` / job
`7933c1e5-a970-47a3-81c5-0c9bd01bd332` completed, post-benchmark parity passed
with `max_abs_diff=0.0`, and Step 2 wall improved `858.1 s → 573.927 s`.
The follow-up Top-K route is also accepted with caveat behind
`TRITON_BEHAVIOR_TOP_K_ENABLED=1`. It keeps exact slicing, adds FP32 Top-K
adapters after all four behavior children, routes `behavior_all` through
`behavior_ensemble_gaze_slice_topk`, and uses `TRITON_BEHAVIOR_TOP_K_VALUE=100`.
Production benchmark `cycle9b-topk-crop-frame-20260602T041900` / job
`be4ba9ee-4786-48e9-8334-28feb237a1fb` completed, decoded parity passed exactly
(`failed_count=0`, `max_score_diff=0.0`, `max_box_diff=0.0`), and Step 2 frame
wall improved `573.927 s → 540.399 s`. Average GPU utilization did not improve,
so the caveat is explicit. Standalone B.2.a top-K-only was not separately
benchmarked; the accepted production baseline has moved to B.2.c.

#### B.2 measurement matrix (one prod bench per row)

Record in `docs/cycle_9b_output_fusion_results.md`. All three rows must be
benchmarked even if only one ships.

| Metric | Legacy (Cycle 9) | B.2.a top-K only | B.2.b exact gaze_h slice | B.2.c both |
|---|---:|---:|---:|---:|
| Step 2 wall (s) | 858.1 | not separately measured | **573.927** | **540.399** |
| Step 2 wall Δ vs legacy | — | not separately measured | **−33.1 %** | **−37.0 %** |
| Total DB-completed elapsed (s) | 1 110.7 | not separately measured | **1052.281** | **1022.952** |
| Overall FPS (DB completed) | 4.09 | not separately measured | **4.307** | **4.429** |
| Behavior dense bytes / frame | 17.1 MB | not separately measured | **~6.85 MB** | **~0.33 MB** |
| Avg behavior RTT (ms) | 107.9 | not separately measured | **91.470** | **84.865** |
| Per-class bbox parity (Δ %) | 0 | not separately measured | **pass** | **pass** |
| Engineering effort (person-days) | — | ~1.5 | ~2 (re-export + accuracy probe) | ~3 |
| Production deploy risk | — | low | medium (engine swap) | medium |
| Rollback complexity | trivial | env flip | env flip + engine swap | env flip + engine swap |

**Selection rule result:** B.2.c is the current production default because it
beats B.2.b on Step 2 wall, behavior RTT, and DB-completed FPS while preserving
correctness. The caveat is that average GPU utilization did not improve, so do
not keep optimizing response-byte volume alone.

**B.2 vs B.1 — should both be built?** Yes. B.2 is a lower-risk fallback
if B.1 slips. They are not mutually exclusive — B.1 (server-side NMS) and
B.2.b (smaller `gaze_horizontal_model` output) compose: a BLS backend
operating on a 6-channel `gaze_horizontal` output is even cheaper than one
operating on 84-channel.

**Risk:** Medium. Top-K `K` value must match
`TRITON_YOLO_MAX_DECODE_CANDIDATES` exactly. B.2.b exact slice has already
passed raw tensor parity; any new B.2.b replacement must pass the same gate.

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

**Current measured status (2026-06-02):** Step 1 was first completed against
the pre-Top-K baseline and **REMEASURED** against the accepted Cycle 9b Top-K
baseline (job `be4ba9ee-4786-48e9-8334-28feb237a1fb`). Production stats and
direct gRPC decomposition under the Top-K topology now show:

| Model | Pre-Top-K server avg | Post-Top-K server avg | Change |
|---|---:|---:|---:|
| `posture_model` | 12.133 ms | 16.324 ms | +4.19 ms |
| `gaze_horizontal_model` | 16.058 ms | **18.790 ms** | +2.73 ms |
| `gaze_vertical_model` | 11.759 ms | 16.236 ms | +4.48 ms |
| `gaze_depth_model` | 11.909 ms | 16.377 ms | +4.47 ms |
| Dominant-vs-next gap | +33 % | **+15 %** | shrank by 18 pp |
| `behavior_ensemble_gaze_slice_topk` server avg | n/a | **30.133 ms** | — |
| Orchestration overhead | n/a | **~8.7 ms / call** | — |

`gaze_horizontal_model` is still the dominant child but the per-child
optimization ceiling shrank to ~4 % Step 2 wall (gap = 2.47 ms out of
30.13 ms ensemble). The remeasurement evidence and its lever-ranking
recommendation are in
`docs/cycle_9b_child_critical_path_remeasure_topk_results.md`. **Step 2
must therefore compare per-child dominant tuning (B.3.b / B.3.d) against
the higher-leverage all-children candidate (Cycle 11 input 320 → 256) per
§E.6 multi-approach rule before any candidate ships.**

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

**Production result (2026-06-02): NOT ACCEPTED.** Replay key
`cycle9b-b4-maxframes4-20260602T175820Z`, job
`416efe8c-772c-442f-8e55-cf44c54fe261`, completed `4541/4541` frames.
Worker RSS stayed bounded (`1120.328 MiB`) and Step 2 frame wall improved
`540.399 s → 512.445 s` (`-5.17 %`), but behavior RTT mean regressed
`84.865 ms → 99.251 ms`, `StudentTrack` count dropped `53 → 47`, and
baseline-agreement F1@IoU0.5 failed for `attention_tracking` (`24.531 %`),
`hand_raising` (`26.648 %`), and `sitting_standing` (`17.217 %`). Keep
`TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=2`. Evidence:
`docs/cycle_9b_batch_window_results.md` and
`docs/production_inference_benchmark.md` §21.

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

## C. Cycle 10 — Logical Path Matrix (LPM) (STAGED, HOOK + TELEMETRY WIRED LOCALLY)

### C.1 What landed already

- `backend/apps/pipeline/services/logical_path_matrix.py` — pure-function
  constraint solver with `LpmConfig`, `LpmFrameInput`, `LpmState`,
  `LpmBatchMetrics`, `apply(...)`, `estimate_head_yaw_deg(...)`,
  `apply_to_detection_boxes(...)`.
- 28 unit tests passing covering all of C1–C4, the disabled-flag identity,
  the exclusion contract, the RTMPose head-yaw helper, batch metrics, and
  the integration helper.
- C.2.1 hook wiring is now locally implemented in
  `backend/apps/video_analysis/tasks.py` and covered by
  `backend/tests/unit/video_analysis/test_lpm_crop_behaviour_hook.py`.
- C.2.2 telemetry is now locally implemented: `TelemetryLpmEvent`,
  migration `0002_telemetrylpmevent`, `LpmEventMeta`,
  `TelemetrySession.record_lpm_event(...)`, writer bulk insert + JSON fallback,
  and crop-frame `_tel_record_lpm_event(...)` are covered by telemetry and hook
  tests.
- All settings registered in `backend/config/settings/base.py`
  (`LPM_ENABLED` default `0`).
- All prod env knobs registered in `tools/prod/prod_enable_parallel_flow.sh`
  at value `0`.
- CI workflow gates the new module + tests.
- Formal spec at `docs/logical_path_matrix_spec.md`.

### C.2 What remains before acceptance (this is the TODO list)

#### C.2.1 Wire the LPM hook into `tasks.py` — LOCALLY STAGED

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

**Local status (2026-06-01):** implemented behind `LPM_ENABLED=0` default.
The hook maintains a per-job prior-state cache and applies only after crop
behavior/gaze boxes are decoded. Integration coverage is in
`backend/tests/unit/video_analysis/test_lpm_crop_behaviour_hook.py`. The
top-level checkbox remains unchecked until production deployment and the
C.2.3 benchmark evidence exist.

#### C.2.2 Telemetry table for LPM events — LOCALLY STAGED

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

**Implementation:** model in `backend/apps/telemetry/models.py`, migration
`backend/apps/telemetry/migrations/0002_telemetrylpmevent.py`, public
`LpmEventMeta` + `TelemetrySession.record_lpm_event(...)`, writer helper
`_bulk_insert_lpm_events`, JSON fallback keys `lpm_events` /
`lpm_event_count_in_json`, and crop-frame `_tel_record_lpm_event(...)`.

**Local status (2026-06-01):** implemented and covered by
`backend/tests/unit/telemetry/test_telemetry_layer.py` plus
`backend/tests/unit/video_analysis/test_lpm_crop_behaviour_hook.py`. The
top-level checkbox remains unchecked until the migration is applied on
production and C.2.3 benchmark evidence exists.

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
| 4 | **B.2b exact server-side `gaze_horizontal` slice** | 9b | accepted on production; superseded by B.2c as the current baseline |
| 5 | **B.2c exact slice + Top-K anchor packing** | 9b | accepted with caveat on production; current baseline before B.1/B.3/B.4 |
| 6 | **B.3 child critical-path analysis — Step 1 REMEASURED 2026-06-02** | 9b | `gaze_horizontal_model` still dominant but gap shrank to +15 %; per-child ceiling ≤ 4 % Step 2 wall. Cycle 11.A input `320 → 256` was benchmarked on production: Step 2 improved strongly, but persisted behavior signals regressed and GPU avg util fell, so it is not accepted. |
| 7 | **B.1 server-side compact postprocessing (BLS)** | 9b | highest impact but highest risk — depends on B.2c learnings; also recovers the ~8.7 ms/call ensemble orchestration overhead measured in the B.3 remeasure |
| 8 | **C.2.4 LPM Phase 2 (move math into BLS)** | 10 (Phase 2) | combines with B.1 — both land in the same BLS Python backend |
| 9 | **B.4 larger ensemble batches** | 9b | measured and rejected on production; do not repeat `max_frames=4` unless the tracking/model-agreement failure is addressed first |

Steps 1–3 unblock the LPM acceptance gate. Steps 4–6 are the path to the
≥ 10 % Step 2 wall reduction that Cycle 9 failed to achieve. Step 7 is the
architectural cleanup that fuses LPM into the same BLS layer as compact
postprocessing.

---

## E. Hard rules (re-stated)

These are NOT guidelines. They are gating conditions. Any commit that
violates one of these must be reverted and re-done, regardless of the
quality of the diff.

1. **No "accepted" without a production benchmark on the Linux RTX 5090
   server.** Code review, parity probes, local pytest passes, and PR
   approvals are necessary but NEVER sufficient. Cycle 9 is the precedent —
   working code + measurable FPS improvement + parity, but the wrong
   metric moved, so the cycle is NOT ACCEPTED. The acceptance bar is
   evidence on the live production server, not the dev box.

2. **No optimization-cycle hypothesis without naming the lever it pulls**
   (the Cycle 9b §B.5 discipline rule). Future hypotheses must say "this
   attacks GPU child compute" OR "this attacks dense output bytes" OR
   "this attacks single-process Python orchestration" — and the proposed
   change must actually move that lever. Cycle 9 violated this implicitly
   (it attacked gRPC call count, which was already absorbed by concurrent
   dispatch), which is why it didn't deliver the targeted Step 2 wall
   reduction.

3. **LPM scope is HARD-RESTRICTED.** Code that mutates RTMPose /
   person_detector / sitting_standing outputs must fail loudly via the
   structural exclusion tests in `test_logical_path_matrix.py`. The three
   gaze models are the entire scope; nothing else.

4. **CI/CD workflow file updates are MANDATORY, not optional.** Every
   change in this file that lands code must, in the same commit:
   - List any new test file in
     `.github/workflows/inference-parallelization.yml` under the path
     filter AND inside the pytest invocation in `Run inference
     parallelization tests`.
   - List any new app-side module (e.g. a new BLS backend module, a new
     constraint solver helper) under the same workflow's path filter so
     pull requests touching the module re-run the gate.
   - List any new doc that captures acceptance evidence (e.g.
     `docs/cycle_9b_compact_postproc_results.md`) so a docs-only change
     also reruns the workflow.
   - If the change touches Triton model configs, list the affected
     `backend/models/triton_repository_cuda12/**` path filter (already
     present — verify it is not removed).
   - Every new `.env` flag introduced under §B / §C must be exercised by
     at least one unit test. The CI workflow must include that test.
   - Commits that add code without updating the workflow are treated as
     incomplete and must be amended before review.

5. **No option, optimization, or strategy is accepted until it has been
   really tested, measured, and compared to the SLA Baseline on the
   production Linux server.** "Tested" means pytest passes in CI AND on
   prod. "Measured" means `tools/prod/prod_run_parallel_flow_benchmark.sh`
   ran on `combined.mp4` end-to-end, the bench summary JSON was captured,
   the inference_audit was captured, and the GPU monitor CSV was captured.
   "Compared to the SLA Baseline" means the resulting metrics are recorded
   in a row of the matching `docs/cycle_*_results.md` file alongside the
   prior accepted baseline AND the SLA target row from
   `docs/runtime_sla_video_plus_5min.md`. Without all three artefacts plus
   the row in the results doc, the item stays STAGED, not ACCEPTED — even
   if the unit tests are green, even if the code reviewer signed off, even
   if the FPS number looks better.

6. **Multi-approach items in this file (§B.1, §B.2, §B.3) must benchmark
   ALL their approaches on prod before one is selected.** A new
   `docs/cycle_9b_*_results.md` file captures the comparison matrix and is
   the source of truth for the tradeoff. Selection is `.env`-controlled so
   rollback is a single env flip. Shipping only one approach of a
   multi-approach item without the comparison evidence violates this rule.

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
| Cycle 9b B.2.b | Exact server-side horizontal-gaze slice | **ACCEPTED 2026-06-02** | `7933c1e5-a970-47a3-81c5-0c9bd01bd332` | Dense horizontal output returned to Python changed `[84,2100] → [6,2100]`, post-benchmark tensor parity `max_abs_diff=0.0`, Step 2 wall `858.1 → 573.927 s` (`−33.1 %`), DB FPS `4.09 → 4.307`, correctness parity held | `docs/production_inference_benchmark.md` §18, `docs/cycle_9b_output_fusion_results.md`, `docs/crop_frame_optimization_execution.md` |
| Cycle 9b B.2.c | Exact slice + Top-K anchor packing | **ACCEPTED WITH CAVEAT 2026-06-02** | `be4ba9ee-4786-48e9-8334-28feb237a1fb` | FP32 Top-K adapters returned `[C,100]` behavior outputs, decoded parity passed exactly, Step 2 frame wall `573.927 → 540.399 s` (`−5.84 %` vs exact slice), DB FPS `4.307 → 4.429`, behavior RTT mean `91.470 → 84.865 ms`, behavior output traffic `~6.85 → ~0.33 MB/frame`; caveat: average GPU util `9.595 % → 9.3 %` | `docs/production_inference_benchmark.md` §19, `docs/cycle_9b_topk_anchor_packing_results.md`, `docs/cycle_9b_output_fusion_results.md`, `docs/crop_frame_optimization_execution.md` |
| Cycle 9b B.4 | Batch window 2 → 4 | **NOT ACCEPTED 2026-06-02** | `416efe8c-772c-442f-8e55-cf44c54fe261` | Step 2 frame wall `540.399 → 512.445 s` (`−5.17 %`) and DB FPS `4.439 → 4.471`, but behavior RTT mean regressed `84.865 → 99.251 ms`, StudentTrack count dropped `53 → 47`, and model-agreement F1@IoU0.5 failed for three behavior outputs. Prod restored to `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=2`. | `docs/production_inference_benchmark.md` §21, `docs/cycle_9b_batch_window_results.md`, `docs/cycle_9b_batch_window_investigation.md` |
| Cycle 10 | Logical Path Matrix (LPM) Phase 1 | **NOT ACCEPTED 2026-06-01** | `17075418-4386-4b5f-85d4-ea23bec71f66`; safety proof `21666815-f4bd-4f5f-b90e-b9101b4d899d` | Telemetry table worked (`4541` rows), but contradictions were not detected (`C1=0`, `eliminated=0`) and `attention_tracking` boxes regressed 11 776 → 2 680. Safety proof at SHA `31edac44` restored most boxes (`2680 → 11122`) but still failed Cycle 9 parity (`11776`) and still recorded `C1=0`, `eliminated=0`. Prod rolled back to `LPM_ENABLED=0`. | `docs/production_inference_benchmark.md` §16, `docs/cycle_10_lpm_phase1_results.md` |
| Cycle 12.C | Single-inflight behavior overlap | **ACCEPTED 2026-06-03** | `069a217f-fa43-48cc-bf18-c946d53bb3ee` | Step 2 wall `540.399 s → 459.461 s`, DB FPS `4.439 → 4.854`, GPU avg `9.344 % → 10.332 %`, behavior RTT mean `84.865 ms → 83.936 ms`, tracks unchanged, model-agreement F1 `>=99.716 %`. Production profile uses `TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=1`. | `docs/cycle_12_single_inflight_overlap_investigation.md`, `docs/cycle_12_single_inflight_overlap_results.md`, `docs/production_inference_benchmark.md` §26 |
| Cycle 13.A | Embedding-stage profiling | **MEASUREMENT COMPLETE / HYPOTHESIS_ONLY 2026-06-03** | `aa2fe7a9-b3fb-49d7-92a3-eca41c894dcd` | Profiling-only replay `cycle13-embedding-profile-20260603T003853Z` preserved DB/model parity exactly but made no optimization decision. It measured embedding wall `188.620 s`: track lookup `66.223 s`, Redis flush `59.304 s`, DB flush `38.467 s`, existing checks `14.527 s`. | `docs/cycle_13_embedding_profile_results.md`, `docs/production_inference_benchmark.md` §28 |
| Cycle 13.B | Prefetch-aware embedding track lookup | **ACCEPTED 2026-06-03** | `c9f75d55-6043-4f27-bf9e-b2826d299459` | DB elapsed `935.516 s -> 872.317 s`, DB FPS `4.854005 -> 5.205675`, embedding wall `188.620 s -> 121.681 s`, track lookup `66.223 s -> 0.447 s`, exact DB/model parity. Production profile uses `EMBEDDING_PREFETCH_TRACK_LOOKUP=1`. | `docs/cycle_13_embedding_track_lookup_results.md`, `docs/production_inference_benchmark.md` §29 |

### Z.2 Cycles staged or in progress (code exists, awaiting prod evidence)

| # | Title | Status | What is missing | Primary docs |
|---|---|---|---|---|
| **Cycle 9b remaining** | Compact postprocessing, child critical-path, discipline rule | **PARTIALLY ACCEPTED — B.2.b EXACT SLICE AND B.2.c TOP-K ACCEPTED; B.3 STEP 1 REMEASURED; B.4 REJECTED 2026-06-02** | B.2.b separate TensorRT output-slice code behind `GAZE_HORIZONTAL_HEAD_VARIANT=gaze2` is NOT ACCEPTED (`max_abs_diff=9.5`). Exact server-side slicing behind `GAZE_HORIZONTAL_HEAD_VARIANT=slice` is ACCEPTED. Exact slice + Top-K behind `TRITON_BEHAVIOR_TOP_K_ENABLED=1` is ACCEPTED WITH CAVEAT. B.3 Step 1 has been remeasured against the Top-K baseline — dominant child is still `gaze_horizontal_model` but the gap to the next slowest shrank from +33 % to +15 %, capping any per-child Step 2 candidate at ~4 % Step 2 wall reduction. B.4 was production-benchmarked and rejected because `max_frames=4` failed track/model-agreement gates. B.1 and B.3 Step 2 remain; standalone B.2.a Top-K-only was not separately benchmarked and is lower priority than GPU-occupancy/server-side execution work. | This file, `docs/cycle_9_results.md`, `docs/cycle_9b_child_critical_path_results.md`, `docs/cycle_9b_child_critical_path_remeasure_topk_results.md`, `docs/cycle_9b_output_fusion_investigation.md`, `docs/cycle_9b_exact_slice_investigation.md`, `docs/cycle_9b_topk_anchor_packing_investigation.md`, `docs/cycle_9b_topk_anchor_packing_results.md`, `docs/cycle_9b_batch_window_investigation.md`, `docs/cycle_9b_batch_window_results.md`, `docs/cycle_9b_output_fusion_results.md` |
| **Cycle 11.A** | Behavior input 320 → 256 | **NOT ACCEPTED BY REAL BENCHMARK 2026-06-02** | Production benchmark `cycle11-input256-realbench-20260602T161641Z-input256` / job `822b0da4-fbf2-4186-a5a6-dd066f2eb571` completed. It improved Step 2 wall `540.399 s → 391.673 s`, Step 2 FPS `8.403 → 11.594`, and behavior RTT mean `84.865 ms → 51.529 ms`, but detection/bbox rows regressed `72,762 → 101,213` (`+39.10 %`), `attention_tracking` boxes regressed `11,781 → 20,558` (`+74.50 %`), and baseline-agreement F1@IoU0.5 fell to `31.195 %` for `attention_tracking`, `38.032 %` for `hand_raising`, and `65.250 %` for `sitting_standing`; `person_detection` stayed `100.000 %`. GPU avg util fell `9.344 % → 7.367 %`. Production restored to `TRITON_CROP_BEHAVIOR_INPUT_SIZE=320`, exact slice + Top-K. | `docs/cycle_11_input_size_investigation.md`, `docs/cycle_11_input_size_results.md` |
| **Cycle 10 follow-up** | LPM contradiction-signal redesign | **STAGED AFTER REJECTION** | Fresh safety-fix prod benchmark ran (`cycle10-lpm-violationonly-crop-frame-20260601T221110`, job `21666815-f4bd-4f5f-b90e-b9101b4d899d`). It improved the attention-box loss but still failed parity and kept `C1=0`, `eliminated=0`. Missing: redesign that captures pre-decode probabilities instead of post-decode boxes only, or migration into compact postprocessing/BLS where dense gaze outputs are still available. | `docs/cycle_10_lpm_phase1_results.md`, `docs/logical_path_matrix_spec.md`, `docs/cycle_10_investigation.md` |
| **Cycle 12.A** | Persistent async dispatcher measurement | **PHASE A MEASUREMENT COMPLETE; NO OPTIMIZATION DECISION** | Clean production replay `cycle12-async-dispatch-profile-clean-20260602T213441Z` / job `dfa1f138-7086-418a-ba17-9999cd12b9ac` completed. Async-dispatch blocking wall was `349.643 s`; `behavior_all` owned `338.779 s`. No optimization candidate was deployed, so no acceptance/rejection/skip/closure decision exists. Metric decision: do not implement bridge-only dispatcher because the estimated `35.3 s` bridge gap is below the `>=10 %` Step 2 gate. | `docs/cycle_12_persistent_dispatcher_investigation.md`, `docs/cycle_12_persistent_dispatcher_results.md`, `docs/inference_parallelization_plan.md`, `docs/crop_frame_optimization_execution.md` |
| **Cycle 12.B** | Bounded behavior-wait overlap dispatcher | **NEEDS FURTHER ITERATION; NOT ACCEPTED 2026-06-03** | Production benchmark `cycle12-behavior-overlap-20260602T223350Z` / job `46ba8b2a-3c61-4d89-b7b6-63ec72159428` completed. It improved Step 2 wall `540.399 s → 395.495 s` and DB FPS `4.439 → 5.195`, with tracks unchanged and F1 `>=99.716 %`, but behavior RTT mean regressed `84.865 ms → 115.420 ms` and p95 regressed `128.056 ms → 224.661 ms`. The likely contention source is two behavior jobs briefly in flight. | `docs/cycle_12_overlap_dispatcher_investigation.md`, `docs/cycle_12_overlap_dispatcher_results.md`, `docs/production_inference_benchmark.md` §25 |

### Z.3 Current Sorted Open Latency Queue (2026-06-06)

This queue is sorted by measured upside, dependency order, and implementation
readiness. It supersedes the older restaged table below for deciding what to
start next.

| Sort | Cycle | State | Why this position | Expected gain / gate |
|---:|---|---|---|---|
| 1 | **Cycle 20 post-stage timeline, then streaming persistence and embedding overlap** | `20.E NOT ACCEPTED / LOCK RELEASED` | Cycle 18.D is complete and **NOT ACCEPTED**; sharding remains blocked by packet-schema and identity correctness. Cycle 20 became the next non-sharding latency lane and produced serial timeline replay `cycle20-post-stage-timeline-20260605T212526Z`, terminal-marker replay `cycle20c-terminal-marker-r3-20260605T233053Z`, streaming-writer r3 replay `cycle20d-streaming-persistence-r3-20260606T011056Z`, and final-stable replay `cycle20e-final-stable-overlap-20260606T092512Z`. | 20.E fixed the packet root causes (`4541/4541` final-stable packets, `0` failed, Step 3 reconciled `0/4541`), but DB FPS regressed `5.86 %`, total elapsed regressed `6.22 %`, Step 2 through-pose wall regressed `12.29 %`, and embedding stayed serial. Do not rerun the same final-stable profile; a future post-stage attempt must start embedding earlier or expose a separate measured bottleneck. |
| 2 | **Cycle 21 Celery worker/thread/concurrency matrix** | `GOVERNANCE ONLY / BLOCKED BY 20.E RESULT` | Extra workers help only when independent work exists from sharding, post-stage overlap, multiple jobs, or another measured queue bottleneck. Cycle 20.E repaired persistence overlap counters but still did not create beneficial independent embedding work, so this remains governance-only. | Unknown until topology matrix benchmark; must include duplicate-worker checks, resource budgets, DB/Redis/GPU contention, correctness, and rollback. |
| 3 | **Cycle 11.B / Cycle 9b B.3 child-kernel tuning at 320** | `PLANNED / LOW CEILING` | The remaining dominant-child gap is real but small compared with sharding/post-stage work. | About `~4 %` Step 2 wall ceiling, roughly `18 s` on a `459 s` Step 2 frame wall. |
| 4 | **Cycle 9b B.1 / Cycle 14.D compact postprocessing** | `OPEN / NO IMPLEMENTATION SELECTED` | Top-K already removed most output bytes; Phase A measured decode/NMS as small and Python BLS is blocked by the pinned runtime. | Decode/NMS-only gain is about `2 %` of Step 2; a real candidate must reduce server wait or execution, not only output bytes. |
| 5 | **Cycle 19 Redis server-side scripts** | `CONDITIONAL` | Cycle 16.B already removed the measured Redis side-effect bottleneck. Scripts require a new measured Redis read/compute/write hotspot. | No active gain estimate; start only after a production profiler shows a script-shaped hotspot. |
| 6 | **Cycle 10 LPM redesign** | `STAGED AFTER REJECTION` | LPM is a fusion-quality/correctness lane, not the latency-first lane. | No throughput gain expected; acceptance depends on contradiction/fusion correctness, not FPS. |
| Blocked | **Cycle 15.B1 identity-fixed sharding rerun; 15.B2 only after B1 passes** | `B1 NOT ACCEPTED / B2 BLOCKED` | Cycle 18.D fixed packet validity and then tested a real OSNet-AIN `triton_reid` descriptor, but the latest replay still failed merge readiness (`1/2`), StudentTrack parity (`53 -> 57`), model agreement, and label-invariant identity. | Same measured two-shard envelope exists, but no sharding gain is accepted. A new identity-state producer, human-labeled ground-truth evaluation, or association redesign must pass a two-shard benchmark before 15.B1/15.B2 can resume. |

**Most recent cycle decision:** Cycle 20.E final-stable persistence replay
`cycle20e-final-stable-overlap-20260606T092512Z` is **NOT ACCEPTED**.
It repaired the Cycle 20.D root causes (`4541/4541` final-stable packets,
`0` final-stable failures, and Step 3 reconciled `0/4541`), but DB FPS
regressed `5.619787 -> 5.290508`, total elapsed regressed
`808.038 s -> 858.330 s`, Step 2 through-pose wall regressed
`641.154064 s -> 719.937697 s`, behavior call rate regressed
`4.451525 -> 4.190697 calls/s`, and embedding still did not start before
inference finished. The same final-stable profile must not be rerun as a new
decision and must not justify Cycle 21 worker-count scaling.

Prior sharding decision: Cycle 18.D OSNet-AIN Triton ReID replay
`cycle18d-osnet-reid-20260605T202019Z` is **NOT ACCEPTED**. The model build and
parity gates passed, DB FPS improved `5.619787 -> 7.584265`, and Step 2 wall
improved `467.449833 s -> 245.762854 s`, but identity gates still failed:
merge-ready packets `1/2`, shard-1 existing-parent mapping `17/36`, offset
fallbacks `19/36`, StudentTracks `53 -> 57`, minimum model-agreement
F1@IoU0.5 `53.788 %`, and minimum shard-1 global-assignment F1 `79.876 %`.

### Z.3a Historical planned/newly-started map (restaged after Cycle 12.C)

| # | Title | Status | Projected gain | Primary docs |
|---|---|---|---:|---|
| **Cycle 10b** | Pose parallelization across frames (originally Cycle 10 in the playbook — slot taken by LPM; pose work moved here) | **PLANNED** | Pose wall 220 → ~80 s | `docs/cycles_9_to_12_implementation_playbook.md` §3 |
| **Cycle 11.B / B.3 Step 2** | Kernel-tactic or batch-profile tuning on the dominant child at 320 | **PLANNED AFTER CYCLE 12.A MEASUREMENT** | bounded at ~4 % Step 2 wall | `docs/cycle_9b_child_critical_path_remeasure_topk_results.md`, `docs/cycle_11_input_size_results.md` |
| **Cycle 13.C / Cycle 16.A** | Redis/DB side-effect measurement after Cycle 13.B | **MEASUREMENT COMPLETE / HYPOTHESIS_ONLY** | Production replay `cycle13c-redis-command-profile-20260603T020723Z` proved Redis server wall is only `530.485 ms`; client-side helper/payload/pipeline overhead is the next target. | `docs/cycle_13c_redis_db_side_effect_measurement_results.md`, `tools/prod/prod_run_cycle13c_redis_command_profile_benchmark.sh` |
| **Cycle 16.B** | Redis side-effect coalescing for embedding/tracking side effects | **ACCEPTED 2026-06-03** | Redis flush wall improved `59.874 s -> 35.970 s`, embedding wall improved `121.681 s -> 97.505 s`, DB FPS improved `5.205675 -> 5.347791`, exact DB/model parity. | `docs/cycle_16b_redis_side_effect_coalescing_results.md` |
| **Cycle 14.A-C3** | RTMPose pose-tail decomposition, scenario split, batch-size matrix, and batch-32 repair | **CLOSED THROUGH 14.C3** | 14.B2 batch `16` accepted; 14.C1 batch `8`, 14.C2 batch `32`, and 14.C3 batch-32 chunk parallelism are not accepted. | `docs/cycle_14a_pose_tail_decomposition_results.md`, `docs/cycle_14b_rtmpose_scenario_results.md`, `docs/cycle_14c_pose_batch_size_matrix_results.md`, `docs/cycle_14c3_batch32_parallel_chunks_results.md` |
| **Cycle 14.D** | Server-side compact postprocessing / BLS / TRT plugin if it reduces wait or server execution | **PHASE A COMPLETE / NO IMPLEMENTATION SELECTED** | Python BLS is runtime-blocked; current parse/decode is only `3.357 ms/batch` after Top-K, so no compact-output implementation is selected without stronger server-wait proof. | `docs/cycle_14d_server_side_compact_postproc_results.md` |
| **Cycle 15** | CUDA shared memory or sharding architecture decision | **PHASE A MEASURED / 15.B DESIGN-PROOF NEXT** | Shared memory is not selected for immediate implementation; video sharding moves to deterministic stitching/idempotency design proof before code. | `docs/cycle_15_cuda_shared_memory_vs_sharding_results.md` |
| **Cycle 15.B** | Video sharding design proof | **PHASE A STARTED** | First scenario is two-shard dry-run/design proof; no implementation until overlap ownership, track stitching, DB idempotency, and terminal-state coordination are specified. | `docs/cycle_15b_video_sharding_design_proof_investigation.md` |
| **Cycle 15.B1/B2** | Two-shard and four-shard sharded runtime | **B1 NOT ACCEPTED / B2 BLOCKED** | B1 context `32`, B1.C1 context `256`, and B1.C2 majority vote all completed real production benchmarks and failed identity/model-agreement gates. B2 remains blocked because two-shard identity is not correct. | `docs/cycle_15b1_two_shard_runtime_investigation.md`, `docs/cycle_15b_shard_design_probe_results.md`, `docs/production_inference_benchmark.md` |
| **Cycle 15.B design proof** | Production dry-run comparison | **DESIGN PROOF PASSED / RUNTIME REJECTED** | The design proof remains useful, but the implemented runtime candidates failed correctness. Further sharding requires a new identity-state design proof before another runtime benchmark. | `docs/cycle_15b_shard_design_probe_results.md`, `docs/cycle_15b1_two_shard_runtime_investigation.md` |
| **Cycle 17** | Redis Streams for non-authoritative progress and benchmark sampling | **ACCEPTED OBSERVABILITY-ONLY / NOT THROUGHPUT** | Production replay `cycle17-redis-streams-20260604T025328Z` completed with exact DB/model parity, bounded stream evidence (`4729` writes, `XLen=1002`, zero Redis errors), and rollback verified; DB FPS was neutral (`5.620 -> 5.611`), so no throughput gain is claimed. | `docs/cycle_17_redis_streams_progress_sampling_investigation.md`, `docs/production_inference_benchmark.md` |
| **Cycle 18** | Redis boundary-state cache for future sharding | **PHASE A CONTRACT ONLY / RUNTIME BLOCKED** | two-shard runtime failed identity/model-agreement gates; Redis may document boundary-state requirements only until a new identity-state design proof exists | `docs/cycle_18_redis_boundary_state_cache_investigation.md`, `docs/redis_broader_optimization_opportunities.md` |
| **Cycle 19** | Redis server-side scripts for measured read/compute/write hotspots | **CONDITIONAL** | only if Cycle 16.B leaves a measured Redis read/compute/write hotspot that pipelining cannot remove | `docs/redis_broader_optimization_opportunities.md` |
| **Cycle 20** | Streaming DB persistence and embedding overlap with inference | **20.E NOT ACCEPTED 2026-06-06** | 20.E fixed the r3 root causes by persisting `4541/4541` final-stable packets with `0` failures and Step 3 reconciled `0/4541`, but DB FPS regressed `5.86 %`, total elapsed regressed `6.22 %`, Step 2 through-pose wall regressed `12.29 %`, behavior call rate regressed `5.86 %`, and embedding stayed serial. Keep default production `OFFLINE_STREAM_POST_STAGES=0`, `OFFLINE_STREAM_POST_STAGE_TIMELINE=0`, and `OFFLINE_STREAM_POST_STAGE_MODE=inline_db`. | `docs/cycle_20_streaming_persistence_embedding_overlap_investigation.md`, `docs/production_inference_benchmark.md` §51 |
| **Cycle 21** | Celery worker/thread/concurrency scaling matrix | **PLANNED AFTER PARALLEL WORK EXISTS / PHASE A STAGED** | only if extra workers have independent work to consume; otherwise likely idle capacity or contention | `docs/cycle_21_celery_concurrency_scaling_investigation.md` |

Coordination note: the four active agent sessions are divided in
`docs/four_agent_cycle_coordination_board.md`. Agent 18 has released Cycle 17,
Agent 19 owns Cycle 18 design, and Agent 20 owns the remaining Cycle 20
readiness plus Cycle 21 governance lanes. Shared roadmap/benchmark files are
orchestrator-owned, and no agent may run a production benchmark without the
benchmark lock. Current turn ledgers are recorded in
`docs/agent_18_cycle_17_turn.md`, `docs/agent_19_cycle_18_turn.md`, and
`docs/agent_20_remaining_lanes_turn.md`; `AGENTS.md` must be updated when any
turn is taken or released.

### Z.4 Deferred decisions / out-of-scope here

| Topic | Status | Rationale | Doc |
|---|---|---|---|
| **YOLOE integration** | **DEFERRED until SLA reached or Cycle 14a lands** | Adding it now widens the SLA gap and would be redone after the BLS architecture change | `docs/new_models_yoloe_depth_anything_v2_timing_decision.md` |
| **Depth Anything v2 integration** | **DEFERRED** | Same reasoning as YOLOE; the smallest variant alone adds ~2.5 min of new GPU wall per `combined.mp4` | `docs/new_models_yoloe_depth_anything_v2_timing_decision.md` |

### Z.5 SLA contract recap

| Quantity | Value |
|---|---|
| Benchmark video | `combined.mp4` (4 541 frames, 2 m 31 s @ 30 fps) |
| SLA contract | `total_wall ≤ duration(video) + 5 min` |
| For `combined.mp4` | total wall ≤ **7 m 31 s** = 451 s |
| Required overall FPS | **≥ 10.07** |
| Current accepted baseline (Cycle 12.C single-inflight behavior overlap) | 15.6 min = **4.854 FPS** (gap: ~8.1 min) |
| Cycle 9 NOT ACCEPTED but produced | 18.51 min = 4.09 FPS (gap: 11.0 min) |
| Cycle 10 NOT ACCEPTED but produced | 17.93 min = 4.219 FPS (invalid for baseline because correctness regressed) |

Document: `docs/runtime_sla_video_plus_5min.md`.

### Z.6 Constitutional hard rules (re-affirmed across all cycles)

These mirror § E. If § E is updated, this section must be updated in the
same commit so the rules cannot drift between the TODO and the cycle map.

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
4. **CI/CD workflow updates are MANDATORY, not optional.** Every code
   change must update `.github/workflows/inference-parallelization.yml`
   in the same commit: new test files added to both the path filter and
   the pytest invocation; new modules added to the path filter; new
   `.env` flags exercised by a test that runs in CI. Commits that add
   code without updating the workflow are incomplete.
5. **No option, optimization, or strategy is accepted until it has been
   really tested, measured, and compared to the SLA Baseline on the
   production Linux server.** The three required artefacts are (a) bench
   summary JSON from `tools/prod/prod_run_parallel_flow_benchmark.sh`, (b)
   inference_audit JSON, and (c) GPU monitor CSV — all from the prod
   Linux RTX 5090 server. The metrics must be recorded in a row of the
   matching `docs/cycle_*_results.md` file alongside the prior accepted
   baseline and the SLA target row. No prod artefacts = STAGED, not
   ACCEPTED.
6. **Multi-approach items in this file (§B.1, §B.2, §B.3) must measure
   ALL their approaches on prod before one is selected.** A new
   `docs/cycle_9b_*_results.md` file captures the comparison matrix and
   is the source of truth for the tradeoff. Selection is `.env`-controlled
   so rollback is a single env flip.

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
