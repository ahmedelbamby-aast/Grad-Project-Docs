# Unified Telemetry Spine + Atomic Sub-Cycle Decomposition

**Created:** 2026-06-07 · **Branch:** `014-yoloe-scene-srvl`
**Supersedes the numbering in** `docs/pipeline_latency_optimization_plan.md` (the
Telemetry Spine becomes the new **Cycle 017**; the optimization cycles shift to
**018–023**, because you cannot optimize — or accept under §7.1.1 — what you
cannot measure per sub-stage).

> **Hard rule (user directive, constitutional under §1.6 / §7.1.1):** a single
> `postprocess_ms` block is **FORBIDDEN**. Every stage MUST decompose into named
> sub-stage spans down to the smallest measurable operation. One always-on layer
> records *everything* — benchmark, offline job, uploaded-video processing, and
> live streaming — with **zero per-call-site bespoke timing code**.

---

## Part A — Cycle 017: the Unified Telemetry Spine (the "metric watcher")

### A.0 What it is
One process-wide instrumentation layer. Any code block is wrapped once with a
`span(...)`; the layer auto-captures hierarchical timings, attaches the active
run context (mode/job/camera/frame), rolls up percentiles, and fans out to
durable + live sinks and a frontend stream — identically in **all** run modes.

```
with span("postprocess.mask_decode", objects=n):      # one line, anywhere
    masks = decode_masks(...)
# → nanosecond duration, parent = "frame", mode = live|offline|upload|benchmark,
#   scope = (job, camera, frame), auto-flushed to ring→Redis→Postgres→WS
```

### A.1 Span primitive — `backend/core/telemetry/spine.py`
- `Span` = (id, name, t_start_ns, t_end_ns, parent_id, attrs, status). `status ∈ {ok, error, unavailable}` (no silent zero — §1.6).
- Current-span stack via `contextvars` (async- and thread-safe; survives Celery prefetch and asyncio).
- `span(name, **attrs)` context manager + `@timed(name)` decorator — the only two public timing entry points.
- Monotonic `time.perf_counter_ns()`; never wall-clock for durations.

**Atomic sub-cycles (indivisible):**
- 017.1.1 Define `Span` + `SpanStatus` dataclasses.
- 017.1.2 contextvar current-span stack (push/pop).
- 017.1.3 `span()` context manager.
- 017.1.4 `@timed` decorator.
- 017.1.5 unit test: nesting depth, parent linkage, error→status=error, exception re-raise.

### A.2 Mode-agnostic run context
- `TelemetryContext(run_mode, job_id, session_id, camera_id, frame_index)`, `run_mode ∈ {benchmark, offline_job, upload, live}`, stored in a contextvar.
- `frame_scope(ctx)` context manager opens the root `frame` span and binds context; every child span inherits scope automatically. **This is the single ingestion point** — set once per frame, never per sub-stage.

**Atomic sub-cycles:**
- 017.2.1 `TelemetryContext` dataclass + contextvar.
- 017.2.2 `frame_scope()` root-span helper.
- 017.2.3 unit test: child spans inherit mode/scope without explicit passing.

### A.3 Decompose the forbidden `postprocess` block into sub-stage spans
Replace the single `postprocess_ms` with explicit spans (one `with span(...)` each — each wrap is atomic):
- 017.3.1 `postprocess.mask_decode`
- 017.3.2 `postprocess.nms`
- 017.3.3 `postprocess.box_decode`
- 017.3.4 `postprocess.gaze_decode`
- 017.3.5 `postprocess.posture_decode`
- 017.3.6 `postprocess.scene_normalize`
- 017.3.7 `postprocess.srvl_compute`
- 017.3.8 `postprocess.person_embedding` (OSNet pre/infer/post split)
- 017.3.9 `postprocess.contradiction_arbiter`
- 017.3.10 `persistence.detections` / `persistence.poses` / `persistence.embeddings` (per-writer spans)
- 017.3.11 also wrap `decode`, `preprocess`, `infer.<model>` so the whole frame is fully accounted (Σ children ≈ frame, residual = "unattributed" span — never hidden).

