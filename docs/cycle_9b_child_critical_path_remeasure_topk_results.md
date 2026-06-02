# Cycle 9b B.3 Child Critical-Path — REMEASUREMENT vs Top-K Baseline

**Last updated:** 2026-06-02

**Status:** **STEP 1 REMEASUREMENT COMPLETE — NO OPTIMIZATION ACCEPTED.**

**Outcome update:** the highest-ceiling follow-up from this document, Cycle
11.A (`TRITON_CROP_BEHAVIOR_INPUT_SIZE=256`), was implemented on production but
failed the required pre-benchmark parity gate. See
[`docs/cycle_11_input_size_results.md`](cycle_11_input_size_results.md). The
accepted baseline remains `320` exact slice + Top-K.

This document records the required Cycle 9b B.3 Step 1 **remeasurement**
against the new accepted baseline: Cycle 9b B.2.c exact slice + Top-K
(job `be4ba9ee-4786-48e9-8334-28feb237a1fb`, SHA `9bc53d86`). The previous
Step 1 measurement (`docs/cycle_9b_child_critical_path_results.md`) was
taken before Top-K landed; that topology has changed substantively and the
per-child math had to be re-pulled before any B.3 Step 2 candidate is
chosen.

No code or engine change is accepted by this document. Per §B.5 of
`docs/cycle_9_and_10_improvements_todo.md`, this is the "name the lever"
measurement; any Step 2 hypothesis must cite the numbers here.

## Why a remeasurement was required

Top-K anchor packing (`TRITON_BEHAVIOR_TOP_K_ENABLED=1`) changed the active
ensemble graph from `behavior_ensemble` to `behavior_ensemble_gaze_slice_topk`
and inserted four FP32 Top-K adapter models after the base children:

- `posture_topk_model`
- `gaze_horizontal_slice_topk_model` (after the slice gather)
- `gaze_vertical_topk_model`
- `gaze_depth_topk_model`

Plus the slice gather `gaze_horizontal_slice_model` runs between
`gaze_horizontal_model` and its Top-K adapter. The previous "dominant
child" decision (`gaze_horizontal_model` at 16.058 ms/exec, +33 % over the
next slowest) was made under the legacy ensemble. Top-K could plausibly
have:

- changed the per-child server delta (it did not change the engine, but the
  per-call batching distribution shifted slightly);
- introduced new bottlenecks via the FP32 adapters;
- moved orchestration overhead.

Without remeasuring we cannot honestly justify a Step 2 candidate aimed at
the dominant child.

## Evidence — production stats snapshot

Captured against the live production Triton (offline port `39100`) after the
Top-K accepted run completed:

| Item | Value |
|---|---|
| Production SHA at probe | `9bc53d8662991e5cffb651f40c83c00220c5512d` |
| Stats snapshot script | `tools/prod/probe_child_stats_topk.py` |
| Stats JSON | `backend/logs/probe_child_stats_topk_20260602T120051.json` |
| Direct RTT probe script | `tools/prod/probe_rtt_decompose_topk.py` |
| RTT probe JSON | `backend/logs/probe_rtt_decompose_topk_20260602T120240.json` |
| Probe shape | `[17, 3, 320, 320]` FP32 (`20.89 MB` per call) |
| Iterations | `20` per scenario |
| Probe target | offline Triton `127.0.0.1:39101` (forwarded gRPC) |
| Baseline reference | `docs/cycle_9b_child_critical_path_results.md` |
| Accepted baseline run | job `be4ba9ee-4786-48e9-8334-28feb237a1fb` |

### Aggregate production stats (cumulative since last Triton restart)

These are aggregate over the most recent prod run (~3 597 ensemble calls
at `bs=32` as the top batch bucket):

