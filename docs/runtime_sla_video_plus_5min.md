# Runtime SLA: `total_wall ≤ video_duration + 5 min`

**Last updated:** 2026-06-02

**Status:** Plan adopted 2026-06-01. Latest accepted baseline updated
2026-06-02 after Cycle 9b exact slice + Top-K. Acceptance only after each cycle
is measured on prod with before/after numbers, per the constitution.

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

Current state (Cycle 9b exact slice + Top-K, job
`be4ba9ee-4786-48e9-8334-28feb237a1fb`):
- total wall = **1022.952 s (17.0 min)**
- overall FPS (DB-completed) = **4.429**
- gap = **2.27×** speed-up required

---

## 2. Where the ~1 023 s lives today (measured, not estimated)

| Stage | Cycle 9b exact slice + Top-K wall (s) | % of total | Per-frame avg (ms) |
|---|---:|---:|---:|
| Step 2 — frame inference (Triton offline) | **540.4** | 52.8 % | 119.0 |
| Step 2 — pose post-processing | **227.2** | 22.2 % | 50.0 |
| Post-pose audit stages to `run.complete` | 66.2 | 6.5 % | 14.6 |
| Embedding / DB completion tail | **189.1** | 18.5 % | 41.7 |
| **TOTAL** | **1023.0** | 100 % | 225.3 |

The bold blocks are still the dominant cost. Top-K reduced frame inference but
did not move pose or the DB completion tail.

---

## 3. Per-stage budget to hit 451 s total

The 451 s target distributes roughly as:

| Stage | Cycle 9b actual | **Target SLA** | Required Δ | Achievable by |
|---|---:|---:|---:|---|
| Step 2 frame inference | 540.4 s | **≤ 300 s** | **−44 %** | Cycle 13: compact postprocessing/BLS, sharding, or GPU-occupancy fix |
| Pose post | 227.2 s | **≤ 60 s** | **−74 %** | Cycle 10b: parallelize across frames OR inline during Step 2 |
| Post-pose audit stages | 66.2 s | **≤ 45 s** | −32 % | Cycle 12: bulk persistence/render writers |
| Embedding / DB completion tail | 189.1 s | **≤ 50 s** | **−74 %** | Cycle 12/13: streaming or server-side embedding contract |
| **TOTAL** | **1023.0 s** | **≤ 451 s** | −56 % | cycles 10b/12/13 plus one more Step 2 architecture win |

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
| Cycle 9b remaining | Compact postprocessing / dominant-child variants / larger batch window | Must be measured | medium | medium-high |
| Cycle 10b | Pose parallelization OR inline pose | −150 to −180 s | medium | medium |
| Cycle 11 | Behavior input 320→256 | −100 to −150 s | medium-high | high (accuracy parity needed) |
| Cycle 12 | Persistence bulk-COPY + parallel render | −15 to −25 s | high | low |
| Cycle 13 | BLS or multi-process architecture decision | step-change required | medium | high |

Historical accepted wins: Cycle 8 reduced embedding wall from `450.7 s` to
`~174 s`; Cycle 9 was not accepted; Cycle 9b exact slice reduced Step 2 wall
from `858.1 s` to `573.927 s`; Cycle 9b exact slice + Top-K reduced it again
to `540.399 s` but did not improve average GPU utilization.

Even after Cycle 9b exact slice + Top-K, the pipeline is still **~9.5 minutes over** the
SLA. Incremental cycles may not reach 10 FPS alone. To close the rest we need a
step-change:

- **Cycle 13**: move pose, behavior fan-out, and embedding *all* onto the Triton server via BLS, returning compact tuples to Python. Replaces frame-synchronous orchestration with one round-trip per frame. Expected: ≥ 10 FPS overall.

Or, alternatively, decouple inference from rendering & embedding by **streaming**: as soon as a frame's behavior boxes land, emit to render and embedding tasks in parallel rather than waiting for Step 2 to finish for ALL frames. This is invasive but is the cleanest way to make wall ≈ duration(video).

---

## 6. Acceptance & constitutional discipline

**Non-negotiable rule, re-affirmed 2026-06-01:** no optimization cycle may
be marked accepted/success/agreed without a production benchmark on the
Linux RTX 5090 server that demonstrates (a) the targeted metric improvement
and (b) zero correctness regression vs. the prior accepted baseline. Code
review, local testing, theoretical reasoning, parity probes, and
correctness counter matches are necessary but never sufficient. The
**Cycle 9 precedent** stands as the reference: parity passed, code
reviewed, FPS improved, but the designated acceptance gate (Step 2 wall
reduction ≥ 10 %) failed, so the cycle was held back even though the
change is technically working.

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

Use Cycle 9b exact slice + Top-K as the accepted baseline. The next cycle must
attack one of the remaining measured costs: server-side compact postprocessing /
GPU occupancy, pose parallelization, or the Cycle 13 architecture decision.
Do not add new intelligence models until the SLA gap is closed.
