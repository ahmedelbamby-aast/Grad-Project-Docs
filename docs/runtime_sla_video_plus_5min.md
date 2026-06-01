# Runtime SLA: `total_wall ≤ video_duration + 5 min`

**Status:** Plan adopted 2026-06-01. Acceptance only after each cycle is measured on prod with before/after numbers, per the constitution.

---

## 1. The contract

For any uploaded video `V`:

```
total_wall(upload → DB status=completed) ≤ duration(V) + 5 min
```

For the canonical benchmark `combined.mp4`:
- frames = 4 541
- duration = 4 541 / 30 fps ≈ **151.4 s (2 m 31 s)**
- target total wall = 151.4 + 300 = **451 s (7 m 31 s)**
- required throughput = 4 541 / 451 ≈ **10.07 FPS overall (DB-completed)**

Current state (Cycle 7, job `515fe118`):
- total wall = **1 582 s (26.37 min)**
- overall FPS (DB-completed) = **2.87**
- gap = **3.51×** speed-up required

---

## 2. Where the 1 582 s lives today (measured, not estimated)

| Stage | Cycle 7 wall (s) | % of total | Per-frame avg (ms) |
|---|---:|---:|---:|
| Step 2 — frame inference (Triton offline) | **842.6** | 53.3 % | 185.6 |
| Step 2 — pose post-processing | **220.6** | 13.9 % | 48.6 |
| Step 3 — persistence | 42.4 | 2.7 % | 9.3 |
| Step 4 — render (annotated + pose video) | 25.8 | 1.6 % | 5.7 |
| Step 5 — embedding generation | **450.7** | 28.5 % | 99.3 |
| **TOTAL** | **1 582.1** | 100 % | 348.4 |

The four bold blocks together are 96 % of the wall. **Persistence and render are within budget already.**

---

## 3. Per-stage budget to hit 451 s total

The 451 s target distributes roughly as:

| Stage | Cycle 7 actual | **Target SLA** | Required Δ | Achievable by |
|---|---:|---:|---:|---|
| Step 2 frame inference | 842.6 s | **≤ 300 s** | **−64 %** | Cycles 9–10: Triton ensemble/BLS + smaller behavior input |
| Pose post | 220.6 s | **≤ 60 s** | **−73 %** | Cycle 10: parallelize across frames OR inline during Step 2 |
| Persistence | 42.4 s | **≤ 30 s** | −29 % | Cycle 11: bulk_create + COPY-FROM |
| Render | 25.8 s | **≤ 15 s** | −42 % | Cycle 11: parallel writers (annotated + pose in parallel) |
| Embedding | 450.7 s | **≤ 50 s** | **−89 %** | **Cycle 8**: track-level reuse + bulk_create + lazy cv2 read |
| **TOTAL** | **1 582.1 s** | **≤ 455 s** | −71 % | All cycles 8–11 |

The largest single block to attack first is **embedding** (saves ~400 s of the 1 131 s gap = 35 % of the entire gap in one cycle), then **Step 2** (saves ~540 s = 48 %).

---

## 4. Per-stage hypotheses & measured-cost-budgets

### 4.1 Embedding (Cycle 8 target) — 450.7 s → ≤ 50 s

**Where the 450.7 s actually goes** (counts from job `515fe118`):

- 72 579 embeddings produced × 6.21 ms/embedding average.
- Per embedding: cv2 video seek (frame-level) + `model.embed([crop])` (detection-level) + DB INSERT + 3 Redis writes.

**Three independent wins, all already wired into the codebase but turned OFF:**

1. **Track-level embedding reuse** (`OFFLINE_EMBEDDING_REUSE_BY_TRACK=1`)
   - Code already in `apps/video_analysis/tasks.py:generate_embeddings`:
     ```python
     reuse_by_track = bool(getattr(settings, 'OFFLINE_EMBEDDING_REUSE_BY_TRACK', False))
     if reuse_by_track:
         vector = get_cached_job_track_embedding(job_id, track_id)
     ```
   - Without reuse: 72 k `model.embed()` calls + 72 k cv2 seeks.
   - With reuse: ~57 unique tracks (StudentTrack count from the job) → **57 `model.embed()` calls** + 72 k Redis GET hits.
   - DB writes stay 72 k (the contract requires a FrameEmbedding row per Detection).
   - **Expected stage wall: ~250–300 s saved.**

2. **Lazy cv2 frame seek**
   - Today `cv2_cap.set(...)` + `.read()` runs once per Frame *before* iterating its detections. If `reuse_by_track` covers all detections in that frame, the cv2 read is wasted.
   - Fix: only read the frame if at least one detection misses the track-reuse cache.
   - **Expected stage wall: 30–80 s saved on top of #1.**

3. **`FrameEmbedding` bulk_create in batches of 500–1000**
   - Today: `FrameEmbedding.objects.create(...)` per detection → 72 k single-row INSERTs.
   - Fix: batched create with `ignore_conflicts=False` (matches existing telemetry/frames pattern).
   - **Expected stage wall: 40–80 s saved on top of #1+#2.**

