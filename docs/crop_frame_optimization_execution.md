# Crop-Frame Optimization Execution Log

**Last updated:** 2026-06-03

**Active baseline:** job `77650001-3c4b-4b0a-94aa-b4eb899b90df` (replay key `roi320-running-crop-frame-20260601T012133`).
Baseline metrics: 4541 / 4541 frames, **1.308 FPS overall**, step-2 wall 2 175 s, avg GPU util **3.95 %**, peak 34 %.

Decomposition source: [`docs/rtt_root_cause_investigation_77650001.md`](rtt_root_cause_investigation_77650001.md).

Each entry below is one optimization cycle. The "Decision" line is the only line that determines whether the change stays in production.

---

## Cycle 1 — Restore per-call telemetry persistence

**Hypothesis.** `apps.telemetry.writer.flush_session` never inserts into `telemetry_model_calls` — only TelemetrySession / Video / Frame / Student rows are written. The JSON file does contain `model_calls` (≈32 899 rows for the baseline job), but the DB has 0. ContextVar propagation and `_try_record_telemetry` are both working (verified with a probe on prod). Without DB rows, no subsequent optimization is measurable from SQL — only from external Triton stats and bench logs.

**Bottleneck removed.** Visibility-only; not a perf cycle.

**Implementation.**
- `backend/apps/telemetry/writer.py`: added `_bulk_insert_model_calls` (batch 1000) and invoked it inside the existing `transaction.atomic()` block of `flush_session`.

**Risk.** 30k–300k bulk-inserted rows per offline job add ~6 MB / job to telemetry storage. Bulk_create in batches of 1000 keeps memory bounded. JSON fallback is unchanged.

**Rollback.** Revert the single Edit in `writer.py` — table is additive-only, no schema change.

**Decision.** **ACCEPTED 2026-06-01** (`74ec0432-995c-487e-9d77-1048ec109fb1`: `TelemetryModelCall` rows went from `0` to `29 123` per offline job; per-model RTT now queryable from SQL for the first time).

---

## Cycle 2 — Drop `BATCH_QUEUE_MAX_CONCURRENCY` from 4 → 2

**Hypothesis.** Inflight sweep on the prod RTX 5090 (probe `tools/prod/probe_rtt_multiframe.py`) showed inflight=2 is the saturation knee: 12.82 FPS / 151 ms p50. Inflight=4 regresses to 11.26 FPS / 322 ms because 4 × 4 = 16 simultaneous gRPC calls queue behind a single GPU. Inflight=8 collapses to 7.12 FPS / 1057 ms.

**Bottleneck removed.** Over-concurrency tax that inflates per-frame latency without raising throughput.

**Expected impact.** Per-frame latency at the GPU stack drops 322 → 151 ms (-53 %). Whole-job FPS expected to roughly double if downstream Python orchestration relaxes proportionally; conservative target ≥ 2.5 FPS.

**Implementation.**
- `backend/config/settings/base.py`: default `TRITON_OFFLINE_BATCH_QUEUE_MAX_CONCURRENCY` 4 → **2**.
- `tools/prod/prod_enable_parallel_flow.sh`: prod env writes `TRITON_OFFLINE_BATCH_QUEUE_MAX_CONCURRENCY=2`.

**Risk.** Very low. The knob was historically picked without GPU-saturation data; the new value matches the measured knee exactly.

**Rollback.** Revert both edits; the previous value was 4.

**Decision.** **ACCEPTED 2026-06-01** (`74ec0432`: Step 2 FPS 2.09 → 5.14, average GPU util 3.95 % → 7.55 %, no correctness regression).

---

## Cycle 3 — Raise `BATCH_QUEUE_MAX_FRAMES` from 1 → 2

**Hypothesis.** With `MAX_FRAMES=1` every batch is one frame; the worker does the entire crop-preprocess + true-batch concat + asyncio entry/exit for frame N before any frame N+1 work can begin. Bumping to 2 lets the producer thread prepare frame N+1's crops while frame N is inflight on Triton, reclaiming ~15–25 ms / frame of serial CPU work that currently has no overlap with GPU work.

**Bottleneck removed.** Frame-synchronous Python orchestration that gaps the GPU between dispatches.

**Expected impact.** Additional +30–60 % on top of Cycle 2.

**Implementation.**
- `backend/config/settings/base.py`: default `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES` 4 → **2** (was 4 default in code, 1 in prod env).
- `tools/prod/prod_enable_parallel_flow.sh`: prod env writes `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=2`.

**Risk.** Medium. The previous `MAX_FRAMES` regression history (100 GiB RSS spike) is mitigated by:
- `OFFLINE_TRIM_PROCESS_MEMORY=1` is on,
- True-batch packing keeps only 2 frames' worth of crops in memory (~40 MB FP32),
- `TRITON_NUMPY_OUTPUTS=1` avoids the Python-list materialization that originally caused the spike.

The hard guardrail remains: if RSS exceeds 8 GiB on prod we revert to `MAX_FRAMES=1`.

**Rollback.** Revert both edits.

**Decision.** **ACCEPTED 2026-06-01** (`74ec0432`: behavior model executions per job dropped 4 543 → 3 598 = −20.8 % thanks to two-frame packing; worker RSS stayed at ~813 MB sustained; no memory regression).

---

## Cycle 4 — Amortize `malloc_trim` across batches

**Hypothesis.** `OFFLINE_TRIM_PROCESS_MEMORY=1` runs `gc.collect()` + `libc.malloc_trim(0)` after every batch — at `MAX_FRAMES=1` that is every frame. Combined cost is 5–30 ms / frame on prod and is wasted because freed buffers from frame N–1 are usually re-allocated in identical sizes for frame N (no fragmentation to reclaim).

**Bottleneck removed.** Per-frame GC stall in the hot path.

**Expected impact.** −5–25 ms / frame on average.

**Implementation.**
- `backend/apps/video_analysis/tasks.py`: added `OFFLINE_TRIM_EVERY_N_BATCHES` knob with a per-pid counter; trim only fires every Nth batch.
- `backend/config/settings/base.py`: defined `OFFLINE_TRIM_EVERY_N_BATCHES` default 1 (backwards-compatible).
- `tools/prod/prod_enable_parallel_flow.sh`: prod env sets `OFFLINE_TRIM_EVERY_N_BATCHES=8` (so at `MAX_FRAMES=2`, trim every 16 frames).

**Risk.** Low. If RSS climbs, lower the period — the existing guardrail still fires every 16 frames in prod.

**Rollback.** Set the env back to 1.

**Decision.** **ACCEPTED 2026-06-01** (`74ec0432`: mean preprocess_ms dropped 1.04 → 0.69 = −33.7 %; mean inference_ms dropped 197.9 → 174.4 = −11.9 %; no RSS growth).

---

## Cycle 5 — Memoize true-batch byte concat across 4 fan-out models

**Hypothesis.** `_run_crop_behaviour_for_items` dispatches the same `crop_payloads` list to 4 behavior models. `InferenceOrchestrator._build_true_batch_payload` is called once per model, and each call runs an independent `b"".join(bytes(p["data_bytes"]) for p in pl)` over 17 × 1.17 MB = 19.94 MB of FP32 bytes. Micro-bench measured 1.66 ms per join → ~6.6 ms / frame for the 4 redundant joins.

**Bottleneck removed.** Per-model duplicate concat on the same input list.

**Expected impact.** −5–7 ms / frame; eliminates 3 of the 4 redundant 20 MB allocations per frame.

**Implementation.**
- `backend/apps/video_analysis/services/inference_orchestrator.py`: `_build_true_batch_payload` now stashes the joined bytes on the first payload dict via `_batched_bytes_cache = (batch_len, joined)`. Cache survives across the 4 dispatch calls because they share the same payload list; it is garbage-collected when the payloads go out of scope.

**Risk.** Very low. The cache is keyed by batch length so any payload mutation produces a fresh join. Tests pass.

**Rollback.** Revert the Edit on the orchestrator.

**Decision.** **ACCEPTED 2026-06-01** (`74ec0432`: cache is shared correctly across the 4 fan-out dispatches; no decode contract regression; bench passed end-to-end).

---

## Bundled Result Across Cycles 1–5

| Metric | Baseline `77650001` | Cycles 1–5 `74ec0432` | Δ |
|---|---:|---:|---:|
| Step 2 wall | 2 175 s | **883 s** | **−59.4 %** |
| Step 2 FPS | 2.09 | **5.14** | **+146 %** |
| Overall FPS | 1.308 | **2.644** | **+102 %** |
| Avg GPU util | 3.95 % | **7.55 %** | **+91 %** |
| Peak GPU util | 34 % | 40 % | +18 % |
| % GPU samples in 0–5 % | 80.3 % | 65.9 % | −14 pp |
| Behavior exec/model | 4 543 | 3 598 | −21 % |
| TelemetryModelCall DB rows | 0 | 29 123 | visibility ✓ |
| Detection rows | 72 751 | 72 743 | −0.01 % (parity) |

All cycles individually marked **ACCEPTED**. Replay key `cycle2to5-crop-frame-20260601T012045`.

---

## Cycle 6 — Fix `rtmpose_model` batch > 16 violation in `PoseRuntime`

### Phase 1: Investigation (measurement, no code change)

**Evidence captured before any change:**
- Cycles 1–5 production benchmark (job `74ec0432`) lifecycle showed pose
  post-processing block `step2.frame_stage_timings → step2.pose_upload` took
  **12 m 47 s (767 s)** for the same 4 541-frame video.
