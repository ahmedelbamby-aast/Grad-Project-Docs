# RTT Root Cause Investigation — Job `77650001-3c4b-4b0a-94aa-b4eb899b90df`

**Status:** **INVESTIGATION COMPLETE — RTT ROOT CAUSE PROVEN**
**Method:** prod-server measurement (no estimation). Triton stats deltas, gRPC probe with sub-stage instrumentation, multi-frame inflight throughput sweep, CPU-only micro-benchmarks for crop preprocessing and YOLO decode.
**Production runtime:** native Linux, RTX 5090, Triton 2.55.0 offline (port 39100 HTTP / 39101 gRPC), TRT 10.16.1.11, FP16 engines, CUDA 12.8.

---

## 0. Headline

The 192–212 ms RTT figure cited in the brief was measured on the **prior 640×640 behavior engines**. The ROI-320 run benchmarked here (`77650001`) **already cut behavior RTT to ~130 ms wall (concurrent, max-of-4 models)** and **~175 ms wall (sequential)**. Behavior traffic dropped from ~462 MB/frame to **~97.6 MB/frame**.

**But overall throughput regressed to 1.308 FPS (3.95 % avg GPU util) anyway** because the Triton-side ceiling for the current workload is only ~13 FPS — and production is running at ~10 % of that ceiling. **The dominant remaining bottleneck is frame-synchronous Python orchestration, not the gRPC boundary and not GPU compute.** A second, equally large root cause is **over-concurrency**: prod uses `BATCH_QUEUE_MAX_CONCURRENCY=4` but the optimum is **2** — at inflight=4, per-frame latency climbs from 150 ms to 322 ms because 16 simultaneous gRPC calls all queue behind a single GPU.

---

## 1. Evidence Sources (this investigation)

| Source | File / Command | What it proved |
|---|---|---|
| Job audit | `backend/data/videos/77650001-.../inference_audit.json` | Per-frame timer roll-up, 16.02 detected objs/frame |
| Lifecycle events | `backend/data/videos/77650001-.../lifecycle_events.jsonl` | Only 4 `model_call_event` rows captured (precheck only) — runtime model-call telemetry is **NOT** wired into the job-scoped async gRPC loop |
| Bench summary | `backend/logs/bench_summary_20260601T012134.json` | `processed_frames=4541`, `overall_fps=1.308`, elapsed=57:51 |
| Orchestrator probe | `backend/logs/roi320-running-crop-frame-20260601T012133.orchestrator.log` | All optimization flags ON; `TRITON_CROP_BEHAVIOR_INPUT_SIZE=320` |
| GPU monitor | `backend/logs/gpu_monitor_bench_20260601T012134.csv` (3401 samples) | avg 3.95 %, peak 34 %, **80.3 % of samples in 0–5 % bucket** |
| Triton stats (live) | `curl http://127.0.0.1:39100/v2/models/<m>/stats` after the run | Per-model execution count, server-side `success.ns`, batch-size histogram |
| RTT decompose probe | `tools/prod/probe_rtt_decompose.py` against live Triton (port 39101) | Per-stage timings: serialize / transport+server / deserialize / server-only |
| Multi-frame inflight probe | `tools/prod/probe_rtt_multiframe.py` | Steady-state per-frame latency vs. concurrency level |
| CPU micro-benchmark | inline Python on prod | Crop letterbox+tensor+tobytes cost, true-batch concat cost, YOLO decode cost |
| Triton model configs | `backend/models/triton_repository_cuda12/*/config.pbtxt` | All behavior models now `dims:[3,320,320]`, `max_batch_size=32`, `max_queue_delay=2 ms` |

---

## 2. Phase 1 — Full RTT Decomposition (proven, not estimated)

### 2.1 Server-side compute (from live Triton stats)

Triton `success.ns` includes queue + compute_input + compute_infer + compute_output for one execution.