### A.3.1 Required derived metric — generated bounding-box throughput
Every metric surface (rollup, REST, live stream, figures) MUST expose
generated-bounding-box throughput in **all five dimensions**: **per frame, per
ms, per second, per call, per call_ms** (`core/telemetry/box_throughput.py`).
Boxes are sourced from spans tagged with a `boxes` count (e.g.
`postprocess.scene_normalize`, `infer.person_detector`). A dimension whose
denominator is missing/zero is `None`, never measured-zero (§1.6).

### A.4 Sinks (fan-out, all behind one flush)
- 017.4.1 **Ring buffer** — bounded in-proc deque (latest N spans) for live scrape + crash forensics.
- 017.4.2 **Redis live sink** — `XADD telemetry:{mode}:{job}:{camera}` stream (capped, §3.4 rebuildable) for realtime fan-out.
- 017.4.3 **Postgres durable sink** — batched insert into a new `SpanRecord` (or extend `NormalizedTelemetryRecord`); flush every K spans / T ms (off critical path).
- 017.4.4 **Rollup** — per (span_name, scope, window) → count, p50/p95/p99, measured-zero vs unavailable; for figures + frontend gauges.

### A.5 Frontend endpoints (ready for WebGL/WebGPU)
- 017.5.1 **REST** `GET /api/telemetry/spans` — windowed aggregates (filter mode/job/camera/span).
- 017.5.2 **Live stream** `GET /api/telemetry/stream` (SSE) **and** `ws://…/ws/telemetry/{scope}` (WS) — push each rollup tick.
- 017.5.3 **Binary metric-frame schema** — fixed-layout `Float32Array`/`ArrayBuffer` (column-major: [t, span_id, p50, p95, count, status]) so the browser uploads straight into a GPU buffer with no per-message JSON parse (the agreed WebGL/WebGPU path).
- 017.5.4 **OpenAPI/contract version** for the schema (§ contract authority).

### A.6 Frontend realtime renderer (WebGL/WebGPU)
- 017.6.1 TS types in `frontend/src/types/telemetry.ts`.
- 017.6.2 `useTelemetryStream` hook (WS/SSE → ring buffer → GPU buffer).
- 017.6.3 `TelemetryGpuRenderer` — WebGPU (fallback WebGL2) flame/stack + per-sub-stage latency lanes, updated at stream cadence.
- 017.6.4 Stall/stale/disconnect handling (show `stale`/`unavailable`, never fake liveness).

### A.7 Always-on wiring (one switch, every mode)
- 017.7.1 Open `frame_scope` at the **single** frame entry of each mode: offline `_on_scene_frame`/inference loop (`tasks.py`), benchmark command loop, upload pipeline, live consumer. Sub-stage spans already inside the shared functions then fire automatically in every mode.
- 017.7.2 Global enable flag `TELEMETRY_SPINE_ENABLED` (default on); sampling knob for live (`TELEMETRY_SPINE_SAMPLE_RATE`) so live streaming pays bounded cost.
- 017.7.3 Acceptance: run each mode, assert the span tree is identical-shaped and `postprocess` never appears as a leaf (only its children do).

**Cycle 017 exit:** every mode emits the full sub-stage span tree; the live WS/WebGPU view shows per-sub-stage latency in realtime; figures render per-sub-stage p50/p95; zero bespoke timing remains at call sites.

---

## Part B — Optimization cycles, recursively decomposed to atomic sub-cycles

Each leaf is **atomic = a single change + a single spine-measured before/after**.
None is accepted without the §12.5 full RTX-5090 benchmark **at frame stride = 1**
(every decoded frame inferred; stride > 1 = profiling-only, no authority —
§7.1.1 v2.12.0) + §7.1.1 figure bundle (now per-sub-stage, courtesy of Cycle
017). Every run is recorded in
[`docs/BENCHMARK_RESULTS_LEDGER.md`](BENCHMARK_RESULTS_LEDGER.md) with decision +
reason.

### Cycle 018 — Async / overlapped Triton dispatch (feed the idle GPU)
- 018.1 Enable `async_dispatch`.
  - 018.1.1 flip flag + plumb config — atomic.
  - 018.1.2 spine: confirm `infer.*` spans now overlap — atomic.