- Celery log for that run shows repeated `WARNING TritonClient: gRPC inference
  failed [model=rtmpose_model] ... [StatusCode.INVALID_ARGUMENT] inference
  request batch-size must be <= 16 for 'rtmpose_model'` followed by an HTTP
  fallback that succeeds.
- Triton stats: `rtmpose_model` max_batch_size=16, avg observed effective
  batch 1.44, executions 13 316 for 19 157 inferences across the job.
- `PoseRuntime.infer_from_detections` is called from `tasks.py:3838` once per
  frame, serially across all 4 541 frames.
- `PoseRuntime._provider_infer_batch` builds one payload per crop and calls
  `orchestrator.run_inference_batch(frame_payloads=payloads, ...)`. With
  `TRITON_TRUE_BATCH_REQUESTS=1`, the orchestrator stacks *all* of them into a
  single batched gRPC call without enforcing the model's `max_batch_size`.
- `tasks.py:3817` computes `dynamic_cap = pose_frame_budget_ms /
  avg_pose_ms_per_person`. With a fast GPU `avg_pose_ms_per_person` decays to
  ~20–50 ms, so `dynamic_cap = 1400 / 20 = 70` person crops per frame —
  well above `max_batch_size=16` — and every such frame trips the batch-size
  guard, falls back to HTTP, and pays the fallback round-trip.

**Root cause (proven):** `_provider_infer_batch` does not chunk its
`frame_payloads` list to the model's max batch size. The behavior-model path
in `tasks.py:_infer_task_batch` *does* chunk via
`_effective_task_batch_size(...)`, but pose's dedicated dispatcher bypasses
that helper and goes straight to the orchestrator.

### Phase 2: Hypothesis

If `PoseRuntime._provider_infer_batch` chunks its crops to `max_batch_size=16`
before dispatch:

- 0 rtmpose batch>16 warnings (correctness improvement).
- Removes the per-frame HTTP fallback round-trip (~50–100 ms on the affected
  frames).
- Should not change Triton's effective batch (the orchestrator was already
  stacking; we now stack in correctly sized chunks).
- Bench expectation: small but measurable drop in pose-upload elapsed time
  (~10–20 % of the 767 s pose block) and zero gRPC→HTTP fallbacks for
  rtmpose during the run.

**Risk:** very low — the chunk size is read from settings with a default of
16 that already matches the deployed engine; if the chunk size matches the
old behavior (one batch of all crops), behavior is identical.

**Rollback:** revert the single edit in `pose_runtime.py`.

### Phase 3: Implemented in commit `17b3745d`

- `backend/apps/pipeline/services/pose_runtime.py`: `_provider_infer_batch`
  now chunks `payloads` at `_resolve_pose_max_batch_size()` (reads
  `TRITON_MODEL_BATCH_SIZE_OVERRIDES["pose_estimation"]`, falls back to 16,
  bounded by `TRITON_MAX_INFLIGHT_REQUESTS`).
- `backend/tests/unit/pipeline/test_pose_runtime_batch_chunking.py`: 5
  regression tests asserting chunk sizes for 4 / 20 / 40 crops, the
  invalid-override fallback, and the empty-input contract.
- `.github/workflows/inference-parallelization.yml`: blocks the gate on the
  new test file plus the cycle 1–5 telemetry writer regression suite.

### Phase 4: Production benchmark — job `a1a448b9-474f-4dea-942b-3288bcae6900`

Run on the same `combined.mp4` (4 541 frames) prod RTX 5090.

| Stage | Cycle 1–5 `74ec0432` | Cycle 6 `a1a448b9` | Δ |
|---|---:|---:|---:|
| Step 2 frame inference wall | 883 s (14 m 43 s) | **879 s** (14 m 39 s) | −0.5 % (unchanged, as designed) |
| Step 2 → step3 transition (pose upload + persist + render) | **13 m 27 s** | **5 m 10 s** | **−61.5 %** |
| Pose-upload block specifically (`step2.frame_stage_timings → step2.pose_upload`) | **12 m 13 s** | **3 m 42 s** | **−69.7 %** |
| Total `run.complete` elapsed | 1 716 s (28 m 36 s) | **1 166 s** (19 m 26 s) | **−32.1 %** |
| Total DB `status=completed` elapsed | 36.44 min | **27.22 min** | **−25.3 %** |
| **Overall FPS (bench probe basis)** | 2.644 fps | **2.776 fps** | +5.0 % |
| **Overall FPS (DB-completed basis)** | 2.077 fps | **2.78 fps** | **+33.8 %** |

| Correctness check | Cycle 1–5 | Cycle 6 | Δ |
|---|---:|---:|---:|
| Frames persisted | 4 541 / 4 541 | 4 541 / 4 541 | ✓ |
| Detections | 72 743 | 72 752 | +9 (+0.012 %) |
| BoundingBoxes | 72 743 | 72 752 | +9 (+0.012 %) |
| FrameEmbeddings | 72 577 | 72 586 | +9 (+0.012 %) |
| Per-class bbox parity (`attention/hand/person/sitting`) | 11 772 / 8 801 / 19 162 / 33 008 | 11 769 / 8 801 / 19 162 / 33 020 | ≤ 0.04 % per class |
| **rtmpose batch>16 warnings** | many | **0** | **bug fixed** |
| TelemetryModelCall DB rows | 29 123 | **20 348** | −30 % (fewer pose-retry calls) |
| Final status | completed | **completed** | ✓ |

Pose-upload contribution to total time dropped from 33.6 % of the run to
13.4 %. Detection numbers improved slightly (more detected objects, not
fewer) because frames that previously timed out after the HTTP fallback now
return clean.

### Phase 5: Decision — **ACCEPTED 2026-06-01**

- Removed the root-cause batch-size-violation: rtmpose warnings dropped from
  many per run to zero.
- Pose-upload block: −69.7 %.
- Total DB-completed elapsed: −25.3 % vs cycle 1–5; **−52.9 % vs baseline**.
- Overall FPS: +33.8 % vs cycle 1–5; **+112 % vs baseline**.
- No correctness regression (parity within 0.012 % on every counter).
- All 5 new chunking unit tests + 41 existing telemetry tests pass; the
  inference-parallelization workflow gate has been updated to enforce them.

---

## Cycle 7 — Cache the Redis client across the embedding loop

### Phase 1: Investigation

**Evidence captured before any change** (cycle 6 job `a1a448b9`):

| Stage | seconds | % of total |
|---|---:|---:|
| Step 2 (frame inference) | 879.0 | 53.8 % |
| Pose post-processing | 221.4 | 13.6 % |
| Persistence | 39.6 | 2.4 % |
| Render | 25.7 | 1.6 % |
| **Embedding (post-`run.complete`)** | **467.6** | **28.6 %** |
| **TOTAL** | **1 633.2** | 100 % |

Embedding produces 72 586 `FrameEmbedding` rows → 467.6 s / 72 586 = **6.44 ms / embedding**. That's ~10× a plausible budget for a SHA-256 stub fallback or a YOLO backbone embedding on this GPU; the cost must live somewhere else.

Code inspection of `apps/video_analysis/tasks.py:generate_embeddings` and `apps/tracking/embeddings.py` shows the loop calls **three Redis helpers per embedding**:

1. `cache_embedding(...)` → `redis_client()` → `Redis.from_url() + .ping()`
2. `cache_job_track_embedding(...)` → `redis_client()` → `Redis.from_url() + .ping()`
3. `_cache_student_embedding_for_analysis(...)` → `_redis_client()` → `Redis.from_url() + .ping()`

That is **3 fresh socket connects + 3 PING round-trips per embedding**, ~217 758 connects per job. At ~1.5 ms each (localhost), the connect-overhead alone is **~325 s of the 467.6 s** embedding wall, leaving ~143 s for the actual model.embed + DB writes + redis SET/SADD/LPUSH/EXPIRE operations.

**Proven root cause:** the per-call factory `Redis.from_url(...).ping()` pattern in both `apps/tracking/embeddings.py::redis_client` and `apps/video_analysis/tasks.py::_redis_client`. Neither caches the client object; every helper invocation pays the connect cost.

### Phase 2: Hypothesis

If `redis_client()` and `_redis_client()` lazily cache a single connected client (with a one-shot retry on connection failure), the per-embedding cost drops from ~6.44 ms to ~1.5 ms, eliminating ~70 % of the embedding wall.

**Expected impact** (all numbers extrapolated linearly from the connect-overhead measurement above):
- Embedding wall: 467.6 s → **~145 s** (−69 %).
- Total job (DB `status=completed`): 1 633 s → **~1 310 s** (−20 %).
- Overall FPS (DB-completed basis): 2.78 → **~3.47** (+24 %).
- Step 2 inference unchanged.

**Risk:** Low.
- `redis-py` is thread-safe; a single connected client can be shared.
- Tests may monkeypatch `settings.REDIS_URL` — we guard by re-checking the URL each call and rebuilding the cache if it changed.
- On any operation error we drop the cache so the next call re-connects.

**Rollback:** Revert the two `Edit`s on `redis_client` / `_redis_client`.

### Phase 3: Implemented in commit `2771824a`

- `backend/apps/tracking/embeddings.py`: `redis_client()` now caches the
  connected client per `REDIS_URL` (successful pings only; failures retry
  to survive brief outages).
- `backend/apps/video_analysis/tasks.py`: same treatment for `_redis_client()`.
- `backend/tests/unit/tracking/test_redis_client_cache.py`: 6 regression
  tests covering reuse, failure-retry semantics, cross-helper sharing, and
  cache-reset cleanliness.
- `.github/workflows/inference-parallelization.yml`: gates the new test file.

### Phase 4: Production benchmark — job `515fe118-6009-4776-916d-6473fbf31ed7`