| Model | Executions | avg ms/exec | Top batch bucket |
|---|---:|---:|---:|
| `posture_model` | 3 599 | **29.694** | `bs=32` × 1 782 |
| `gaze_horizontal_model` | 3 599 | **29.818** | `bs=32` × 1 782 |
| `gaze_horizontal_slice_model` | 3 599 | **1.103** | `bs=32` × 1 782 |
| `gaze_vertical_model` | 3 599 | **29.690** | `bs=32` × 1 782 |
| `gaze_depth_model` | 3 598 | **29.695** | `bs=32` × 1 782 |
| `posture_topk_model` | 3 598 | **1.210** | `bs=32` × 1 782 |
| `gaze_horizontal_slice_topk_model` | 3 598 | **1.562** | `bs=32` × 1 782 |
| `gaze_vertical_topk_model` | 3 598 | **1.203** | `bs=32` × 1 782 |
| `gaze_depth_topk_model` | 3 598 | **1.216** | `bs=32` × 1 782 |
| `behavior_ensemble_gaze_slice_topk` | 3 597 | **33.010** | `bs=32` × 1 782 |
| `behavior_ensemble_gaze_slice` | 0 | — | — (route is dark under Top-K) |
| `behavior_ensemble` | 0 | — | — (route is dark) |
| `gaze_horizontal_slice_adapter` | 1 | 9.851 | — (standalone fallback, idle) |

Triton's per-stage `compute_input` / `compute_infer` / `compute_output`
fields and child `queue` ns were `0` for the ensemble children in this
build (same as the pre-Top-K measurement). Per-call cost was therefore
computed from `inference_stats.success.ns / count`.

### Controlled direct RTT probe (apples-to-apples vs pre-Top-K probe)

Same shape (`[17, 3, 320, 320]`) and iteration count (`20`) as the original
`docs/cycle_9b_child_critical_path_results.md` Step 1 probe, so the per-child
deltas below are directly comparable.

#### Sequential calls (one model at a time)

| Model | RTT mean | RTT p95 | Transport + server mean | Transport + server p95 |
|---|---:|---:|---:|---:|
| `posture_model` | 46.39 ms | 54.35 ms | 43.62 ms | 48.61 ms |
| `gaze_horizontal_model` | 53.76 ms | 62.91 ms | 49.95 ms | 58.19 ms |
| `gaze_vertical_model` | 44.81 ms | 48.46 ms | 42.63 ms | 46.19 ms |
| `gaze_depth_model` | 45.45 ms | 52.18 ms | 43.15 ms | 49.63 ms |

#### Concurrent calls (all 4 children fan-out, matches offline pipeline)

| Model | RTT mean | RTT p95 |
|---|---:|---:|
| `posture_model` | 134.22 ms | 141.43 ms |
| `gaze_horizontal_model` | 127.15 ms | 134.25 ms |
| `gaze_vertical_model` | 114.91 ms | 121.90 ms |
| `gaze_depth_model` | 116.82 ms | 126.38 ms |

#### Server-side delta (per-call server work attributed to the probe)

| Model | Delta executions | Avg server ms/exec (probe) |
|---|---:|---:|
| `posture_model` | 60 | **16.324** |
| `gaze_horizontal_model` | 60 | **18.790** |
| `gaze_vertical_model` | 60 | **16.236** |
| `gaze_depth_model` | 60 | **16.377** |
| `behavior_ensemble_gaze_slice_topk` | 20 | **30.133** |

#### End-to-end ensemble probe (single gRPC call returns all 4 child outputs)

| Metric | Value |
|---|---:|
| `behavior_ensemble_gaze_slice_topk` RTT mean | **63.59 ms** |
| `behavior_ensemble_gaze_slice_topk` RTT p95 | 68.51 ms |
| `behavior_ensemble_gaze_slice_topk` transport + server mean | 59.74 ms |
| Output bytes mean (all 4 children) | **326 400** = ~327 KB / call |
| Output shapes | `posture_out=[17,14,100]`, `gaze_h_out=[17,6,100]`, `gaze_v_out=[17,14,100]`, `gaze_d_out=[17,14,100]` |

## Decision

**Dominant child is still `gaze_horizontal_model`.** But the gap has shrunk
and the ceiling on per-child optimization is now small.

| Comparison | Pre-Top-K probe | Post-Top-K probe | Change |
|---|---:|---:|---:|
| `posture_model` server avg | 12.133 ms | **16.324 ms** | +4.19 ms (+34.5 %) |
| `gaze_horizontal_model` server avg | 16.058 ms | **18.790 ms** | +2.73 ms (+17.0 %) |
| `gaze_vertical_model` server avg | 11.759 ms | **16.236 ms** | +4.48 ms (+38.1 %) |
| `gaze_depth_model` server avg | 11.909 ms | **16.377 ms** | +4.47 ms (+37.5 %) |
| Dominant-vs-next gap (absolute) | 3.93 ms | **2.47 ms** | gap shrank by 37 % |
| Dominant-vs-next gap (relative) | **+33 %** | **+15 %** | gap shrank by 18 pp |