- 018.2 Batch per-frame crops into one request per model.
  - 018.2.1 person_detector batched request — atomic.
  - 018.2.2 gaze (h/v/depth) batched — atomic each (×3).
  - 018.2.3 posture batched — atomic.
  - 018.2.4 OSNet batch (verify already-batched ≤64) — atomic.
  - 018.2.5 preserve batch→(frame,identity) attribution (§5.4) + test — atomic.
- 018.3 Overlap dispatch with postprocess (bounded producer/consumer, §8.6).
  - 018.3.1 bounded queue between infer and postprocess — atomic.
  - 018.3.2 worker thread for postprocess — atomic.
  - 018.3.3 backpressure + ordering test — atomic.
- 018.4 Triton `dynamic_batching` tuning per model (queue delay / preferred sizes).
  - 018.4.x one model config at a time — atomic each.
- 018.5 Re-benchmark; accept on GPU-util↑ + infer-wall↓ with parity.

### Cycle 019 — GPU / vectorized postprocess (kill the 66 %)
- 019.1 mask decode on GPU (torch, no per-crop Python).
  - 019.1.1 batch `process_mask` on CUDA — atomic.
  - 019.1.2 numerical parity diff vs baseline — atomic.
- 019.2 NMS via torchvision GPU.
  - 019.2.1 swap NMS — atomic.
  - 019.2.2 parity — atomic.
- 019.3 gaze decode vectorized — 019.3.1 vectorize, 019.3.2 parity.
- 019.4 posture decode vectorized — 019.4.1, 019.4.2.
- 019.5 scene normalize: remove per-object Python loop — 019.5.1 vectorize, 019.5.2 parity.
- 019.6 SRVL hot path: confirm no Python pair-loop (already vectorized) + extend — atomic.
- 019.7 aggregate correctness-agreement table vs frozen baseline — atomic.

### Cycle 020 — Pose kinematics off the CPU critical path
- 020.1 move pose-kinematics compute to GPU/vectorized batch — 020.1.1 port, 020.1.2 parity.
- 020.2 run on a side lane, join at persistence (not in per-frame serial path) — atomic.
- 020.3 spine: `infer.rtmpose` + `postprocess.pose_kinematics` no longer serialize — atomic.

### Cycle 021 — Decode + preprocess pipelining
- 021.1 producer thread: decode + letterbox + tensor build — atomic.
- 021.2 pinned host buffers / reuse — atomic.
- 021.3 bounded frame queue (§8.6) + backpressure — atomic.
- 021.4 (optional) NVDEC GPU decode under CUDA 12.8 — 021.4.1 spike, 021.4.2 adopt-or-drop.

### Cycle 022 — Governed concurrency scaling (§8.1.1)
- 022.1 declare queue topology + resource ceilings + duplicate-worker check — atomic.
- 022.2 raise offline worker/thread concurrency one step — atomic.
- 022.3 measure GPU/DB/Redis contention via spine — atomic.
- 022.4 repeat 022.2/022.3 until GPU-util plateaus — atomic loop.

### Cycle 023 — Persistence / Redis / DB batching
- 023.1 bulk Postgres insert for detections — atomic.
- 023.2 bulk insert poses — atomic.
- 023.3 bulk insert embeddings + Redis pipeline — atomic.
- 023.4 move persistence off the per-frame critical path — atomic.

---

## Part C — Sequencing & acceptance
1. **017 first** (the spine) — without per-sub-stage truth, no optimization can be accepted (§7.1.1).
2. **018 → 019** next (feed GPU + cut CPU postprocess) — the bulk of ~23–35 min → ≤10 min.
3. 020–023 to mop up the remaining serial stages.
4. Acceptance per cycle = full `combined.mp4` all-models RTX-5090 run ≤ target FPS for that cycle **and** correctness parity table **and** the spine's per-sub-stage figure bundle. The live WebGPU view is the operator proof that the GPU is finally working.

## Part D — Streaming-safety (§8.6) per cycle
Each cycle ships a `streaming_compatibility` line. The spine itself is
live-safe (bounded ring + sampled live sink). Offline-only levers (whole-file
batching in 018.2/021) MUST NOT ship to the live profile without a separate
live-mode decision.