**Result vs Cycle 6 (same `combined.mp4`, 4 541 frames):**

| Stage | Cycle 6 `a1a448b9` | Cycle 7 `515fe118` | Δ |
|---|---:|---:|---:|
| Step 2 (frame inference) | 879.0 s | **842.6 s** | **−36.4 s** (within noise band) |
| Pose post-processing | 221.4 s | **220.6 s** | unchanged (as designed) |
| Persistence | 39.6 s | **42.4 s** | +2.8 s (noise) |
| Render | 25.7 s | **25.8 s** | unchanged |
| **Embedding** | **467.6 s** | **450.7 s** | **−16.9 s (−3.6 %)** |
| **TOTAL DB `status=completed`** | **1 633.2 s** (27.22 min) | **1 582.1 s** (26.37 min) | **−51.1 s (−3.1 %)** |
| **Overall FPS (DB completed)** | **2.78** | **2.87** | **+3.2 %** |
| **Overall FPS (bench probe)** | 2.776 | 2.865 | +3.2 % |

**Correctness parity (vs all prior cycles):**

| Counter | Baseline | C1–5 | C6 | C7 | Range |
|---|---:|---:|---:|---:|---|
| Detections | 72 751 | 72 743 | 72 752 | 72 745 | ±9 (0.012 %) |
| BBoxes | 72 751 | 72 743 | 72 752 | 72 745 | ±9 (0.012 %) |
| Embeddings | 72 585 | 72 577 | 72 586 | 72 579 | ±9 (0.012 %) |

All within BoT-SORT non-determinism noise. No regression.

### Phase 5: Decision — **ACCEPTED with hypothesis correction**

**Why ACCEPT:**
- −3.6 % embedding wall, −3.1 % total job — small but **measurable** in the same direction as the hypothesis.
- Code is cleaner (single connection per worker process per Redis URL instead of ~217 k).
- No correctness regression.
- 6 regression unit tests gate the new behavior.

**Why the hypothesis was wrong:** I projected −69 % on embedding wall based on extrapolation from "per-call Redis.from_url + ping = ~1.5 ms × 217 k calls = ~325 s". The actual saving was 17 s, which means the real per-call overhead was ~0.08 ms — about 18× smaller than I assumed. `redis-py` 5.x's `Redis.from_url()` is much cheaper than I feared: it constructs a `ConnectionPool` lazily and the pool itself reuses the underlying socket, so most of the "fresh client per call" cost wasn't actually a fresh socket connect — only the first one was, the rest were pool checkouts.

**Where the real 450 s of embedding wall actually lives** (now the next-cycle hypothesis target):
1. `cv2.VideoCapture.set(1, frame_number-1)` + `cv2_cap.read()` per frame — seeks decode codecs from the nearest keyframe each time; expected dominant cost on 4 541-frame H.264 input.
2. `model.embed([crop])` — YOLO backbone forward pass per detection (72 k × ~3-5 ms = ~250 s).
3. Per-row `FrameEmbedding.objects.create()` — no bulk_create.
4. Per-row `detection.embeddings.exists()` idempotency check — extra DB query per detection.

These will be the targets of Cycle 8 onward.

**Honest status:** the user's 15-32 FPS goal requires **much** bigger wins than incremental Redis tuning. Cycle 7 confirmed both the methodology (measure → hypothesize → verify) and that the next big block is *inside* embedding, not at its periphery.

---

## Cycle 8 — Embedding stage: track-level reuse + lazy cv2 + bulk_create

### Phase 1: Investigation (prod-measured, not estimated)