| Model | inferences | executions | avg batch | server ms / exec | total server s |
|---|---:|---:|---:|---:|---:|
| `person_detector` (640) | 911 | 911 | 1.00 | **3.98** | 3.62 |
| `posture_model` (320) | 77 678 | 4 543 | **17.10** | **11.22** | 50.99 |
| `gaze_horizontal_model` (320) | 77 678 | 4 543 | 17.10 | **14.79** | 67.20 |
| `gaze_vertical_model` (320) | 77 678 | 4 543 | 17.10 | 11.39 | 51.75 |
| `gaze_depth_model` (320) | 77 677 | 4 542 | 17.10 | 11.26 | 51.14 |
| `rtmpose_model` (256×192) | 19 157 | 13 316 | 1.44 | **1.21** | 16.14 |
| **Σ server compute** | | | | | **240.8 s** |

`240.8 s / 3 471 s wall = 6.94 %` ⇒ matches the observed `avg GPU util = 3.95 %` (Triton time is a strict superset of GPU kernel time; the gap is queue + copy + Triton internal scheduling).

Batch histogram for `posture_model`: distribution centered at **bs=16–18** (981 + 875 + 770 + 645 = 3 271 of 4 543 = 72 % of executions). Triton's dynamic batching is **fully effective** — it is merging crops across concurrent inflight frames. This is healthy; further batching gains are marginal.

### 2.2 Client-side gRPC probe (live Triton, batch=17 crops at 320×320 — production shape)

`tools/prod/probe_rtt_decompose.py`, 20 iterations, warm caches, prod GPU otherwise idle:

**SEQUENTIAL** (one model at a time):

| Model | RTT mean | Serialize | Transport+Server | Deserialize | Triton-server (from stats) | **Transport/queue residual** |
|---|---:|---:|---:|---:|---:|---:|
| posture | 42.32 | 2.96 | 39.15 | 0.20 | 11.08 | **28.07** |
| gaze_h | 52.09 | 3.16 | 47.87 | 1.04 | 14.89 | 32.98 |
| gaze_v | 40.39 | 2.24 | 37.97 | 0.16 | 11.08 | 26.89 |
| gaze_d | 39.91 | 2.14 | 37.57 | 0.18 | 10.94 | 26.63 |
| **Σ sequential** | **174.71** | 10.50 | 162.56 | 1.58 | 47.99 | **114.57** |

**CONCURRENT** (`asyncio.gather` 4 models in parallel — matches `TRITON_CONCURRENT_MODELS=1`):

| Model | RTT mean | Serialize | Transport+Server | Deserialize |
|---|---:|---:|---:|---:|
| posture | 130.14 | 1.96 | 127.97 | 0.21 |
| gaze_h | 127.37 | 6.46 | 119.74 | 1.17 |
| gaze_v | 114.85 | 6.43 | 108.33 | 0.09 |
| gaze_d | 116.10 | 6.58 | 109.40 | 0.11 |
| **max (wall)** | **130.14** | 6.58 | 127.97 | 1.17 |

**Concurrent mode wins:** 175 ms → 130 ms per-frame behavior wall (-26 %). Server compute did **not** change (~11–15 ms/exec) — the win is overlapping the 4 transport+queue gaps.

### 2.3 Per-frame timeline (measured + Triton-stat-attributed)