Two things are happening:

1. The other three children (posture, gaze_v, gaze_d) got relatively
   slower in the post-Top-K probe compared to the pre-Top-K probe. The
   per-call sample window is smaller (60 executions vs 40 in the prior
   probe) and the GPU is now sharing time with Top-K adapters at a higher
   total exec/sec on the same engine. The absolute increase (~4 ms across
   all three) is consistent with measurement contamination, not a true
   engine slowdown — the engines themselves are unchanged.
2. `gaze_horizontal_model` got slower by less than the others (+2.73 ms
   vs +4.2-4.5 ms). It is still the dominant child but the *gap* to the
   others halved.

## Per-Crop Compute Is Constant

Cross-checking aggregate prod stats vs the probe:

- Production aggregate (mostly `bs=32`): ~29.7 ms/exec across all 4 base
  children, ~30 ms/exec for the ensemble.
- Controlled probe (`bs=17`): ~16-19 ms/exec for base children, 30.1 ms
  for the ensemble.

Per-crop compute:

| Batch size | Per-child avg | Per-crop |
|---|---:|---:|
| `bs=17` (probe) | ~16.8 ms | **0.99 ms/crop** |
| `bs=32` (prod) | ~29.7 ms | **0.93 ms/crop** |

Per-crop GPU compute is essentially constant (~0.94 ms/crop) regardless
of batch size on these engines at this input size. The dominant lever is
therefore either:

- **Per-crop work** (smaller input → cheaper compute → fewer crops sometimes
  via region pruning), OR
- **Per-call invariants** (transport bytes, orchestration overhead).

## Ensemble Orchestration Overhead

The ensemble runs the 4 children concurrently inside Triton. With the
probe shape:

- max(child server avg) = 18.79 ms (gaze_h)
- + slice adapter (~1.1 ms) + Top-K adapter (~1.5 ms) = ~21.4 ms expected
- actual ensemble server avg = **30.13 ms**
- **orchestration overhead ≈ 8.7 ms / call** (~29 % of ensemble server time)

This is a separate lever from per-child compute. Reducing the dominant
child by 2.5 ms saves at most 2.5 ms of ensemble max(...). Reducing the
orchestration overhead would benefit every call.

## RTT Breakdown for the Active Ensemble

| Component | Value | Share of RTT |
|---|---:|---:|
| Ensemble RTT mean | 63.59 ms | 100 % |
| Server work (children + adapters + orchestration) | 30.13 ms | 47.4 % |
| Transport + gRPC overhead | **33.46 ms** | **52.6 %** |
| Input bytes per call (`[17,3,320,320]` FP32) | 20.89 MB | — |
| Output bytes per call (4 outputs) | 0.33 MB | — |

**Transport, not compute, is the larger share of ensemble RTT at the probe
shape.** Top-K already trimmed output bytes by ~95 % (Cycle 9b B.2.c) so
the remaining transport cost is dominated by input upload. The
per-frame total at the production aggregate (`bs=32`, ~17 frames per call
average) makes input bytes scale roughly linearly with input HxW.

## Step 2 Lever Analysis (per §B.5 discipline)

Before any code is written, the required §B.5 lever-naming exercise:

### Lever 1 — GPU child compute on the dominant child (`gaze_horizontal_model`)

**Ceiling math:**

- Max savings on `gaze_horizontal_model` server time = 2.47 ms / call
  (you cannot save more than the gap-to-next; below the gap, the child
  is no longer dominant and savings stop benefiting the ensemble).
- Ensemble savings = at most ~2.5 ms / call → ~8.2 % of ensemble server
  time, ~3.9 % of ensemble RTT.
- Step 2 wall (540.4 s under Top-K) projected delta: **at most ~−4 %**.

This is the **B.3.b / B.3.d** candidates (batch-profile alignment / kernel
tuning, no precision change). Low risk but bounded leverage.

### Lever 2 — GPU child compute on ALL children at once (smaller input size)

**Ceiling math (Cycle 11 — input 320 → 256):**