**Combined Cycle 8 target: 450.7 s → 50–80 s (−83 to −89 %).**

### 4.2 Step 2 — frame inference (Cycles 9–10) — 842.6 s → ≤ 300 s

**The Triton/GPU ceiling at this workload was measured at ~13 FPS** (probe sweep `tools/prod/probe_rtt_multiframe.py`, inflight=2). 4 541 frames / 13 fps = 349 s — close to the target.

To stay under 300 s we need to **reduce per-frame Triton compute**, not just orchestration:

- **Cycle 9 — Triton ensemble / BLS** for the 4 behavior models (posture, gaze h/v/depth):
  - Today: 4 separate gRPC calls per frame, each transferring the same 17×3×320×320 input tensor.
  - With ensemble: 1 gRPC call, 4 outputs, shared input on the server. Saves 3× the per-frame transport.
  - Expected step-2 wall: ~600 s (−29 %).

- **Cycle 10 — Smaller behavior input** (320 → 256 or 192):
  - 256² = 64 % of pixels; expected GPU compute roughly proportional.
  - Engine rebuild + accuracy validation gate required.
  - Combined with ensemble: ~300–400 s step-2 wall (−52 to −64 %).

### 4.3 Pose post-processing (Cycle 10) — 220.6 s → ≤ 60 s

Today pose runs SERIALLY across frames after Step 2 completes (4 541 frames × 48.6 ms each).

Two options:
- **Inline pose into Step 2**: dispatch pose alongside behavior models on each frame. Pipeline overlap absorbs pose latency. Risk: more concurrent Triton requests; cycle 6 already proved chunking works.
- **Parallelize across frames**: keep pose as a separate phase but process N frames concurrently with asyncio (similar to behavior batch queue). Lower risk, simpler change.

Either reduces pose wall by 3-4× → **≤ 60 s**.

### 4.4 Persistence + Render (Cycle 11)

Small absolute wins (~25 s combined), keep for last.

---

## 5. Ordered cycle plan

| Cycle | Target | Expected Δ overall wall | Confidence | Risk |
|---|---|---|---:|---|
| **Cycle 8** | Embedding: track-level reuse + lazy cv2 + bulk_create | **−350 to −400 s** | high | low (flag-gated, contract preserved) |
| Cycle 9 | Triton ensemble for 4 behavior models | −200 to −250 s | medium | medium (.pbtxt + parity test) |
| Cycle 10 | Pose parallelization OR inline | −150 to −180 s | medium | medium |
| Cycle 11 | Behavior input 320→256 | −100 to −150 s | medium-high | high (accuracy parity needed) |
| Cycle 12 | Persistence bulk-COPY + parallel render | −15 to −20 s | high | low |

Projected after Cycle 8: **~1 200 s (4.0 FPS overall)**.
Projected after Cycle 9: **~1 000 s (4.5 FPS overall)**.
Projected after Cycle 10: **~830 s (5.5 FPS overall)**.
Projected after Cycle 11: **~680 s (6.7 FPS overall)**.
Projected after Cycle 12: **~660 s (6.9 FPS overall)** — still 1.5× off the SLA.

That projection says **incremental cycles alone reach ~6–7 FPS, not 10 FPS**. To close the rest we need a step-change:

- **Cycle 13**: move pose, behavior fan-out, and embedding *all* onto the Triton server via BLS, returning compact tuples to Python. Replaces frame-synchronous orchestration with one round-trip per frame. Expected: ≥ 10 FPS overall.

Or, alternatively, decouple inference from rendering & embedding by **streaming**: as soon as a frame's behavior boxes land, emit to render and embedding tasks in parallel rather than waiting for Step 2 to finish for ALL frames. This is invasive but is the cleanest way to make wall ≈ duration(video).

---

## 6. Acceptance & constitutional discipline

Per the constitution and per the prior agreed protocol, **each cycle independently**:

1. Measures the relevant stage on a prod baseline.
2. Documents hypothesis + risk + rollback in `docs/crop_frame_optimization_execution.md`.
3. Implements **one** focused change with regression unit tests.
4. Runs `tools/prod/prod_run_parallel_flow_benchmark.sh` on `combined.mp4`.
5. Compares before/after FPS, stage wall, and correctness counters.
6. Marks ACCEPTED only if FPS improves AND every correctness counter is within 0.1 %.
7. Updates `docs/production_inference_benchmark.md` + `AGENTS.md` + `docs/inference_parallelization_plan.md` after acceptance.

No claim is made until prod evidence exists.

---

## 7. Next immediate action

**Cycle 8 — Embedding optimization.** Start with track-level reuse + lazy cv2 read, both already-implemented flags + a small code change to the embedding loop. Then bulk_create as a second change to keep cycles atomic.

See `docs/crop_frame_optimization_execution.md` Cycle 8 (to be added).