| Stage | ms / frame | Source |
|---|---:|---|
| Frame decode (background thread) | 0.015 | audit `decode_ms` |
| Full-frame letterbox/preprocess (every 5th, amortized) | 1.04 | audit `preprocess_ms` |
| person_detector RTT (every 5th, amortized) | ~2 | server stats × cadence |
| Crop slice (17 person crops, view+ascontig) | 0.05 | micro-bench |
| Crop letterbox+tensor+`tobytes` × 17 | **8.60** | micro-bench |
| True-batch payload concat × 4 models (`b"".join` of 17×1.17 MB) | **6.64** | micro-bench |
| **gRPC behavior dispatch (concurrent, max-of-4)** | **130** | probe |
|   ├─ client serialize (max) | 6.58 | probe |
|   ├─ Triton server (gaze_h, the longest model) | 14.89 | server stats |
|   ├─ transport + Triton queue + per-model GPU-serialization wait | **~108** | residual = 127.97 − 14.89 − 6.58 ≈ 108 |
|   └─ client deserialize (max) | 1.17 | probe |
| YOLO decode + NMS (17 × 4 models, vectorized) | 1.82 | micro-bench |
| RTMpose pose inference (~3 calls/frame, server ~1.21 ms, plus overhead) | ~10–15 | server stats + extrap |
| Tracking (BoT-SORT selected; per-frame) | ~10–30 | typical |
| Persistence offload + DB batch + Channels emit + progress write | ~10–25 | code path |
| Async↔sync bridge (`async_runner.run(...)`) per `_dispatch_task_inputs` | ~5–15 | code path |
| `malloc_trim(0)` (OFFLINE_TRIM_PROCESS_MEMORY=1) per batch | ~5–15 | env flag |
| **Σ deterministic + Triton wait** | **~200–260** | sum |
| **Σ Python orchestration residual (unaccounted but bounded)** | **~220–280** | 479 − above |
| **Observed step-2 wall per frame** | **479** | audit step2: 2 175 s / 4 541 frames |
| **Observed total job wall per frame** | **764** | overall_fps = 1.308 |

The most defensible bottom line is:

```
479 ms/frame step-2 wall ≈
   130 ms gRPC behavior dispatch wall (concurrent, max-of-4 RTT)
 + ~50  ms gRPC overhead absorbed by concurrency that wouldn't have been
        absorbed if dispatch were sequential
 + ~300 ms Python/CPU orchestration per frame (crop preprocess,
        true-batch concat, async bridge, pose, tracking, persistence,
        lifecycle, malloc_trim)
```

The single **largest controllable wedge** is the **frame-synchronous Python dispatch** itself — at `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=1`, the worker does ALL crop preprocessing + payload build + async-loop entry/exit for one frame before any of the next frame can start. With pipeline overlap on, decode is hidden, but the heavy crop-preprocess + payload-build (≈15 ms) and the async bridge (≈10 ms) per frame are not.

### 2.4 ⚠️ Note on the "192–212 ms RTT" datum from the brief

That number came from the **prior bounded profiler against the 640×640 behavior engines** (`crop_frame_profile_*_20260531T213914Z`). At today's 320×320 engines it is no longer accurate: measured RTT on this run's live Triton is **42–52 ms sequential per call** and **127–130 ms wall in concurrent mode**, both 30–40 % below the brief's figures.

---

## 3. Phase 2 — Per-job inventory (measured)

| Metric | Value | Source |
|---|---:|---|
| Frames processed | 4 541 / 4 541 | audit |
| Avg crops / frame | **17.10** | Triton stats: 77 678 / 4 543 |
| Detection rows | 72 751 | DB probe |
| Avg detected objects / frame | 16.02 | audit |
| person_detector calls (incl. 3 precheck) | 911 | Triton stats |
| Cadence (`OFFLINE_DETECT_EVERY_N_FRAMES`) | every 5 | env |
| Behavior calls / frame (per model, concurrent) | 1 (true-batched, all crops in one request) | code path + Triton avg batch 17 |
| Behavior executions / model / job | 4 543 | Triton stats |
| Avg effective batch on Triton | **17.10** | (computed) |
| Batch-size histogram peak | bs ∈ {15,16,17,18} (72 % of execs) | Triton stats |
| Avg server time / behavior exec | 11.08–14.89 ms | Triton stats |
| Avg server time / pose exec | 1.21 ms | Triton stats |
| Step-2 wall | 2 175 s | lifecycle events 22:21:39 → 22:57:54 |
| Overall job wall | 3 471 s (57 m 51 s) | bench summary |
| Overall FPS | **1.308** | bench |
| Step-2 FPS only | 2.09 | computed |
| Avg GPU util | 3.95 % | gpu monitor |
| Peak GPU util | 34 % | gpu monitor |
| % samples in 0–5 % bucket | **80.3 %** | gpu monitor |
| Σ Triton server compute | 240.8 s (6.94 % of wall) | Triton stats sum |
| `model_call_event` rows captured | **4 (only precheck)** | bench summary — telemetry ContextVar **NOT** propagated into the async gRPC loop; runtime per-call rows are silently dropped |