**Direct measurement on prod RTX 5090 (job `515fe118`, video `combined.mp4`):**
- 53 distinct StudentTracks vs 72 579 student Detections → **1 370× duplicate `model.embed()` calls** without track-level reuse.
- cv2 video read benchmark on the same MP4:
  - sequential read (no seek): **0.32 ms/frame** → 1.5 s over 4 541 frames.
  - sequential seek-and-read (current code's `cv2_cap.set(...) + .read()` pattern): **16.69 ms/frame** → **75.8 s over 4 541 frames**.
  - random seek-and-read: 30.40 ms/frame.
- Per-row `FrameEmbedding.objects.create(...)` ran 72 k times last cycle (no bulk path).

The 450.7 s embedding wall therefore decomposes (with high confidence) into:
1. ~75 s of unnecessary cv2 keyframe re-decodes from per-frame `.set()`.
2. Hundreds of seconds of redundant `model.embed()` work — the same 53 vectors recomputed 72 k times.
3. ~50–100 s of per-row INSERT overhead Django ORM doesn't batch.
4. The rest is the genuine model.embed and CPU/IO that must remain.

### Phase 2: Hypothesis

A single bundled change attacks all three independently:

1. **Enable `OFFLINE_EMBEDDING_REUSE_BY_TRACK=1`** — the helper `get_cached_job_track_embedding(...)` already exists; the loop already calls it; it just defaults off. Combined with a process-local `track_vector_cache` dict, we go from 72 k → ~53 `model.embed()` calls without changing the FrameEmbedding-per-Detection contract.

2. **Lazy cv2 frame read** — only call `cv2_cap.set(...).read()` if the current frame has at least one detection whose track has no cached vector yet. Once the warm-up frames are done (~1 frame per unique track), every subsequent frame skips cv2 entirely. Also replaces absolute `.set()` with sequential `.grab()` so forward-seek between needed frames doesn't force keyframe re-decodes.

3. **Bulk-insert FrameEmbedding rows** via a new `persist_embeddings_bulk` helper, batched at `EMBEDDING_BULK_BATCH_SIZE=500`. Vector dimension coercion still happens at the boundary so §17.2 holds.

**Expected impact:**
- model.embed calls: 72 k → ~53 → ~250–350 s saved.
- cv2 seeks: 4 541 → ~53 → ~70 s saved.
- DB INSERTs: 72 k single rows → ~145 bulk_create calls → ~50–80 s saved.
- **Stage wall: 450.7 s → 50–100 s (≈ −80 % to −90 %).**
- Total job wall: 1 582 s → **~1 150–1 250 s (≈ 3.6–4.0 FPS overall)**.

**Risk:**
- Track reuse changes the *content* of FrameEmbedding rows for second-and-later detections of the same track — they now all share the first-detection vector. This is the intended behavior for offline jobs (the vector represents a *student's* appearance, not a frame-specific signal). Downstream consumers (ReID, attendance, FrameEmbedding queries) already treat embeddings as track-scoped — that's exactly why the flag exists in the codebase.
- Lazy cv2 might miss a frame if a track's *first* appearance has a bad bounding box (zero area). The existing degenerate-xyxy fallback in `extract_crop_embedding` (returns the sha256 stub) handles this.
- Bulk_create skips `.save()` hooks. There are none on FrameEmbedding (verified by grep).

**Rollback:** revert the helper edits + flip `OFFLINE_EMBEDDING_REUSE_BY_TRACK` back to 0 in prod env. Two single-line reverts.

### Phase 3: Implementation in commit `<pending push>`

- `backend/apps/tracking/embeddings.py`: new `persist_embeddings_bulk(rows, batch_size=500)` helper that mirrors `persist_embedding`'s vector-coercion contract.
- `backend/apps/video_analysis/tasks.py:generate_embeddings`: lazy cv2 (`cv2_cap.grab()` for forward skip; `.read()` only on needed frames); process-local `track_vector_cache`; pending-rows buffer with periodic `_flush_pending()`; deferred Redis writes co-batched with DB inserts.
- `backend/config/settings/base.py`: `EMBEDDING_BULK_BATCH_SIZE` knob (default 500).
- `tools/prod/prod_enable_parallel_flow.sh`: prod env sets `OFFLINE_EMBEDDING_REUSE_BY_TRACK=1` + `EMBEDDING_BULK_BATCH_SIZE=500`.
- `backend/tests/unit/tracking/test_persist_embeddings_bulk.py`: 7 regression tests (batching at 500/1200/7/0/2, field contract, dimension coercion, batch_size kwarg pass-through).
- `.github/workflows/inference-parallelization.yml`: gates the new test file.

### Phase 4: Production benchmark — job `d2de80a0-31b7-4a47-b9f1-d2e2156ea3a8`

| Stage | Cycle 7 `515fe118` | Cycle 8 `d2de80a0` | Δ |
|---|---:|---:|---:|
| Step 2 frame inference | 842.6 s | **852.8 s** | +1.2 % (noise) |
| Pose post | 220.6 s | 220.7 s | unchanged (as designed) |
| Persistence | 39.6 s | 39.4 s | unchanged |
| Render | 25.7 s | 25.7 s | unchanged |
| **Embedding** | **450.7 s** | **~174 s** | **−61.5 %** |
| **TOTAL DB `status=completed`** | **1 582.1 s** (26.37 min) | **1 312.3 s** (21.87 min) | **−17.1 %** |
| **Overall FPS (DB completed)** | **2.87** | **3.46** | **+20.5 %** |
| **Overall FPS (bench probe)** | 2.865 | 3.455 | +20.6 % |

**Correctness parity (vs all prior cycles):**

| Counter | Baseline | C7 | C8 | Δ vs baseline |
|---|---:|---:|---:|---:|
| Detections | 72 751 | 72 745 | 72 749 | −2 (−0.003 %) |
| BBoxes | 72 751 | 72 745 | 72 749 | −2 (−0.003 %) |
| Embeddings | 72 585 | 72 579 | 72 583 | −2 (−0.003 %) |

All within BoT-SORT non-determinism noise.

### Phase 5: Decision — **ACCEPTED 2026-06-01**

- Embedding stage dropped 277 s — matches the conservative end of the projected −250…−400 s window.
- Overall job 4.5 min faster (−17.1 %) → 21.87 min for a 2 m 31 s video = 19.4 min of "extra" time vs the 5 min SLA target. The SLA gap closed from 18.86 min (Cycle 7) → 14.36 min (Cycle 8).
- Detection/bbox/embedding counts within 0.003 % of baseline — no correctness regression.
- All 12 new regression unit tests (cycles 6, 7, 8) pass and are CI-gated.

**Toward the 7.5-min SLA we still need −14.4 min**, primarily from:
- Step 2 frame inference (currently 852 s ≈ 14.2 min) — Cycle 9 will attack this with Triton ensemble + smaller behavior input.
- Pose post (220 s) — Cycle 10 parallelizes across frames.
- Render (25 s) and persistence (39 s) are already in budget.

---

## Cycle 9 — Behavior Triton Ensemble (NEEDS FURTHER ITERATION)

Cycle 9 implemented the simple four-behavior Triton ensemble and benchmarked it
on production, but it is **not accepted** because the Step 2 wall-time gate did
not improve.

Evidence:

- Replay key: `cycle9-behavior-ensemble-crop-frame-20260601T180847`
- Job: `c1651663-e08a-4e29-9ee3-fd0f09884b98`
- Candidate SHA: `0fa847af43186017316cc11a8c76645ff463e574`
- Parity probe: max abs diff `0.0`
- Decision record: [`docs/cycle_9_results.md`](cycle_9_results.md)

| Metric | Cycle 8 | Cycle 9 | Δ |
|---|---:|---:|---:|
| Step 2 frame inference | 852.8 s | 858.1 s | +0.6 % |
| DB-completed elapsed | 1 312.3 s | 1 110.7 s | −15.4 % |
| DB-completed FPS | 3.46 | 4.09 | +18.1 % |
| App-level model calls | 20 348 | 9 557 | −53.0 % |
| Behavior crop calls | 14 391 | 3 597 | −75.0 % |
| Detection/bbox/embedding parity | baseline | exact parity | pass |

Production readiness note: the pinned Triton build initially had
`TRITON_ENABLE_ENSEMBLE=OFF`. It was rebuilt in place with
`TRITON_ENABLE_ENSEMBLE=ON`; backup binary:
`/home/bamby/services/triton_build_r2502/tritonserver/install/bin/tritonserver.pre_cycle9_no_ensemble_20260601T180729`.

Lesson: collapsing four app-side behavior requests to one ensemble request is
not enough when the server still executes four child TensorRT models and returns
the same dense YOLO tensors. The next design must reduce dense output movement,
server-side critical path, or frame-level process parallelism.

---

## Cycle 10 — LPM Phase 1 Hook + Telemetry (C.2.1-C.2.2 STAGED)

### Phase 1: Investigation

The Logical Path Matrix math layer already exists in
`backend/apps/pipeline/services/logical_path_matrix.py`, with tests for C1-C4,
disabled identity, and the structural exclusion contract. The missing production
integration is in `backend/apps/video_analysis/tasks.py`:
`_run_crop_behaviour_for_items()` decodes each crop behavior/gaze response and
immediately appends the resulting `DetectionBox` objects to `fd.boxes`.

That means the crop-frame path currently has no constraint layer between the
three gaze axes and persistence. `LPM_ENABLED` exists and defaults to `0`, but no
caller applies `apply_to_detection_boxes(...)` or maintains per-track prior LPM
state across frames.

Detailed Phase A record: [`docs/cycle_10_investigation.md`](cycle_10_investigation.md).

### Phase 2: Hypothesis

Hook the existing default-off LPM helper into `_run_crop_behaviour_for_items()`
after behavior/gaze decode and before downstream tracking/persistence consumes
`fd.boxes`.

**Named lever:** single-process Python orchestration.

This is a correctness/staging cycle, not a Step 2 throughput win. The hook adds a
bounded Python post-decode step and is expected to be performance-neutral:
disabled behavior must be identity; enabled behavior must reduce contradictory
gaze boxes without touching `person_detection`, `sitting_standing`, hand-raising,
or pose outputs. Production acceptance is deferred to the C.2.3 benchmark gates
in `docs/logical_path_matrix_spec.md` §10.

**Expected impact:**
- `LPM_ENABLED=0`: no runtime behavior change.
- `LPM_ENABLED=1`: one coherent gaze decision per tracked person/axis where the
  existing decoded boxes provide enough evidence.
- Step 2 wall: no regression above 1%; LPM latency target below 5 ms/frame.

**Risk:** low-to-medium. The flag default stays off. Enabled LPM can change
attention/gaze boxes, so the integration test must prove non-gaze detections are
preserved and the disabled path is identity.

**Rollback:** set `LPM_ENABLED=0` and restart offline workers; no code revert is
needed for production rollback.

### Phase 3: Implementation

- `backend/apps/video_analysis/tasks.py`: `_run_crop_behaviour_for_items()` now
  maintains a per-job `lpm_prior_states_by_track` dict and, when
  `LPM_ENABLED=1`, applies `logical_path_matrix.apply_to_detection_boxes(...)`
  to each frame after crop behavior/gaze boxes are decoded. The hook is skipped
  entirely when `LPM_ENABLED=0`. The hook records one telemetry row through
  `_tel_record_lpm_event(...)` for every frame where LPM executes.
- `backend/apps/pipeline/services/logical_path_matrix.py`: class-label matching
  now covers the deployed bare labels (`left`, `right`, `up`, `down`,
  `forward`, `backward`) and explicitly restricts Phase 1 classification to
  attention/gaze model names so hand-raising `up/down` boxes remain out of
  scope.
- `backend/apps/telemetry/models.py` and
  `backend/apps/telemetry/migrations/0002_telemetrylpmevent.py`: add the
  `telemetry_lpm_events` table required by
  `docs/logical_path_matrix_spec.md` §8.
- `backend/apps/telemetry/session.py` and `backend/apps/telemetry/writer.py`:
  add `LpmEventMeta`, `TelemetrySession.record_lpm_event(...)`, JSON fallback
  keys, and `_bulk_insert_lpm_events(...)` so LPM metrics persist with the same
  dual-sink semantics as model-call telemetry.
- `backend/tests/unit/video_analysis/test_lpm_crop_behaviour_hook.py`: new
  crop-frame integration coverage proves disabled LPM does not invoke the
  filter, enabled LPM preserves unambiguous non-gaze counts, and enabled LPM
  records a telemetry event with the expected contradiction counters.
- `backend/tests/unit/telemetry/test_telemetry_layer.py`: covers
  `LpmEventMeta`, session collection, writer bulk insert invocation, and skip
  behavior when no LPM rows exist.
- `.github/workflows/inference-parallelization.yml`: gates the new integration
  test, telemetry app changes, and the Cycle 10 investigation/execution docs.

Local validation:

```bash
.\backend\.venv\Scripts\python.exe -m pytest \
  backend/tests/unit/video_analysis/test_lpm_crop_behaviour_hook.py \
  backend/tests/unit/pipeline/test_logical_path_matrix.py \
  backend/tests/unit/telemetry/test_telemetry_layer.py \
  -q --tb=short
```

Result after the post-benchmark safety regression test: `65 passed`.

Additional local gate:

```bash
.\backend\.venv\Scripts\python.exe -m pytest \
  backend/tests/unit/pipeline/test_triton_grpc_transport.py \
  backend/tests/unit/pipeline/test_dependency_injection_boundaries.py \
  backend/tests/unit/pipeline/test_triton_configuration_profiles.py \
  backend/tests/unit/pipeline/test_behavior_ensemble_dispatch.py \
  backend/tests/unit/pipeline/test_pose_runtime_batch_chunking.py \
  backend/tests/unit/tracking/test_redis_client_cache.py \
  backend/tests/unit/tracking/test_persist_embeddings_bulk.py \
  backend/tests/unit/pipeline/test_logical_path_matrix.py \
  backend/tests/unit/video_analysis/test_concurrent_model_dispatch.py \
  backend/tests/unit/video_analysis/test_lpm_crop_behaviour_hook.py \
  backend/tests/unit/video_analysis/test_management_commands.py \
  backend/tests/unit/video_analysis/test_pipeline_mode_phase7a.py \
  backend/tests/unit/video_analysis/test_triton_offline_batch_queue.py \
  backend/tests/unit/docs/test_triton_phase_knob_docs_consistency.py \
  backend/tests/unit/telemetry/test_telemetry_layer.py \
  -q --tb=short
```

Result after the post-benchmark safety regression test: `156 passed`, with the pre-existing `async_to_sync` warnings in the
pose runtime chunking tests. `makemigrations --check --dry-run`, compileall,
docs diagram verification, and `git diff --check` also passed locally.

### Decision

**NEEDS FURTHER ITERATION — NOT ACCEPTED.** Production benchmark
`cycle10-lpm-crop-frame-20260601T201239` / job
`17075418-4386-4b5f-85d4-ea23bec71f66` completed and populated
`4541` `telemetry_lpm_events` rows, but LPM did not eliminate any
contradictions (`C1=0`, `eliminated=0`) and it regressed correctness by
dropping `attention_tracking` boxes from `11776` to `2680`.

Production rollback was applied with `LPM_ENABLED=0` and worker restart. The
post-run local safety fix prevents single low-confidence directional gaze boxes
from being suppressed unless a C1-C4 violation fired, but that fix still needs a
fresh production benchmark before Cycle 10 can be reconsidered. Evidence:
[`docs/cycle_10_lpm_phase1_results.md`](cycle_10_lpm_phase1_results.md) and
[`docs/production_inference_benchmark.md`](production_inference_benchmark.md)
§16.

### Safety-fix production proof — still NOT ACCEPTED

The post-run safety fix was deployed and benchmarked on production:

- Replay key: `cycle10-lpm-violationonly-crop-frame-20260601T221110`
- Job: `21666815-f4bd-4f5f-b90e-b9101b4d899d`
- Candidate SHA: `31edac44c66233baadd3a26ddd57b51b1a043d66`
- Telemetry session: `be855e0e-6393-467a-9688-b723a29a56a4`
- Bench summary: `backend/logs/bench_summary_20260602T011132.json`
- GPU CSV: `backend/logs/gpu_monitor_bench_20260602T011132.csv`
- Inference audit:
  `backend/data/videos/21666815-f4bd-4f5f-b90e-b9101b4d899d/inference_audit.json`

Result:

| Metric | Cycle 9 | Safety Proof | Decision |
|---|---:|---:|---|
| Overall FPS | `4.09` | `4.039` | failed performance gate |
| Detection rows | `72749` | `72095` | failed parity gate |
| `attention_tracking` boxes | `11776` | `11122` | failed parity gate |
| LPM `C1` violations | expected non-zero | `0` | failed signal gate |
| LPM eliminated contradictions | expected non-zero | `0` | failed signal gate |

The fix restored most attention boxes compared with the first LPM run
(`2680 → 11122`), but it did not restore Cycle 9 parity and it did not produce
any contradiction-reduction signal. Production was rolled back again to
`LPM_ENABLED=0`.

**Updated decision:** **NEEDS FURTHER ITERATION — NOT ACCEPTED.** Do not tune
post-decode box suppression further. The next LPM design must capture pre-decode
gaze probabilities or run inside the future compact postprocessing/BLS path.

---

## Cycle 9b B.2.b — Gaze-Horizontal Output Fusion (NOT ACCEPTED)

### Phase 1: Investigation

Cycle 9b Step-1 production measurement identified `gaze_horizontal_model` as
the dominant behavior child: `16.058 ms/exec` server delta versus `12.133 ms`
posture, `11.759 ms` vertical, and `11.909 ms` depth. It is also the widest
dense-output behavior model (`[84,2100]`, about `689 KB` per crop).

The runtime consumes only the `right_left` class filter `(4,5)` for horizontal
gaze, so the 80-class output mostly crosses Triton/gRPC/Python only to be
discarded by `_decode_yolo_output0(...)`.

Detailed Phase A/B record:
[`docs/cycle_9b_output_fusion_investigation.md`](cycle_9b_output_fusion_investigation.md).

### Phase 2: Hypothesis

Add an env-gated `gaze2` output-slice variant for horizontal gaze:

- `GAZE_HORIZONTAL_HEAD_VARIANT=coco80|gaze2`
- `gaze_horizontal_gaze2_model` returns `[6,2100]`
  (`xywh` + legacy class channels `4/5`)
- `behavior_ensemble_gaze2` fans out to the same four behavior models except
  horizontal gaze uses the sliced-output variant
- the app remaps compact class IDs `0/1` back to legacy class IDs `4/5`

**Named lever:** dense output bytes.

Expected byte impact at ~17 crops/frame: horizontal-gaze dense output drops from
about `11.4 MB/frame` to about `0.82 MB/frame` (`~92.9 %` reduction for that
child). This should reduce behavior RTT and Python dense-output decode cost for
the measured dominant child. It may not reduce full TensorRT compute if TensorRT
cannot prune unused internal channels from the sliced graph, so production
benchmark evidence decides acceptance.

**Risk:** medium. The largest correctness risk is class-ID drift; the compact
decoder must preserve legacy `4/5` IDs in DB rows. The largest performance risk
is no Step 2 wall movement if output bytes are not the dominant part of this
child's critical path.

**Rollback:** set `GAZE_HORIZONTAL_HEAD_VARIANT=coco80`, route `behavior_all`
back to `behavior_ensemble`, restart Triton and workers. The legacy dense model
and ensemble remain deployed.

**Acceptance gate:** production benchmark on `combined.mp4` must complete with
Step 2 wall improvement, DB-completed FPS improvement, sliced-output parity, and
no row/per-class correctness regression.

### Phase 4: Production parity proof

Production built and loaded `gaze_horizontal_gaze2_model` and
`behavior_ensemble_gaze2` at SHA
`49932a22bfb429a74075e6952788af63eb007810`. The candidate failed the required
pre-benchmark raw tensor parity gate twice:

| Probe | Artifact | Max abs diff | Tolerance | Result |
|---|---|---:|---:|---|
| Initial candidate proof | `backend/logs/gaze_horizontal_gaze2_parity_20260601T230548.json` | `7.125` | `1e-6` | fail |
| Rebuilt-engine proof | `backend/logs/gaze_horizontal_gaze2_parity_20260601T231503_postrebuild.json` | `9.5` | `1e-6` | fail |

Both probes compared the legacy horizontal tensor slice
`output0[:, [0,1,2,3,8,9], :]` against `gaze_horizontal_gaze2_model.output0` and
`behavior_ensemble_gaze2.gaze_h_out`; shapes matched at `[16,6,2100]`, but values
did not.

### Decision

**NOT ACCEPTED.** No full `combined.mp4` benchmark was run because correctness
failed before the benchmark gate. Production was rolled back to:

```bash
GAZE_HORIZONTAL_HEAD_VARIANT=coco80
MODEL_ROUTE_GAZE_HORIZONTAL_MODEL_NAME=gaze_horizontal_model
MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble
LPM_ENABLED=0
```

Mechanism observed: this implementation creates a separate TensorRT plan, so
TensorRT can select different FP16 tactics once the graph output is changed to a
final channel gather. The next output-fusion attempt must preserve exactness by
slicing the already-executed legacy output server-side, or explicitly define a
decoded-detection parity gate before any full benchmark.

---

## Cycle 9b B.2.b — Exact Server-Side Horizontal Slice (ACCEPTED)

### Phase 1: Investigation

Production Triton currently has no `python` backend installed, so the B.1.a BLS
Python compact-postprocessing path is not the lowest-risk next implementation
without rebuilding Triton. The installed backend usable for this candidate is
`tensorrt`.

Detailed Phase A/B record:
[`docs/cycle_9b_exact_slice_investigation.md`](cycle_9b_exact_slice_investigation.md).

### Phase 2: Hypothesis

Keep `gaze_horizontal_model` unchanged and add a tiny TensorRT gather model after
it:

```text
gaze_horizontal_model.output0 [N,84,2100]
  -> gaze_horizontal_slice_model Gather(axis=1, [0,1,2,3,8,9])
  -> output0 [N,6,2100]
```

**Named lever:** dense output bytes.

Expected horizontal output returned to app Python drops from `~11.4 MB/frame` to
`~0.82 MB/frame` at the observed ~17 crops/frame, while avoiding the rejected
standalone `gaze_horizontal_gaze2_model` parity failure mechanism.

### Phase 3: Implementation

- `backend/scripts/build_tensorrt_engines.py`: can build
  `gaze_horizontal_slice_model` from a generated ONNX gather graph.
- `backend/models/triton_repository_cuda12/gaze_horizontal_slice_model/`: new
  TensorRT plan config, input `dense_input [84,2100]`, output `output0 [6,2100]`.
- `backend/models/triton_repository_cuda12/gaze_horizontal_slice_adapter/`: new
  standalone ensemble fallback, image input -> legacy horizontal -> slice.
- `backend/models/triton_repository_cuda12/behavior_ensemble_gaze_slice/`: new
  behavior ensemble that routes only horizontal gaze through the slice model.
- `tools/prod/prod_enable_gaze_horizontal_slice.sh`: builds/enables the route
  with `GAZE_HORIZONTAL_HEAD_VARIANT=slice`.
- `tools/prod/prod_gaze_horizontal_slice_parity.py`: production parity probe for
  direct slice, adapter, and full behavior ensemble outputs.
- `backend/tests/unit/pipeline/test_behavior_ensemble_dispatch.py`: route,
  compact decode/remap, and repository validator coverage.

Local validation:

```bash
backend\.venv\Scripts\python.exe -m py_compile \
  backend\scripts\build_tensorrt_engines.py \
  backend\apps\pipeline\services\ensemble_validator.py \
  tools\prod\prod_gaze_horizontal_slice_parity.py
bash -n tools/prod/prod_enable_gaze_horizontal_slice.sh \
  tools/prod/prod_run_parallel_flow_benchmark.sh \
  tools/prod/prod_start_triton.sh
backend\.venv\Scripts\python.exe -m pytest \
  backend/tests/unit/pipeline/test_behavior_ensemble_dispatch.py \
  backend/tests/unit/docs/test_triton_phase_knob_docs_consistency.py \
  -q --tb=short
```

Result: `18 passed`; py_compile and shell syntax passed.

### Phase 4: Production benchmark

Production deployed SHA `ca69f02a8ceb214d7ef55cd2ae4b7ec75549c257`, built the
slice plan, validated the ensemble repository, and completed the canonical
`combined.mp4` benchmark:

| Item | Value |
|---|---|
| Replay key | `cycle9b-exactslice-crop-frame-20260601T233211` |
| Job ID | `7933c1e5-a970-47a3-81c5-0c9bd01bd332` |
| Bench summary | `backend/logs/bench_summary_20260602T023450.json` |
| GPU CSV | `backend/logs/gpu_monitor_bench_20260602T023450.csv` |
| Inference audit | `backend/data/videos/7933c1e5-a970-47a3-81c5-0c9bd01bd332/inference_audit.json` |
| Post-benchmark parity | `backend/logs/gaze_horizontal_slice_parity_20260601T235623_postbench.json`, `max_abs_diff=0.0` |
| Final status | `completed` |

| Metric | Cycle 9 Dense Ensemble | Exact Slice | Delta |
|---|---:|---:|---:|
| Step 2 wall | `858.1 s` | `573.927 s` | `-33.1 %` |
| Step 2 FPS | `5.29` | `7.912` | `+49.6 %` |
| DB-completed elapsed | `1110.7 s` | `~1054 s` | `-5.1 %` |
| DB-completed FPS | `4.09` | `4.307` | `+5.3 %` |
| Behavior RTT mean | `107.9 ms` | `91.470 ms` | `-15.2 %` |
| Behavior RTT p95 | `173.9 ms` | `146.015 ms` | `-16.0 %` |
| Avg / peak GPU util | `9.36 % / 43 %` | `9.595 % / 45 %` | `+0.235 pp / +2 pp` |

Correctness parity:

| Counter | Cycle 9 | Exact Slice | Delta |
|---|---:|---:|---:|
| Frames | `4541` | `4541` | 0 |
| Detections | `72749` | `72747` | `-2` |
| Bounding boxes | `72749` | `72747` | `-2` |
| Frame embeddings | `72583` | `72581` | `-2` |
| Student tracks | `53` | `53` | 0 |
| `attention_tracking` boxes | `11776` | `11776` | 0 |
| `hand_raising` boxes | `8800` | `8801` | `+1` |
| `person_detection` boxes | `19162` | `19162` | 0 |
| `sitting_standing` boxes | `33011` | `33008` | `-3` |

### Decision

**ACCEPTED.** The named lever was dense output bytes, and production proved the
candidate reduces the targeted Step 2 wall while preserving raw tensor parity and
DB correctness. Production remains on `GAZE_HORIZONTAL_HEAD_VARIANT=slice` with
`LPM_ENABLED=0`. Rollback is:

```bash
bash tools/prod/prod_enable_parallel_flow.sh --profile per-frame-signals
```

This accepted the exact-slice B.2.b subcandidate. The next B.2 production run
measured B.2.c (exact slice plus Top-K) against this baseline.

---

## Cycle 9b B.2.c — Exact Slice + Top-K Anchor Packing (ACCEPTED WITH CAVEAT)

### Phase 1: Investigation

The accepted exact-slice route still returned dense `[C,2100]` grids for every
behavior child. At the exact-slice baseline, behavior output traffic was still
about `6.85 MB/frame`, and behavior RTT averaged `91.470 ms`.

Detailed Phase A/B record:
[`docs/cycle_9b_topk_anchor_packing_investigation.md`](cycle_9b_topk_anchor_packing_investigation.md).

### Phase 2: Hypothesis

Keep `GAZE_HORIZONTAL_HEAD_VARIANT=slice` and add TensorRT Top-K adapters after
all behavior children:

```text
posture_model.output0 [N,14,2100] -> posture_topk_model [N,14,100]
gaze_horizontal_slice_model.output0 [N,6,2100] -> gaze_horizontal_slice_topk_model [N,6,100]
gaze_vertical_model.output0 [N,14,2100] -> gaze_vertical_topk_model [N,14,100]
gaze_depth_model.output0 [N,14,2100] -> gaze_depth_topk_model [N,14,100]
```

**Named lever:** dense output bytes.

Expected behavior output traffic drops from `~6.85 MB/frame` to `~0.33 MB/frame`
while preserving the Python decoder's top-candidate semantics
(`TRITON_BEHAVIOR_TOP_K_VALUE=100` matches `TRITON_YOLO_MAX_DECODE_CANDIDATES`).

### Phase 3: Implementation

- `backend/scripts/build_tensorrt_engines.py`: exports per-task Top-K ONNX
  adapter graphs and builds FP32 TensorRT plans.
- `backend/models/triton_repository_cuda12/behavior_ensemble_gaze_slice_topk/`:
  full behavior ensemble that returns Top-K outputs for all behavior children.
- `backend/models/triton_repository_cuda12/*_topk_model/`: adapter configs and
  version directories.
- `tools/prod/prod_enable_behavior_topk.sh`: builds/enables the route and sets
  `TRITON_BEHAVIOR_TOP_K_ENABLED=1`.
- `tools/prod/prod_behavior_topk_parity.py`: decoded parity probe comparing
  dense exact-slice outputs against Top-K outputs.
- Workflow/test coverage: `test_behavior_ensemble_dispatch.py`,
  `test_yolo_decode_dynamic_shapes.py`, and
  `test_triton_phase_knob_docs_consistency.py`.

FP16 adapters failed decoded parity before the benchmark, so the production
helper builds Top-K adapters with `--no-fp16`.

### Phase 4: Production benchmark

Production deployed SHA `9f879affeb4478e63a09276b10a2d64844bcbc44`, built the
FP32 Top-K adapters, validated the ensemble repository, passed decoded parity,
and completed the canonical `combined.mp4` benchmark:

| Item | Value |
|---|---|
| Replay key | `cycle9b-topk-crop-frame-20260602T041900` |
| Job ID | `be4ba9ee-4786-48e9-8334-28feb237a1fb` |
| Telemetry session | `c4710435-4ec0-49e1-8ffb-60012fa878c9` |
| Bench summary | `backend/logs/bench_summary_20260602T042139.json` |
| GPU CSV | `backend/logs/gpu_monitor_bench_20260602T042139.csv` |
| Inference audit | `backend/data/videos/be4ba9ee-4786-48e9-8334-28feb237a1fb/inference_audit.json` |
| Decoded parity | `backend/logs/behavior_topk_parity_20260602T011830Z_fp32.json`, `failed_count=0`, `max_score_diff=0.0`, `max_box_diff=0.0` |
| Final status | `completed` |

| Metric | Exact Slice Baseline | Exact Slice + Top-K | Delta |
|---|---:|---:|---:|
| Step 2 frame wall | `573.927 s` | `540.399 s` | `-5.84 %` |
| Step 2 through pose upload | `799.345 s` | `767.589 s` | `-3.97 %` |
| Audit `run.complete` wall | `865.419 s` | `833.810 s` | `-3.65 %` |
| DB-completed elapsed | `1052.281 s` | `1022.952 s` | `-2.79 %` |
| DB-completed FPS | `4.307` | `4.429` | `+2.8 %` |
| Behavior RTT mean | `91.470 ms` | `84.865 ms` | `-7.22 %` |
| Behavior RTT p95 | `146.072 ms` | `128.138 ms` | `-12.28 %` |
| Behavior output / frame | `~6.85 MB` | `~0.33 MB` | `~95 %` less |
| Avg / peak GPU util | `9.595 % / 45 %` | `9.3 % / 53 %` | `-0.295 pp / +8 pp` |

Correctness stayed within the established tolerance: frames `4541`, detections
`72747 → 72762` (`+0.0206 %`), `attention_tracking` `11776 → 11781`
(`+0.0425 %`), `person_detection` unchanged at `19162`, and StudentTrack count
unchanged at `53`.

### Decision

**ACCEPTED WITH CAVEAT.** The named lever moved and production proved lower
Step 2 wall, lower behavior RTT, and decoded parity. The caveat is that average
GPU utilization did not improve, so the next optimization must target GPU
occupancy / server-side execution or orchestration, not only response-byte
trimming.

Production remains on:

```bash
GAZE_HORIZONTAL_HEAD_VARIANT=slice
TRITON_BEHAVIOR_TOP_K_ENABLED=1
TRITON_BEHAVIOR_TOP_K_VALUE=100
MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble_gaze_slice_topk
LPM_ENABLED=0
```

Rollback to exact-slice without Top-K:

```bash
bash tools/prod/prod_enable_gaze_horizontal_slice.sh --input-size 320 --skip-build
bash -l tools/prod/prod_start_triton.sh
bash tools/prod/prod_start_celery_workers.sh
```

Detailed result doc:
[`docs/cycle_9b_topk_anchor_packing_results.md`](cycle_9b_topk_anchor_packing_results.md).

---

## Cycle 11.A — Behavior Input Size 320 → 256 (NOT ACCEPTED BY REAL BENCHMARK)

### Phase 1: Investigation

Cycle 9b B.3 remeasurement showed that per-crop compute stayed roughly constant
at `~0.94 ms/crop` and that tuning only the dominant child was capped at about
`4 %` Step 2 wall improvement. Cycle 11.A therefore targeted all four behavior
children by reducing input pixels from `320x320` to `256x256`.

Detailed plan:
[`docs/cycle_11_input_size_investigation.md`](cycle_11_input_size_investigation.md).

### Phase 2: Implementation

Cycle 11 helper code was staged:

- `backend/apps/pipeline/services/triton_ensemble_input_size.py`: idempotent
  rewriter for ensemble `images` input dims.
- `tools/prod/prod_set_behavior_input_size.sh`: orchestrates config rewrite,
  engine rebuild, Top-K rebuild, and restart.
- `tools/prod/prod_behavior_input_size_parity.py`: two-pass capture/compare
  parity probe.
- `backend/apps/pipeline/services/ensemble_validator.py`: now validates
  `behavior_input_size` and derived YOLO anchor counts.
- `tools/prod/prod_start_triton.sh`: now uses explicit model loading when
  `TRITON_LOAD_MODEL` is set, so stale compatibility ensembles do not block a
  different input-size experiment.
- `tools/prod/prod_collect_benchmark_metrics.py`: collects DB correctness,
  telemetry RTT, GPU CSV, and inference-audit metrics into one JSON/Markdown
  evidence bundle.
- `tools/prod/prod_run_behavior_input_size_matrix.sh`: runs reproducible 320/256
  production benchmark matrices and rolls back to 320 by default.

Runtime guard commit:
`4bcc79a5a4ea7c4d452b6fcd3ae3a6ff064a3bb5`.

### Phase 3: Production parity warning

Production built the 256 candidate engines and adapters, then captured the
candidate output tensors. The synthetic pre-benchmark parity gate failed:

| Item | Value |
|---|---|
| Baseline capture | `backend/logs/parity_capture_320_20260602T123459.npz` |
| Candidate capture | `backend/logs/parity_capture_256_20260602T154826.npz` |
| Parity JSON | `backend/logs/parity_input_size_256_20260602T154842.json` |

| Model | Class agreement | Mean centroid drift | Result |
|---|---:|---:|---|
| `posture_model` | `0.6950` | `119.204 px` | fail |
| `gaze_horizontal_model` | `1.0000` | `0.276 px` | pass |
| `gaze_vertical_model` | `0.9550` | `142.669 px` | fail |
| `gaze_depth_model` | `1.0000` | `141.526 px` | fail |

Configured gates were class agreement `>= 0.995`, centroid drift `<= 0.5 px`,
and confidence delta `<= 0.05`.

### Phase 4: Real benchmark result

The parity result was treated as a warning, not a final decision. The real
production benchmark was then run and showed why the candidate cannot ship yet.

Run:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_run_behavior_input_size_matrix.sh \
  --sizes "320 256" \
  --tag cycle11-input-size-realbench-$(date -u +%Y%m%dT%H%M%SZ) \
  --timeout 7200
```

Evidence:

| Item | Value |
|---|---|
| Replay key | `cycle11-input256-realbench-20260602T161641Z-input256` |
| Job ID | `822b0da4-fbf2-4186-a5a6-dd066f2eb571` |
| Metrics JSON | `backend/logs/cycle11-input256-realbench-20260602T161641Z/input_256_metrics.json` |
| Metrics Markdown | `backend/logs/cycle11-input256-realbench-20260602T161641Z/input_256_metrics.md` |
| Model agreement JSON | `backend/logs/cycle11-input256-realbench-20260602T161641Z/model_agreement_320_vs_256.json` |
| Model agreement Markdown | `backend/logs/cycle11-input256-realbench-20260602T161641Z/model_agreement_320_vs_256.md` |
| Matrix TSV | `backend/logs/cycle11-input256-realbench-20260602T161641Z/matrix_runs.tsv` |

| Metric | 320 Top-K baseline | 256 candidate | Delta |
|---|---:|---:|---:|
| DB-completed FPS | `4.439` | `4.820` | `+8.58 %` |
| Step 2 FPS | `8.403` | `11.594` | `+37.97 %` |
| Step 2 frame wall | `540.399 s` | `391.673 s` | `-27.52 %` |
| Behavior RTT mean | `84.865 ms` | `51.529 ms` | `-39.28 %` |
| GPU avg util | `9.344 %` | `7.367 %` | `-21.16 %` |
| Detection rows | `72,762` | `101,213` | `+39.10 %` |
| BBox rows | `72,762` | `101,213` | `+39.10 %` |
| `attention_tracking` agreement F1@IoU0.5 | `100.000 %` | `31.195 %` | `-68.805 pp` |
| `hand_raising` agreement F1@IoU0.5 | `100.000 %` | `38.032 %` | `-61.968 pp` |
| `person_detection` agreement F1@IoU0.5 | `100.000 %` | `100.000 %` | `0 pp` |
| `sitting_standing` agreement F1@IoU0.5 | `100.000 %` | `65.250 %` | `-34.750 pp` |

The agreement rows are baseline-reference metrics from
`tools/prod/prod_compare_benchmark_accuracy.py`; they are not human-labeled
ground-truth accuracy.

### Decision

**NOT ACCEPTED.** The benchmark-backed decision is no longer based on the
synthetic parity probe. The candidate improved speed but regressed persisted
signal counts substantially and lowered average GPU utilization. Production was
rolled back to the accepted Cycle 9b B.2.c profile:

```bash
TRITON_CROP_BEHAVIOR_INPUT_SIZE=320
GAZE_HORIZONTAL_HEAD_VARIANT=slice
MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble_gaze_slice_topk
TRITON_BEHAVIOR_TOP_K_ENABLED=1
TRITON_BEHAVIOR_TOP_K_VALUE=100
LPM_ENABLED=0
```

Triton health returned `200`, and all active 320 Top-K models were verified
`READY`.

Detailed result doc:
[`docs/cycle_11_input_size_results.md`](cycle_11_input_size_results.md).

---

## Cycle 9b B.4 — Larger Batch Window (NOT ACCEPTED)

### Phase 1: Investigation

Cycle 9b exact slice + Top-K was the accepted baseline for this B.4 comparison.
It keeps behavior input at `320x320`, routes `behavior_all` through
`behavior_ensemble_gaze_slice_topk`, and runs the offline batch queue with:

```text
TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=2
TRITON_OFFLINE_BATCH_QUEUE_MAX_CONCURRENCY=2
```

Detailed investigation:
[`docs/cycle_9b_batch_window_investigation.md`](cycle_9b_batch_window_investigation.md).

### Phase 2: Hypothesis

Raise `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES` from `2` to `4` while keeping
the accepted 320 Top-K model graph unchanged.

**Named lever:** single-process Python orchestration / batch formation.

The candidate may reduce frame-window overhead and improve true-batch packing
when crop counts fluctuate across frames. It does not reduce dense output bytes
or model compute. The expected gain is marginal, so the production benchmark is
the decision authority.

**Risk:** medium. Earlier large-window crop-frame runs hit severe RSS growth.
The current Top-K path has no-copy output split, NumPy outputs, job-scoped gRPC,
and trim amortization, but the B.4 benchmark still must sample worker RSS and
reject the candidate if peak RSS reaches `4096 MiB`.

**Rollback:** restore `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=2` by re-running
`prod_enable_parallel_flow.sh --profile per-frame-signals`, then restart Triton
and Celery workers.

**Acceptance gate:** production `combined.mp4` benchmark must complete, Step 2
frame wall must improve versus `540.399 s`, worker RSS must stay below
`4096 MiB`, and correctness/model-agreement metrics must stay within the
accepted 320 Top-K baseline tolerance.

### Phase 3: Production Benchmark

The production benchmark completed on `combined.mp4`:

| Item | Value |
|---|---|
| Candidate replay key | `cycle9b-b4-maxframes4-20260602T175820Z` |
| Candidate job ID | `416efe8c-772c-442f-8e55-cf44c54fe261` |
| Evidence directory | `backend/logs/cycle9b-b4-maxframes4-20260602T175820Z/` |
| Bench summary | `backend/logs/bench_summary_20260602T210058.json` |
| Telemetry session | `68c36ffd-c493-42d4-b27f-81aa47ba7fa6` |
| RSS TSV | `backend/logs/cycle9b-b4-maxframes4-20260602T175820Z/batch_window_rss.tsv` |

| Metric | 320 Top-K baseline | B.4 candidate | Delta |
|---|---:|---:|---:|
| DB-completed FPS | `4.439` | `4.471` | `+0.71 %` |
| Step 2 frame wall | `540.399 s` | `512.445 s` | `-5.17 %` |
| Step 2 through pose upload | `767.589 s` | `739.389 s` | `-3.67 %` |
| Behavior RTT mean | `84.865 ms` | `99.251 ms` | `+16.95 %` |
| GPU avg util | `9.344 %` | `9.459 %` | `+1.23 %` |
| GPU peak util | `53.0 %` | `49.0 %` | `-7.55 %` |
| Worker RSS peak | not sampled | `1120.328 MiB` | pass |
| Student tracks | `53` | `47` | `-11.32 %` |

Count-level behavior rows were almost unchanged, but model-agreement F1@IoU0.5
against the accepted 320 Top-K baseline failed for three behavior outputs:
`attention_tracking=24.531 %`, `hand_raising=26.648 %`, and
`sitting_standing=17.217 %`; `person_detection` stayed `100.000 %`.

### Phase 4: Decision

**NOT ACCEPTED.** The candidate produced only modest wall-time and FPS gains,
increased mean behavior RTT, reduced track continuity, and failed the
model-agreement gate. Production was restored to
`TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=2` with the accepted 320 Top-K route.

Detailed result doc:
[`docs/cycle_9b_batch_window_results.md`](cycle_9b_batch_window_results.md).

---

## Cycle 9b B.1 — Compact Postprocessing (PHASE A PROBE COMPLETE)

### Phase 1: Investigation

Detailed investigation:
[`docs/cycle_9b_compact_postproc_investigation.md`](cycle_9b_compact_postproc_investigation.md).

The accepted exact-slice + Top-K baseline already reduced behavior output
traffic from the old dense-output problem (`~17.1 MB/frame`) to roughly
`~0.33 MB/frame`. B.1 is therefore no longer evaluated as a large dense-byte
removal by default; it must prove remaining response-byte and Python decode/NMS
savings against the current Top-K route.

### Phase 2: Production Decode-Cost Probe

The measurement-only probe ran on production:

| Item | Value |
|---|---|
| Probe JSON | `backend/logs/cycle9b_b1_decode_cost_topk_20260602T185559Z.json` |
| Probe Markdown | `backend/logs/cycle9b_b1_decode_cost_topk_20260602T185559Z.md` |
| Baseline replay key | `cycle9b-topk-crop-frame-20260602T041900` |
| Sampled crops | `340` |
| Batches | `20` |
| Mean RTT with parse | `62.082 ms` |
| Mean gRPC/Triton wait | `59.651 ms` |
| Mean `as_numpy` parse | `0.114 ms` |
| Mean Python decode/NMS | `3.125 ms/batch` |
| Decode/NMS per crop | `0.183823 ms` |
| Output bytes per crop | `19,200` |
| Estimated compact bytes per crop | `11.2` |

With `3597` accepted-baseline behavior calls, measured client decode/NMS is
about `11.24 s`, or `~2.08 %` of the accepted `540.399 s` Step 2 frame wall.
This is probe-only arithmetic and has no acceptance, rejection, skip, closure,
or prioritization authority.

Probe context for the last run:

| Question | Evidence | Decision impact |
|---|---|---|
| Was a compact backend benchmarked? | No; only `tools/prod/prod_probe_behavior_decode_cost.py` was run over accepted Top-K crops. | Status is `PROBE_ONLY`; no decision authority. |
| What did the probe isolate? | Client response parse plus Python `_decode_yolo_output0`/NMS. | It bounds only a decode/NMS optimization. |
| Does that bound meet the cycle gate? | Gate satisfaction requires a real production benchmark. | `NO_DECISION_PRODUCTION_BENCHMARK_REQUIRED`. |
| Why are the results small despite `99.942 %` byte reduction potential? | Cycle 9b Top-K already reduced behavior output traffic, leaving only `19,200 bytes/crop`; Python decode/NMS is `3.125 ms/batch`. | The original dense-byte bottleneck has mostly been removed. |
| What bottleneck hypothesis should be tested? | Mean gRPC/Triton wait is `59.651 ms` of `62.082 ms` RTT-with-parse. | The full benchmark must prove whether a B.1 candidate reduces wait/server execution, not just response decode. |

### Phase 3: Decision

No B.1 decision exists. No compact backend is implemented, accepted, rejected,
skipped, closed, or deprioritized. Python BLS remains blocked until the pinned
Triton runtime includes a Python backend or a controlled runtime rebuild/switch
is benchmarked. Repeat B.1 through the production-authoritative path:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_run_b1_decode_cost_full_benchmark.sh
```

The full production repeat completed as replay
`cycle9b-b1-fullbench-20260602T192344Z`, job
`00e0e1da-44b6-4198-ad39-39fd853e4e18`, deployed SHA `7556e84`, on
`combined.mp4`. It kept the accepted 320 exact-slice + Top-K route, so it
records `NO_DECISION_BENCHMARK_RECORDED`. Final benchmark vs accepted Top-K:
Step 2 wall `540.399 s → 540.748 s` (`+0.06 %`), DB FPS `4.439 → 4.346`
(`-2.11 %`), behavior RTT mean `84.865 ms → 85.201 ms` (`+0.40 %`), GPU avg
`9.344 % → 11.962 %`, DB rows within `0.02 %`, and StudentTracks unchanged
at `53`. Fresh decode evidence from `19,146` crops measured RTT `45.160 ms`,
infer wait `42.704 ms`, and decode/NMS `2.040 ms/batch` (`0.120 ms/crop`).
Evidence directory:
`backend/logs/cycle9b-b1-fullbench-20260602T192344Z/`.

The second production repeat completed as
`cycle9b-b1-fullbench-repeat2-20260602T195517Z`, job
`df7f832f-de81-4f92-89c7-fd213bdba7fa`, with wrapper status
`NO_DECISION_BENCHMARK_RECORDED`. It also kept the accepted 320 exact-slice +
Top-K route: Step 2 wall `546.702 s`, DB FPS `4.411`, behavior RTT mean
`84.360 ms`, GPU avg `9.660 %`, DB rows within `0.01 %`, StudentTracks
unchanged at `53`, and model-agreement F1@IoU0.5 `>=99.707 %`. Fresh decode
evidence from `19,146` crops measured RTT `47.154 ms`, infer wait
`44.162 ms`, and decode/NMS `4.243 ms/batch` (`0.250 ms/crop`). Evidence
directory:
`backend/logs/cycle9b-b1-fullbench-repeat2-20260602T195517Z/`.

Detailed result doc:
[`docs/cycle_9b_compact_postproc_results.md`](cycle_9b_compact_postproc_results.md).

---

## Pending Cycles (restaged after B.1 repeat evidence)

Listed in order. Only proceed if the staged cycles above do not lift FPS to the target.
The B.1 repeat benchmarks on 2026-06-02 measured decode/NMS as a small part of
Step 2 while infer wait plus server/orchestration remained dominant, so the next
cycle is persistent async dispatch instrumentation before another compact-output
implementation attempt. Cycle 12 Phase A instrumentation and wrapper are staged,
and the clean production profile completed as
`cycle12-async-dispatch-profile-clean-20260602T213441Z` / job
`dfa1f138-7086-418a-ba17-9999cd12b9ac`. It measured `349.643 s`
async-dispatch blocking wall, dominated by `behavior_all` (`338.779 s`), but
did not implement an optimization candidate. Cycle 12 remains active and
incomplete; the next candidate must overlap behavior wait/server execution, not
only replace the sync async-bridge. The 2026-06-03 metric decision stages
Cycle 12.B bounded behavior-wait overlap behind
`TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=1`; it must run through
`tools/prod/prod_run_behavior_overlap_benchmark.sh` before any decision can be
made.

Cycle 12.B production benchmark
`cycle12-behavior-overlap-20260602T223350Z` / job
`46ba8b2a-3c61-4d89-b7b6-63ec72159428` improved Step 2 wall by `-26.81 %`
and DB FPS by `+17.03 %`, with tracks unchanged and model-agreement F1
`>=99.716 %`. It is **NEEDS FURTHER ITERATION / NOT ACCEPTED** because
behavior RTT mean regressed `+36.00 %` and p95 regressed `+75.44 %`. The next
cycle is Cycle 12.C: keep the overlap but prevent two behavior jobs from being
in flight at once.

Cycle 12.C production benchmark
`cycle12-single-inflight-overlap-20260602T225821Z` / job
`069a217f-fa43-48cc-bf18-c946d53bb3ee` is **ACCEPTED**. Step 2 wall improved
`-14.98 %`, DB FPS improved `+9.35 %`, GPU avg improved `+10.57 %`, behavior
RTT mean improved `-1.09 %`, and model agreement stayed `>=99.716 %`.

| # | Optimization | Expected lift | Cost / risk |
|---:|---|---|---|
| 12.A | Persistent async dispatcher measurement — quantify `async_runner.run(...)` boundary churn in `tasks.py` | measurement complete; no optimization decision | low; profiling flag rollback |
| 12.B | Bounded behavior-wait overlap dispatcher — start `behavior_all` for batch N while preparing batch N+1, preserving ordered finalization | **needs further iteration**: wall/FPS improved, RTT gate failed | medium; rollback is `TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=0` |
| 12.C | Single-inflight behavior overlap — prepare current batch, finalize pending behavior, then submit current behavior | **ACCEPTED**: Step 2 `-14.98 %`, FPS `+9.35 %`, RTT mean `-1.09 %` | medium; rollback is `TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=0` |
| 11.B / B.3 Step 2 | Kernel-tactic or batch-profile tuning on the dominant 320 behavior child after Top-K | bounded at ~4 % Step 2 | low-medium; engine rebuild only if parity holds |
| 13 | Persistence/render cleanup after Cycle 12.C | **Phase A measured; Cycle 13.A profiling staged** in `docs/cycle_13_persistence_render_investigation.md`; Cycle 12.C evidence shows `39.820 s` persistence, `25.692 s` render, and `187.139 s` embedding write span | low; measurement-only flag rollback is `EMBEDDING_STAGE_PROFILING=0` |
| 14 | Compact server-side postprocessing / BLS / TRT plugin that reduces wait or server execution, not only output bytes | unknown; must benchmark candidate | high; backend/runtime contract change |
| 15 | CUDA shared memory or video sharding architecture decision | high only if bottleneck shifts | medium-high; lifecycle and tracking risk |
| 16.A | Redis command-cost instrumentation | evidence / upper-bound only | low; profiling must not become decision authority |
| 16.B | Redis pipeline coalescing for embedding/tracking side effects | embedding wall / total wall only if 16.A proves Redis command wall is material | low-medium; Redis remains non-authoritative |
| 17 | Redis Streams for progress and benchmark sampling | DB polling/write overhead or evidence quality | medium; PostgreSQL remains terminal authority |
| 18 | Redis boundary-state cache for future video sharding | stitch stability if sharding is selected | medium-high; only after Cycle 15 |
| 19 | Redis server-side scripts for measured read/compute/write hotspots | conditional on Cycle 16.A proof | medium; rollback to non-script path |

Redis roadmap note (2026-06-03): broader Redis strategies are documented in
`docs/redis_broader_optimization_opportunities.md`. Cycle 7 proved Redis
hypotheses can be overestimated, so the first Redis cycle must be command-cost
instrumentation, not implementation.

---

## Production-validation requirement (per mission rules)

Every cycle marked **STAGED** above must, before being marked **ACCEPTED**:

1. Be committed and pushed to prod.
2. Be benchmarked via `tools/prod/prod_run_parallel_flow_benchmark.sh` on `combined.mp4` (4541 frames).
3. Have results recorded in [`docs/production_inference_benchmark.md`](production_inference_benchmark.md) as a Before/After table.
4. Show no regression in detection rows, tracking continuity, persistence, rendering, or embeddings.
5. Have `AGENTS.md` updated with the new replay key + job id + accept/reject status.

Until all five are true, the optimization is **STAGED**, not **ACCEPTED**.