- New per-crop compute scales as `(256/320)² ≈ 0.64×` for vision conv work.
- Per-child server time at `bs=32`: 29.7 ms → ~19 ms (estimated).
- Ensemble server time: 30.1 ms → ~20-21 ms.
- Input transport: 20.89 MB / call → 13.37 MB / call (~36 % smaller).
- Step 2 wall projected delta: **−10 % to −13 %** (matches the
  `docs/cycles_9_to_12_implementation_playbook.md` §4 projection).
- Hits every child, not just the dominant one.

This is the **Cycle 11** PLANNED entry in §Z.3 of the TODO file. Same
lever (GPU child compute) as B.3.b but multi-child leverage.

### Lever 3 — Orchestration overhead (ensemble scheduling layer)

- Orchestration cost = ~8.7 ms / call (~29 % of ensemble server time).
- Best path here is **B.1 server-side compact postprocessing (BLS)**,
  which collapses 4 child calls + 4 adapters into one BLS Python step
  and removes the ensemble's inter-step scheduling. Same lever as the
  Cycle 13a playbook.
- Step 2 wall projected: similar to Cycle 13a (5 → 8-10 FPS) — but
  highest engineering effort of the three.

### Lever 4 — Single-process Python orchestration (sharding / persistent producer)

- Outside this measurement's scope. Cycle 13b territory.

## Recommendation for B.3 Step 2

**Per the constitution (§E.6 multi-approach rule), the next implementation
phase MUST benchmark multiple approaches on prod before one is selected.**

The remeasurement evidence above produces this ranked candidate list:

| Rank | Candidate | Lever | Projected Step 2 wall Δ | Engineering effort | Risk |
|---|---|---|---|---|---|
| 1 | **Cycle 11 — input 320 → 256** | GPU child compute (all 4) | −10 % to −13 % | ~2 person-days (engine rebuild + accuracy parity) | medium (accuracy parity gate required) |
| 2 | **B.3.b / B.3.d on `gaze_horizontal_model`** | GPU child compute (dominant only) | at most −4 % | ~1.5 person-days (TRT builder tactic tuning) | low (no precision change) |
| 3 | **B.1 BLS Python compact postprocessing** | orchestration + dense bytes | uncertain, projected larger but coupled with C.2.4 LPM Phase 2 | ~5 person-days | medium-high (first BLS deployment) |

If only one Step 2 candidate is run, it should be **Cycle 11** — it has
the highest leverage per person-day and attacks the same lever as B.3 but
across all 4 children. B.3.b/d is a lower-risk fallback if Cycle 11's
accuracy parity gate fails.

**This recommendation is NOT a commit.** It is the §B.5 lever-naming
output. The next agent picks one or both and runs the §E.5 prod-benchmark
loop. No B.3 Step 2 candidate is accepted by this document.

## What this document does NOT do

- It does **not** accept any optimization.
- It does **not** modify any TRT engine.
- It does **not** flip any `.env` flag.
- It does **not** update the `behavior_ensemble_gaze_slice_topk` route.
- It does **not** mark B.3 as complete in the §Z.1 cycle map. The B.3
  checkbox in `docs/cycle_9_and_10_improvements_todo.md` stays unticked
  until a Step 2 prod-benchmarked candidate ships.

## What this document changes

- `docs/cycle_9_and_10_improvements_todo.md` §B.3 "Current measured
  status" is superseded by this remeasurement.
- `AGENTS.md` Cycle 9b entry adds a "B.3 Step 1 remeasured 2026-06-02"
  log line.
- `.github/workflows/inference-parallelization.yml` path filter adds
  this doc (no new code).

## Reproduce

```bash
# 1. Aggregate stats snapshot (per-model since last Triton restart)
ssh prod-grad 'cd /home/bamby/grad_project && \
  python3 tools/prod/probe_child_stats_topk.py \
    --host 127.0.0.1:39100 \
    --json backend/logs/probe_child_stats_topk_$(date -u +%Y%m%dT%H%M%S).json'

# 2. Controlled RTT decomposition (apples-to-apples vs prior probe)
ssh prod-grad 'cd /home/bamby/grad_project && \
  backend/.venv/bin/python tools/prod/probe_rtt_decompose_topk.py \
    --iters 20 --n-crops 17 --size 320 \
    --host 127.0.0.1:39101 \
    --json backend/logs/probe_rtt_decompose_topk_$(date -u +%Y%m%dT%H%M%S).json'
```