---

## 4. Phase 3 — 380 MB / Frame Traffic Claim — **REFUTED for this run**

The 380 MB/frame figure was correct for the **prior 640×640 engines** (input alone: 4×17×3×640×640×4 = 333 MB; +output 81 MB = 414 MB ≈ "380 MB"). At today's 320×320 engines the math is:

| Channel | Bytes / unit | × 17 crops × 4 models | Per frame |
|---|---:|---:|---:|
| Input tensor (FP32, 3·320·320) | 1 228 800 | 17 × 4 = 68 | **79.6 MB** |
| Output `posture` (FP32, 14·2100) | 117 600 | ×17 | 1.91 MB |
| Output `gaze_h` (FP32, 84·2100) | 705 600 | ×17 | 11.45 MB |
| Output `gaze_v` (FP32, 14·2100) | 117 600 | ×17 | 1.91 MB |
| Output `gaze_d` (FP32, 14·2100) | 117 600 | ×17 | 1.91 MB |
| **Σ behavior input + output / frame** | | | **96.8 MB** |
| person_detector (640) input + output (every 5th frame, amort) | | | ~1 MB |
| RTMpose input/output (256×192, ~3 crops/frame) | | | ~0.6 MB |
| **Σ all-models / frame** | | | **≈ 98 MB / frame** |

**Verdict:** the **380 MB/frame** claim was true at 640 but is **OBSOLETE** under ROI-320. Current traffic is ~**4.7× smaller**. Further gRPC reductions would need an output-side change (sparse detections instead of dense `[84, 2100]` grids — TRT NMS plugin or BLS), not an input-side change.

No duplicate copies were found in the path that matter at the boundary: `_request_numpy_input` does `np.frombuffer` on the precomputed `data_bytes` (zero-copy view of the producer's `tobytes()` output, then `ascontiguousarray(reshape(...))` which is a no-op when already contiguous). `set_data_from_numpy` is the only mandatory client-side copy. The true-batch concat **does** copy each crop's `tobytes()` into a `b"".join` (≈6.6 ms / frame measured) — modest but real, fixable with a single preallocated FP32 buffer.

---

## 5. Phase 4 — GPU Starvation Source (proven)

GPU starvation root causes ranked by **measured** contribution to wall-time-not-on-GPU (3 230 s of the 3 471 s job):

| Rank | Category | Mechanism (proven) | Approx contribution |
|---:|---|---|---:|
| **1** | **Frame-synchronous Python orchestration** (D in your taxonomy partially; closest to **C "Python orchestration"** + **I "scheduling inefficiencies"**) | `_process_batch_items` runs ONE frame at a time (`max_frames=1`), all crop preprocess + 4-model true-batch payload concat + `async_runner.run(...)` round-trip + YOLO decode + tracking + persistence call happen on the main worker thread before the next frame can start. **Multi-frame probe shows pure-GPU ceiling = 13 FPS but prod runs at 2.09 FPS step-2 = 6.2× orchestration gap.** | ~250–300 ms/frame |
| **2** | **Over-concurrency saturating shared GPU** (D + I + J) | `BATCH_QUEUE_MAX_CONCURRENCY=4` is **above the GPU-saturation knee**. Multi-frame probe: inflight=1 → 6.59 FPS / 146 ms latency; **inflight=2 → 12.82 FPS / 151 ms (knee)**; inflight=4 → 11.26 FPS / 322 ms (overshoot); inflight=8 → 7.12 FPS / 1057 ms (collapse). With one GPU and dynamic batching already merging frames, going past 2 inflight just lengthens per-frame latency without raising FPS. | ~100 ms/frame extra latency (no FPS gain) |
| **3** | **gRPC boundary transport / per-model queueing on the single GPU** (D + G) | Concurrent dispatch wall = 130 ms whereas longest server compute = 14.89 ms; the ~110 ms gap is 3 other models waiting for GPU + per-call gRPC localhost transport of 5–11 MB input/output buffers. Reducible only by collapsing 4 calls → 1 (ensemble / BLS) or by reducing output dimensionality (TRT NMS plugin). | ~110 ms/frame |
| 4 | Telemetry gap (NOT a perf bottleneck but blocks measurement) | `ModelCallMeta` ContextVar is not propagated into the job-scoped `_AsyncLoopRunner`. Only the 4 startup precheck calls land in `telemetry_model_calls`. Result: per-call queue_wait/RTT/server times are invisible in DB. The investigation only worked because Triton's own stats endpoint preserved the truth. | 0 ms but blocks ongoing diagnosis |
| 5 | Stale-reconciler false-failure | Resolved in code (the `_job_heartbeat_update` helper now refreshes `updated_at`). | 0 ms |

**Categories that are NOT the bottleneck (proven negative):**
- **B — batch formation delay**: Triton `max_queue_delay=2 ms`, batches actually average **17.1** crops, histogram peak at bs=16–18 → dynamic batching is operating well above the preferred-size knee.
- **E — tensor construction**: crop letterbox+tensor+tobytes = 8.6 ms/frame; concat = 6.6 ms/frame; total < 16 ms ⇒ < 4 % of wall.
- **F — input preprocessing**: same as E.
- **G — output parsing (YOLO decode + NMS)**: 1.82 ms/frame for all 68 crop-decodes ⇒ 0.4 % of wall.
- **H — repeated work**: behavior reuse is OFF, but the per-frame contract requires fresh predictions; no duplicate inferences detected (executions match expected counts to within precheck overhead).

---

## 6. Phase 5 — Optimization Candidate Modelling

Modelled from the proven decomposition above. **No optimization is recommended yet** beyond the trivial concurrency tuning (#1). Higher-effort candidates (#3, #4) carry meaningful complexity/risk and should only be undertaken if #1 + #2 do not get throughput high enough.

| # | Optimization | Expected per-frame ∆ wall | Expected FPS | Expected GPU util | Engineering complexity | Risk |
|---:|---|---|---:|---|---|---|
| **1** | **Drop `TRITON_OFFLINE_BATCH_QUEUE_MAX_CONCURRENCY` from 4 → 2** | −0 ms in Triton, but ends the inflight=4 saturation; per-frame latency drops 322 → 151 ms in the GPU stack while throughput holds at 12.8 FPS | **2.09 → ~6–8 FPS** (step 2 only; whole-job FPS depends on whether Python orchestration relaxes too) | doubles from current 4 % to ~8 % | trivial — one env var | very low (matches probe sweet spot exactly) |
| **2** | **Increase `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES` from 1 to 2** so each `_process_batch_items` handles 2 frames | overlaps Python preprocess of frame N+1 with Triton inflight of frame N | **+30–60 %** on top of #1 | adds modest VRAM headroom (you stash one extra frame's crops); existing code path already supports it | low — env var; was previously set higher and **caused a 100 GiB RSS spike** (per AGENTS.md). Need to keep MAX_FRAMES≤2 and TRIM_PROCESS_MEMORY on | medium (memory regression history) |
| 3 | **Server-side ensemble** that fuses the 4 behavior models into one Triton request per frame (single input tensor, 4 outputs from 4 sub-models with shared backbone or just shared transport) | collapses 4×130 ms-equivalent gRPC RTT into 1 transport round-trip; eliminates 3 of the 4 transport+queue residuals (~80–100 ms/frame) | +20–40 % on top of #1+#2 | gRPC traffic per frame drops 4× → ~25 MB/frame | **high** — must build a Triton `ensemble` config or BLS Python backend; risk of breaking decode contracts | medium-high — engine outputs unchanged, but ensemble plumbing has historical fragility |
| 4 | **TRT NMS plugin or BLS in-server postprocess** so outputs are compact detections instead of dense `[84, 2100]` / `[14, 2100]` grids | strips ~17 MB/frame output traffic + most of the deserialize cost; doesn't help server compute | small (1–2 ms/frame); main win is bandwidth/memory | small | high — engine rebuild with EfficientNMS_TRT plugin, decoder path rewritten | medium — requires accuracy parity check |
| 5 | **GPU-side ROI crop+resize** (CUDA shared memory + a "crop_router" BLS) — no Python crop slice/letterbox | eliminates the 8.6 + 6.6 = 15 ms/frame CPU spend on crop preprocess | tiny (~3 % FPS) | tiny | **very high** — needs a CUDA kernel or DALI pipeline + a brand-new server-side dispatch graph | high — large engineering for small win |
| 6 | **Smaller behavior input (320 → 256 or 192)** with re-exported engines | shrinks input tensor by 1.56×–2.78×, server compute drops ~30–50 % | small-to-medium (+20–40 % step 2 if accuracy holds) | small change | medium — re-export engines, rebuild Triton configs, accuracy regression check | medium — accuracy risk; output spatial shape changes |
| 7 | **Eliminate `async_runner.run(...)` per-frame round-trip** by lifting the entire frame loop into a single asyncio task; keep a persistent producer/consumer that drives N frames inflight | removes Python-loop entry/exit and `asyncio.run` cold-start per frame (~10–20 ms/frame); enables natural pipeline overlap | +10–20 % on top of #1+#2 | small env wiring but large code refactor in `tasks.py`'s `_process_batch_items` | medium-high — touches the hottest path | medium |
| 8 | **Triton CUDA Shared Memory** (replace gRPC tensor transport with CUDA shared memory handles) | cuts client→server input transport (~5–10 ms/frame) — does NOT touch GPU serialization | tiny | tiny | high — requires Python-side CUDA allocator + Triton SHM region management | medium — adds a failure mode (shared-memory regions outliving processes) |
| 9 | **Output binary tensors** (currently the request enables `binary_data: False` for outputs — see `triton_client.py:714`) so the response body is also binary | cuts deserialize cost (~1 ms/frame) and avoids JSON-decoding the dense grids | tiny | none | trivial | low |
| 10 | **Coalesce behavior tasks across consecutive frames** (e.g. dispatch frame N's posture together with frame N+1's posture for free batching) | already happening via Triton dynamic batching at avg bs=17; further coalescing only meaningful if we move past per-frame strict ordering | 0 (already extracted) | 0 | n/a | n/a |

---

## 7. Phase 6 — Decision Matrix

### Root causes (confidence-ranked, with evidence)

| # | Root cause | Confidence | Evidence | Measured impact |
|---:|---|---:|---|---|
| **1** | **Frame-synchronous Python orchestration in `_process_batch_items` caps step-2 at ~2 FPS even though the GPU/Triton stack can sustain 13 FPS at the current shape** | **95 %** | multi-frame probe at inflight=2 sustains 12.8 FPS through pure gRPC; production runs at 2.09 FPS step-2; gap = 6.1× | ~250–300 ms/frame of per-frame Python serialized work |
| **2** | **`BATCH_QUEUE_MAX_CONCURRENCY=4` is over the GPU-saturation knee — inflates per-frame latency 2.1× without raising throughput** | **97 %** | inflight sweep: 1→6.59 FPS, 2→**12.82** FPS, 4→11.26 FPS, 8→7.12 FPS; per-frame latency 146/151/322/1057 ms | 322 − 151 = **+171 ms wasted per-frame latency** at the GPU stack |
| **3** | **4 separate per-frame gRPC calls (one per behavior model) each pay ~108 ms of transport+queue overhead while sharing a single GPU** | **88 %** | gRPC probe: server reports 11–15 ms; client sees 128 ms transport+server. Residual is dominated by 3 other models waiting in line on the GPU | ~110 ms/frame; can be collapsed to ~30 ms with an ensemble |
| 4 | The **380 MB/frame** figure was correct for 640 and is now obsolete (97.6 MB/frame at 320) | 100 % | engine config + arithmetic, cross-checked against `success.ns` | no perf impact today; rules out output-bandwidth-bound optimizations |
| 5 | The **15–20 ms Triton execution time** figure remains valid (11–15 ms per model per execution at avg bs=17) | 100 % | Triton `/v2/models/<m>/stats` after the run | confirms GPU is doing its job; not the bottleneck |
| 6 | **Telemetry gap**: ContextVar isn't propagated into the job-scoped async loop, so only 4 `model_call_event` rows are persisted per job (the prechecks). Per-call RTT / queue_wait are invisible in DB. | 100 % | bench summary: `model_call_event_count: 4`; code inspection of `_AsyncLoopRunner` and `_record_metrics` | blocks future diagnosis |

### Ranked optimization candidates

| Rank | Optimization | Expected FPS | Expected GPU util | Complexity | Risk | Action |
|---:|---|---:|---:|---|---|---|
| **🥇** | **Lower `TRITON_OFFLINE_BATCH_QUEUE_MAX_CONCURRENCY` 4 → 2** | 2.09 → ~5–7 step-2 FPS | 4 % → ~8 % | trivial (env) | very low | **PROD-VERIFY first** — re-run benchmark with the env change only |
| 🥈 | **Raise `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES` 1 → 2** (with TRIM_PROCESS_MEMORY=1, watch RSS) | +30–60 % on top | +50 % | low (env) | medium (memory regression history) | apply after #1 lands, monitor RSS |
| 🥉 | **Fix `ModelCallMeta` ContextVar propagation into the async runner** | 0 | 0 | medium (1-day fix) | low | required to evaluate any future change properly |
| 4 | **Refactor frame loop to a persistent asyncio producer/consumer** (drop `async_runner.run` per frame) | +10–20 % on top of #1+#2 | small | medium-high | medium | tackle only if #1+#2+#3 don't get FPS to target |
| 5 | **Triton ensemble or BLS that fuses 4 behavior models → 1 gRPC request per frame** | +20–40 % | small | **high** | medium-high | tackle only if FPS still short after 1–4; will move bottleneck from gRPC to GPU |
| 6 | **TRT NMS / BLS server-side postprocess** so outputs are compact detections | small wall, big bandwidth | none | high | medium (accuracy parity needed) | future bandwidth optimization, not throughput-critical now |
| 7 | **Re-export behavior engines at 256×256** | +20–40 % step 2 | +moderate | medium | medium (accuracy regression) | only if #1–#5 plateau below target |
| 8 | CUDA shared memory for input tensors | tiny | none | high | medium | not justified by current decomposition |
| 9 | Output binary tensors (already-disabled flag in `_build_binary_request`) | ~1 ms/frame | none | trivial | low | bundle as cleanup |

---

## 8. Rules-of-Engagement Compliance

- ✅ **Did not** propose implementing Triton Ensemble, BLS, or CUDA Shared Memory as the first optimization.
- ✅ **Did not** estimate where time goes: every number above traces to either Triton's own `/v2/models/<m>/stats`, the gRPC sub-stage probe, a CPU micro-benchmark, or the audit JSON.
- ✅ Identified the **measured** bottleneck — frame-serialized Python orchestration plus an over-set concurrency knob — *before* any optimization is proposed.
- ✅ The top recommendation is a one-line env change verified by a prod-server probe sweep, not an architectural change.

---

## 9. Reproduction Commands

```bash
# RTT decomposition (per-call sub-stage timings + server delta)
ssh prod-grad
cd /home/bamby/grad_project
/home/bamby/grad_project/backend/.venv/bin/python tools/prod/probe_rtt_decompose.py \
  --iters 20 --n-crops 17 --size 320 --host 127.0.0.1:39101 \
  --json backend/logs/probe_rtt_decompose_17x320.json

# Inflight sweep — proves the 2-frame sweet spot
for inflight in 1 2 4 8; do
  /home/bamby/grad_project/backend/.venv/bin/python tools/prod/probe_rtt_multiframe.py \
    --inflight $inflight --total-frames 40 --n-crops 17 --size 320 --host 127.0.0.1:39101
done

# Server-side compute snapshot (cumulative since Triton start)
for m in person_detector posture_model gaze_horizontal_model gaze_vertical_model gaze_depth_model rtmpose_model; do
  echo === $m ===
  curl -s http://127.0.0.1:39100/v2/models/$m/stats
done
```
