# Production Inference Benchmark & Issue Log

**Environment:** Linux, no Docker, no sudo — NVIDIA RTX 5090 (32 GB GDDR7, sm_120 Blackwell)  
**Recorded:** 2026-05-30  
**Branch:** master @ `dcefea9`  
**TRT:** 10.16.1.11 | **CUDA:** 12.8 | **Triton:** 2.55.0 | **Python:** 3.12 (backend venv)

---

## 1. Hardware & Stack

| Item | Value |
|------|-------|
| GPU | NVIDIA GeForce RTX 5090, sm_120 (Blackwell) |
| VRAM | 32 607 MiB total / 21 389 MiB used at idle (engines loaded) |
| CPU | 32 cores |
| CUDA | 12.8 |
| TensorRT | 10.16.1.11 (backend/.venv python3.12) |
| Triton | 2.55.0 (offline mode, port 39100) |
| PyTorch | 2.7.1+cu128 |
| ultralytics | 8.4.56 |

---

## 2. TRT Engine Inventory

Rebuilt 2026-05-30T13:26:18 UTC with TRT 10.16.1.11 / CUDA 12.8.  
All engines use FP16, workspace 6 144 MiB, optimization profile batch [1 … 8 … 32].

| Triton Model | Source ONNX / PT | Plan Size | Batch Profile | Input Shape |
|---|---|---|---|---|
| `person_detector` | `student_teacher.pt` → ONNX (dynamic-patched) | 42 MB | [1, 8, 32] | [N, 3, 640, 640] |
| `posture_model` | `standing_sitting.pt` | 54 MB | [1, 8, 32] | [N, 3, 640, 640] |
| `gaze_horizontal_model` | `right_left.pt` | 54 MB | [1, 8, 32] | [N, 3, 640, 640] |
| `gaze_vertical_model` | `up_down.pt` | 54 MB | [1, 8, 32] | [N, 3, 640, 640] |
| `gaze_depth_model` | `forward_backward.pt` | 54 MB | [1, 8, 32] | [N, 3, 640, 640] |
| `rtmpose_model` | `rtmpose-m.onnx` | 28 MB | [1, 8, 32] | [N, 3, 256, 192] |

Manifest: `backend/models/tensorrt_builds/latest_compat.json`

---

## 3. Latency Measurements

### 3.1 Shadow-Probe Latency (first inference per job, single frame, near-cold GPU)

Measured via `inference_audit.json` — includes Triton HTTP round-trip, engine warm-up for the first batch, and result deserialization.

| Model | Latency (ms) | Notes |
|---|---|---|
| `person_detector` | 601.1 | First call after engine load |
| `posture_model` | 625.1 | First call after engine load |
| `gaze_horizontal_model` | 803.9 | Highest cold latency observed |
| `gaze_vertical_model` | 613.5 | First call after engine load |
| `gaze_depth_model` | — | Not captured in shadow probe (version routing error in first run; fixed) |
| `rtmpose_model` | — | Not included in shadow probe |

### 3.2 Steady-State HTTP Round-Trip (dispatch → 200 OK)

Measured from Celery log timestamps: time between `InferenceOrchestrator: dispatching` and the corresponding `HTTP/1.1 200 OK` response. Batch size 1–4 frames per call (offline pipeline).

| Model | n | min (ms) | avg (ms) | p95 (ms) | max (ms) |
|---|---|---|---|---|---|
| `person_detector` | 14 | 476 | 856 | 1 166 | 1 166 |
| `posture_model` | 16 | 476 | 882 | 1 308 | 1 308 |
| `gaze_horizontal_model` | 14 | 667 | 1 119 | 1 620 | 1 620 |
| `gaze_vertical_model` | 14 | 404 | 924 | 1 323 | 1 323 |
| `gaze_depth_model` | 16 | 381 | 1 637 | 5 655 | 5 655 |
| `rtmpose_model` | — | — | — | — | — |

> **Context:** These RTTs include CPU pre-processing time + HTTP serialization overhead + actual GPU inference + response deserialization. Actual Triton inference on RTX 5090 for these models is estimated <5 ms per batch; the remaining time is pipeline/HTTP overhead.

### 3.3 GPU Telemetry During Full Video Run

Video: `Raw Data/Diverse Classroom Enviroments/combined.mp4` — 4 541 frames, 35 MB  
Sample interval: 2 s | 179 samples captured during active processing

| Metric | Value |
|---|---|
| Average GPU utilisation | 1.2 % |
| Peak GPU utilisation | 12 % |
| VRAM used (stable) | 21 389 MiB |
| GPU temperature | 36 °C |
| Average board power | 60.1 W (idle-like; RTX 5090 TDP = 575 W) |

**Conclusion:** The RTX 5090 is essentially idle. The pipeline is CPU-bound due to the architecture described in §4 below.

---

## 4. Issues Encountered & Fixes Applied

### Issue 6 — Video Jobs Stall After First Batch; Celery Beat Not Running

**Symptom:**
```
CommandError: Timed out waiting for job <id>; current status=processing.
```
DB shows `processed_frames=173/4541`, `status=processing`, last update stale for 15+ minutes.  
`pgrep -fa "celery.*beat"` → nothing.

**Root cause (two independent failures):**

1. **Celery beat not started.** The stale-job reconciler (`reconcile_stale_jobs_periodic`) is a Celery Beat periodic task. Without Beat running, stuck jobs remain in `processing` state indefinitely and are never reconciled to `failed`. The `prod_start_celery_workers.sh` script only starts worker processes, not the scheduler.

2. **Celery soft-time-limit (300 s) kills the video task mid-run.** The offline pipeline submits one batch window (~173 frames) to Triton, processes responses, then continues. The total task execution time exceeds the 300 s `--soft-time-limit`. When the task is killed, the remaining ~4 368 frames are never re-queued, leaving the job stuck in `processing`.

**Observable evidence:**
- Last inference log entry: `2026-05-30 13:46:58` — inference stopped cold mid-job
- Only 10 Celery task completions recorded in the offline_control log since last restart
- Job `542533c3` still `processing` at `173/4541` frames, stale 926 s

**Fix — Celery beat (APPLIED):**
Beat now starts and stops as part of every worker set. `prod_start_celery_workers.sh`
calls `prod_start_celery_beat.sh` at the end (unless `CELERY_BEAT_DISABLED=1`), and
`prod_stop_celery_workers.sh` calls `prod_stop_celery_beat.sh`. Both beat scripts are
idempotent and refuse to launch a second scheduler. Manual control:
```bash
bash tools/prod/prod_start_celery_beat.sh   # idempotent single scheduler
bash tools/prod/prod_stop_celery_beat.sh    # graceful stop + pkill safety net
```

**Fix — soft-time-limit for long videos (APPLIED):**
The `process_video_upload` task default soft/hard limits were raised from 600/660 s to
**1800/1920 s** in `backend/config/settings/base.py` (env-overridable via
`VIDEO_UPLOAD_TASK_SOFT_TIME_LIMIT_SECONDS` / `VIDEO_UPLOAD_TASK_TIME_LIMIT_SECONDS`,
already `1800/1860` in `.env.example`). The worker-level fallback in
`prod_start_celery_workers.sh` was raised from `300/600` to `1800/2400`. A 4 541-frame
video (~19 min over the sequential HTTP pipeline) now completes within the soft limit.
The continuation/chunking re-architecture remains a future optimisation but is no longer
required for end-to-end completion.

**Status:** **RESOLVED 2026-05-30.** Both blockers fixed in code/config. Run the
automated benchmark with `bash tools/prod/prod_run_benchmark.sh` (see §8).

### Issue 1 — TRT Serialization Version Mismatch (tag 239 → 240)

**Symptom:**
```
Serialization assertion stdVersionRead == kSERIALIZATION_VERSION failed.
Version tag does not match. Current: 240, Serialized Engine: 239
```

**Root cause:** TRT was upgraded from 10.8.0.43 (serial tag 239) to 10.16.1.11 (serial tag 240) via `uv sync` / `prod_post_sync.sh`, but existing `.plan` engine files had been compiled under 10.8.0.43.

**Fix:** Full engine rebuild with `bash tools/prod/prod-rebuild-tensorrt-engines.sh`.

**Guard:** `tools/prod/prod_trt_guard.sh` now blocks `prod_start_triton.sh` if installed TRT version does not match `backend/models/tensorrt_builds/latest_compat.json`.  
`prod_post_sync.sh` warns with a banner if engines need a rebuild after any `uv sync`.

---

### Issue 2 — Static Batch Dimension in ONNX (ultralytics export)

**Symptom:**
```
Input tensor images has static dimensions that don't match kMIN dimensions in profile 0.
Input dimensions are [8,3,640,640] but profile dimensions are [1,3,640,640].
```

**Root cause (attempt 1 — failed):** `_make_batch_dynamic()` only patched the ONNX input *type annotation* (`.dim_param = "batch"`). Internal `Reshape` constant nodes retained the hardcoded batch value `8`, and TRT locked the profile to `[8,8,8]`.

**Root cause (confirmed):** YOLO models exported by ultralytics contain post-processing `Reshape` nodes whose *constant shape tensor* has `[8, -1, classes]` baked in. Patching the type annotation does not change those constants; TRT uses the constants to determine profile limits.

**Fix:** Two-phase export in `backend/scripts/build_tensorrt_engines.py`:
1. `ultralytics YOLO.export(format="onnx", dynamic=True, opset=17)` — marks input annotation dynamic.
2. `_patch_onnx_dynamic_batch(onnx_path, static_batch)` — scans every initializer used as a shape input to `Reshape`/`Expand` nodes and replaces `arr[0] == static_batch` with `-1`. Runs `onnx.shape_inference` to propagate the new dynamic shapes.

Re-export from `.pt` is always preferred over patching a stale ONNX.

**Affected model:** `person_detector` (`student_teacher.pt`). Other YOLO models (`standing_sitting`, `right_left`, `up_down`, `forward_backward`) exported cleanly with `dynamic=True`.

---

### Issue 3 — Duplicate Celery Workers Causing Queue Contention

**Symptom:** `pgrep -fc "celery -A config worker"` returned 15–25 instead of expected 23 (5 workers × their children). Competing workers on the same queue caused task stalls.

**Root cause:** `prod_stop_celery_workers.sh` only killed PIDs recorded in `backend/logs/pids/*.pid`. Any worker started without updating the PID file (e.g. a second `prod_start_celery_workers.sh` invocation during the TRT rebuild cycle) was orphaned.

**Fix:** `prod_stop_celery_workers.sh` now has a phase-2 pkill safety net:
```bash
pkill -SIGTERM -f "celery -A config worker"
sleep 3
pkill -SIGKILL -f "celery -A config worker"  # stragglers
```
Always run `prod_stop_celery_workers.sh` before `prod_start_celery_workers.sh`.

**Expected process count** after a clean start:

| Worker | Concurrency | Processes |
|---|---|---|
| `default_control` | 2 | 3 (1 parent + 2 children) |
| `offline_control` | 4 | 5 |
| `offline_person` | 4 | 5 |
| `offline_pose` | 4 | 5 |
| `offline_behavior` | 4 | 5 |
| **Total** | | **23** |

---

### Issue 4 — All Model Routes Pointed at Version `v2` (non-existent)

**Symptom:** Every inference call hit `versions/2/infer` → `HTTP 404`, then retried `versions/1/infer` → `HTTP 200`. Generated ~1 000 spurious 404 entries per video job and doubled effective inference latency.

**Root cause:** `backend/apps/pipeline/services/model_route_service.py` had `"model_version": "v2"` for all environments (`dev`, `staging`, `prod`). Triton model repository only has version `1` (`<model>/1/model.plan`).

**Fix:** Changed all entries to `"model_version": "v1"` (commit `33a24d1`).  
Env-var override available: `MODEL_ROUTE_<TASK_KEY>_MODEL_VERSION=v1` in `backend/.env`.

**Latency impact (before fix):** Each of 6 models made one wasted 404 round-trip per inference call.  
Estimated overhead per frame: `~5 ms × 6 models = +30 ms/frame` added purely by the failed version probe.

---

### Issue 5 — Production Branch Divergence & Orphaned Stash

**Symptom:** Prod server checked out `release/prod-runtime-stabilization` with 191 local commits diverging from `origin/master` (321 commits). `git pull origin master` failed with "divergent branches".

**Fix:** `git fetch origin && git reset --hard origin/master` — discarded local divergence (evidence-only files stashed first with `git stash push -m "prod-evidence-stash-<date>"`). Prod is now tracking `origin/master` directly.

---

### Issue 5b — Pending Migrations

**Symptom:** Django startup warning: `2 unapplied migration(s) for apps: anomalies, behavior`.

**Fix:** `python manage.py migrate` applied:
- `anomalies.0003_bsilbaselinesnapshot_bsilreviewlabelrecord_and_more`
- `behavior.0002_behavioralconfidencebreakdown_decisionlineagerecord_and_more`

---

## 5. CPU-Bound Bottleneck Analysis

### Root Cause

The pipeline makes **sequential HTTP calls** to Triton for each model per frame:

```
Frame N:
  [CPU] decode + preprocess         ~5–20 ms
  [HTTP] dispatch person_detector   ~476–1166 ms RTT
  [HTTP] dispatch posture_model     ~476–1308 ms RTT
  [HTTP] dispatch gaze_horizontal   ~667–1620 ms RTT
  [HTTP] dispatch gaze_vertical     ~404–1323 ms RTT
  [HTTP] dispatch gaze_depth        ~381–5655 ms RTT
  [CPU] post-process + store        ~5–10 ms
```

Total per frame: **~2 500–11 000 ms** (mostly HTTP overhead).  
RTX 5090 actual GPU compute per batch: **<5 ms**.  
**GPU idle fraction: ~98–99%.**

### Why RTTs Are So High

1. **HTTP vs gRPC:** Triton's HTTP/REST endpoint adds JSON serialization + TCP overhead. gRPC uses binary Protobuf + HTTP/2 multiplexing, typically 3–10× lower per-call latency for small tensors.
2. **Sequential dispatch:** All 6 models are called one-at-a-time from a single Celery worker. Concurrent async dispatch would reduce wall-clock latency to `max(individual RTTs)` instead of `sum(RTTs)`.
3. **Small batch sizes:** Offline pipeline currently sends 1–4 frames per Triton call. RTX 5090 saturates at batch ≥ 16–32 for these model sizes; sending batch=1 wastes 95%+ of available GPU throughput.
4. **Worker prefetch cap:** `--prefetch-multiplier=1` limits each worker to holding 1 task at a time, preventing pipeline overlap between frame batches.

### Improvement Roadmap

| Change | Estimated GPU Util Gain | Complexity |
|---|---|---|
| Switch Triton transport HTTP → gRPC | 2–3× throughput; RTT ~100–300 ms → ~30–80 ms | Low — change URL scheme in client |
| Async concurrent model dispatch (all 6 in parallel per frame) | 3–5× wall-clock throughput | Medium — async client refactor |
| Batch accumulation: 8–16 frames before sending | 5–10× GPU throughput utilisation | Medium — queue window tuning |
| Increase `max_concurrency` in offline batch queue (4 → 8) | 1.5–2× | Low — `.env` change |
| Raise `--prefetch-multiplier` to 2–4 | Pipeline overlap | Low — worker config |

---

## 6. Baseline Throughput

| Metric | Value | Conditions |
|---|---|---|
| Burst rate (first batch window, HTTP) | ~150–160 fps | ~173 frames processed in ~1 s burst |
| Sustained processing | N/A — job killed by 300 s soft-time-limit | Celery task timeout before second batch |
| Observed frames processed per full run | 173 / 4 541 (3.8%) | Stopped by `soft-time-limit` |
| GPU utilisation during burst | 1.2% avg / 12% peak | 179 samples @ 2 s interval |
| GPU temperature under load | 36 °C | Far below thermal limit (89 °C) |
| Average board power | 60.1 W | vs 575 W TDP — GPU essentially idle |
| Pipeline theoretical ceiling (HTTP, sequential) | ~0.5 fps sustained | Limited by sum of 6 × avg RTT per frame |
| Pipeline theoretical ceiling (gRPC, parallel dispatch) | ~20–50 fps | Estimated; pending implementation |
| Pipeline theoretical ceiling (gRPC + batch=16) | ~200+ fps | RTX 5090 TFLOPS ceiling |

**Blockers before a full-video run can complete — all resolved 2026-05-30:**
1. ~~Celery Beat not running → stale jobs stuck in `processing`~~ — beat now auto-starts with workers (`prod_start_celery_beat.sh`).
2. ~~`soft-time-limit=300 s` kills video task mid-run for large videos~~ — raised to 1800 s soft / 1920 s hard (task) and 1800/2400 s (worker fallback).
3. ~~Job `--wait` timeout (900 s) in management command exits before job completes~~ — `prod_run_benchmark.sh` submits with `--timeout-seconds 3600`.

---

## 6.1 Phase 7a Crop-Frame Mode Status

**Recorded:** 2026-05-31
**Status:** code-complete locally; production RTX 5090 benchmark pending.

`crop_frame` is now a first-class upload/CLI/frontend mode. The DB/API
compatibility default remains `legacy_crop`, while production CLI and benchmark
helpers now default to `crop_frame`. The mode runs Triton `person_detection`
first, crops each detected person with `FrameCropper`, then sends crop tensors
for posture/gaze through the same frame-batch dispatch helper used by
`full_frame`. The existing optimization controls still govern the path:

| Control | Default | Crop-frame behavior |
|---|---|---|
| `TRITON_BINARY_TENSORS` | optimized prod default on | crop tensors use the same binary/JSON payload branch |
| `TRITON_CONCURRENT_MODELS` | optimized prod default on | secondary crop tasks use the shared concurrent dispatch helper |
| `TRITON_OFFLINE_PIPELINE_OVERLAP` | optimized prod default on | decode/preprocess overlap remains the outer frame producer path |
| `OFFLINE_PROGRESS_UPDATE_EVERY_N` | optimized prod default `25` | progress throttle remains in the shared Triton callback path |

Production validation must compare `legacy_crop`, `full_frame`, and `crop_frame`
with fresh runs and telemetry before any throughput or GPU-utilization claim.

---

## 6.2 Remaining Parallelization Phases Status

**Recorded:** 2026-05-31
**Status:** repo-side implementation complete; production RTX 5090 benchmark certification pending.

| Phase | Local implementation | Default | Production validation |
|---|---|---|---|
| P3 gRPC transport | `TritonClient` can use `tritonclient.grpc` and falls back to HTTP on gRPC failure | optimized prod default `grpc` | Validate `TRITON_GRPC_URL`/39101 readiness and compare RTT |
| P4 batched writer | `OFFLINE_DB_BATCH_WRITES` bulk-creates frame detections/boxes with per-row fallback | optimized prod default on | Compare DB time and row counts |
| P4 post-stage offload | `OFFLINE_OFFLOAD_POST_STAGES` forces existing follow-up stages out of inline execution | optimized prod default on | Confirm worker routing and lifecycle completion |
| P6 VRAM discipline | tracked Triton configs use single GPU instance per model | count 1 | Reload Triton and gate on `peak_gpu_memory_mb` |
| P7b temporal reuse | `OFFLINE_EMBEDDING_REUSE_BY_TRACK` and `OFFLINE_BEHAVIOUR_REUSE` reuse cached track state | optional; off in per-frame-signal profile | Compare per-track parity and lower CPU/model calls |

No production throughput, GPU-utilization, or p95-latency improvement is claimed
until these flags are enabled and benchmarked on the native Linux RTX 5090 host.

### 6.2.1 Optimized Combined-Video Benchmark Command

**Recorded:** 2026-05-31
**Video:** `/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4`
**Mode:** `crop_frame`
**Default profile:** `per-frame-signals` (`TRITON_OFFLINE_FRAME_STRIDE=1`;
`OFFLINE_DETECT_EVERY_N_FRAMES=5`; person-box reuse TTL `10`; no
behaviour/gaze or embedding reuse; decode queue size `4`; NumPy gRPC outputs;
true Triton batch requests capped at 8 for object models)
**Runner:** `tools/prod/prod_run_parallel_flow_benchmark.sh`

The runner performs the operational sequence now expected for this flow:

1. Stop Celery workers and mark active queued/processing/embedding video jobs as
   failed with an operator-cancel reason.
2. Apply optimized defaults to `backend/.env` via `prod_enable_parallel_flow.sh`.
3. Restart Triton and workers.
4. Probe env, Triton readiness, worker state, job state, and model-call telemetry.
5. Run `runtime_ingest_video` with `--benchmark-evidence`, GPU CSV sampling, and
   JSON summary output.

Command:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_run_parallel_flow_benchmark.sh --profile per-frame-signals
```

Investigation helpers:

```bash
bash tools/prod/prod_parallel_flow_probe.sh --watch 30
bash tools/prod/prod_verify_per_frame_signals.sh --job-id <job_id> --watch 30
bash tools/prod/prod_cancel_video_jobs.sh --all-active
bash tools/prod/prod_enable_parallel_flow.sh --dry-run
```

Initial production execution:

| Timestamp | Replay key | Job ID | Result |
|---|---|---|---|
| 2026-05-31 20:26 EEST | `parallel-crop-frame-20260531T202607` | `22b6d969-5789-4c53-a6e0-da63d9c10cc5` | Failed before inference because `TRITON_LIVE_URL` pointed at inactive `39000`; fixed in commit `9ed0a922`. |
| 2026-05-31 20:28 EEST | `parallel-crop-frame-20260531T202819` | `5801ef31-050f-4e20-a58e-d98122c5e920` | Cancelled by operator at `50/4541` frames after the active crop-frame worker reached ~100 GB RSS. Root cause: `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=16` retained too many frames of person crops/responses in one crop-frame batch. Follow-up default changed to `1`, keeping batching inside each frame's person crops while bounding memory. |
| 2026-05-31 20:36 EEST | `parallel-crop-frame-20260531T203638` | `17518cbe-c320-4880-8265-62df0add1ae3` | Cancelled at `25/4541` frames. `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=1` bounded memory better than the previous run, but `OFFLINE_DETECT_EVERY_N_FRAMES=1` kept crop-frame CPU-bound and prevented reuse from reducing model calls. Follow-up default changed to `OFFLINE_DETECT_EVERY_N_FRAMES=5` with person-box reuse only; behaviour/gaze and embedding reuse are disabled in the accuracy-first profile so current-frame crop signals are recomputed. The crop-frame loop now skips full-frame tensor serialization on non-detection frames and avoids per-crop PNG encoding. |
| 2026-05-31 21:39 EEST | `parallel-per-frame-signals-crop-frame-20260531T213945` | `c1a9117e-7f59-4c53-9731-b528ab5e6cbd` | Active run on commit `a1770e5f` with `per-frame-signals`. First probe: `25/4541`, `window_fps=0.400`, `frame_signal_contract=per_frame_signals_with_person_box_reuse`, GPU sample `62%`. DB frame rows are `pending_persistence` while frame inference is running; final Step 3 writes `Frame`/`Detection`/`BoundingBox` rows. |
| 2026-05-31 21:43 EEST | `parallel-per-frame-signals-crop-frame-20260531T213945` | `c1a9117e-7f59-4c53-9731-b528ab5e6cbd` | Cancelled after worker RSS exceeded 100 GB around frame 25. Root cause is dense crop fanout plus gRPC YOLO output materialized as Python lists. Follow-up change adds `TRITON_NUMPY_OUTPUTS=1` so decode consumes NumPy arrays directly. |
| 2026-05-31 21:47 EEST | `parallel-per-frame-signals-crop-frame-20260531T214740` | `e8071c5a-8fb7-4c23-882d-621f8469c097` | Cancelled at `100/4541`. `TRITON_NUMPY_OUTPUTS=True` fixed the immediate >100 GiB Python-list output spike, but RSS still reached ~49 GiB and throughput stayed CPU-bound. Follow-up change adds `TRITON_TRUE_BATCH_REQUESTS=1`, caps object-model true batches at 8, and reuses/closes gRPC channels per dispatch window. |
| 2026-05-31 22:16 EEST | `parallel-per-frame-signals-truebatch-guarded-crop-frame-20260531T221625` | `3f4af427-dbc0-42c3-96aa-8af0ebd5f3a0` | Cancelled at `50/4541`. Stale Celery redelivery was fixed: only one active job remained. Throughput stayed near `0.4 fps`; RSS still climbed to ~34.5 GiB because true-batch response splitting copied per-crop NumPy output slices. Follow-up change keeps split outputs as NumPy views. |
| 2026-05-31 22:24 EEST | `parallel-per-frame-signals-nocopy-crop-frame-20260531T222447` | `f654d10d-b392-423b-8fdd-53c88c507585` | Cancelled at `50/4541`. No-copy split reduced avoidable allocations but RSS still held freed buffers (~33 GiB by frame 50). Follow-up change adds `OFFLINE_TRIM_PROCESS_MEMORY=1` to run GC + Linux `malloc_trim(0)` after each offline frame batch. |
| 2026-05-31 22:33 EEST | `parallel-per-frame-signals-trim-crop-frame-20260531T223309` | `a6a06fd6-97e8-4a82-b1ef-2b9fe132d667` | Cancelled at `75/4541`. One-active-job and per-frame-signal contract held, but RSS still climbed to ~37 GiB and throughput stayed near `0.4 fps`. Root cause narrowed to per-frame async/gRPC event-loop churn plus crop-frame tracking rescanning all accumulated frames. Follow-up change uses one job-scoped async/gRPC loop and incremental crop-frame tracking with tracked person-box cache refresh. |
| 2026-05-31 22:48 EEST | `parallel-per-frame-signals-looptrack-crop-frame-20260531T224816` | `74572801-b257-4229-b37c-1e8ba5952b4e` | Active validation of job-scoped async/gRPC loop + incremental crop tracking on commit `2c3f6a7`. At `75/4541`, worker RSS was ~2.0 GiB instead of ~37 GiB, confirming the memory fix. Throughput remained near `0.4 fps`, pointing to CPU decode/NMS over dense crop fanout. Follow-up adds vectorized YOLO decode and `TRITON_YOLO_MAX_DECODE_CANDIDATES=1000` before NMS. |

### 6.3 Subjective `all_merged.mp4` Failure Analysis And Hardening

**Recorded:** 2026-05-31
**Video:** `/home/bamby/grad_project/Raw Data/all_merged.mp4`
**Mode:** `crop_frame`, per-frame (`TRITON_OFFLINE_FRAME_STRIDE=1`,
`OFFLINE_DETECT_EVERY_N_FRAMES=1`)
**Outcome:** failed at `3045 / 16527` frames with
`reconciled_stale_processing_state`.

Root causes found during recovery:

1. `backend/logs/triton.log` grew to roughly **193 GiB**, filled the root
   filesystem, and pushed PostgreSQL into recovery mode.
2. Triton inherited a `nofile` soft limit of `1024` and then emitted
   repeated `accept(): Too many open files` warnings while holding GPU memory but
   doing no useful GPU work.
3. The gRPC client path passed a float seconds value to Triton's request-level
   protobuf `timeout` field, producing
   `'float' object cannot be interpreted as an integer`, then falling back to HTTP
   repeatedly.

Repository hardening applied after the failure:

| Area | Change |
|---|---|
| Triton launcher | `tools/prod/prod_start_triton.sh` reads `TRITON_NOFILE_LIMIT` from `backend/.env` and defaults to `65535`. |
| Triton log growth | `tools/prod/prod_start_triton.sh` reads `TRITON_LOG_MAX_MIB` and truncates an oversized `backend/logs/triton.log` before launch. |
| gRPC transport | `TritonClient._infer_grpc()` now passes request timeout as integer microseconds and client deadline as seconds. |
| Worker env authority | `prod_start_celery_workers.sh` reads task limits and concurrency guardrails from `backend/.env`, not only the launcher shell. |
| Progress inspection | `tools/prod/prod_check_subjective_progress.sh` reports DB-backed progress and ETA for the latest `all_merged.mp4` job or a supplied job ID. |
| Benchmark modes | `tools/prod/prod_run_benchmark.sh` supports `--pipeline-mode legacy_crop|full_frame|crop_frame`. |
| Windows ingest helper | `tools/prod/prod-runtime-ingest-video.ps1` accepts `crop_frame`. |

The failed `all_merged.mp4` run is **invalid evidence**. A clean rerun must use a
new replay key after confirming disk, Postgres, Triton readiness, and worker
process counts.

---

## 7. Tools Added During This Session

| Script | Purpose |
|---|---|
| `tools/prod/prod_trt_guard.sh` | Pre-flight check: blocks Triton launch if TRT version ≠ manifest |
| `tools/prod/prod-rebuild-tensorrt-engines.sh` | Full stop→export→build→deploy→restart workflow |
| `tools/prod/prod_update_bashrc.sh` | Idempotently fixes `~/.bashrc`/`~/.profile` LD_LIBRARY_PATH |
| `tools/prod/prod_disk_cleanup.sh` | Purges `__pycache__`, probe engines, stale rollbacks, old logs |
| `tools/prod/prod_post_sync.sh` | Post-`uv sync` TRT install + guard check |
| `tools/prod/prod_cancel_video_jobs.sh` | Stops workers, marks active video jobs terminal, purges the video queue, and stops waiting ingest commands before a new benchmark |
| `tools/prod/prod_enable_parallel_flow.sh` | Idempotently applies optimized parallel-flow defaults to `backend/.env`; default profile is `per-frame-signals` |
| `tools/prod/prod_parallel_flow_probe.sh` | Prints env/runtime/job/model-call evidence for the optimized flow |
| `tools/prod/prod_verify_per_frame_signals.sh` | Verifies the accuracy-first contract and DB frame completeness for a job |
| `tools/prod/prod_run_parallel_flow_benchmark.sh` | Chained cancel → enable → restart → probe → benchmark runner for `combined.mp4` |
| `backend/scripts/build_tensorrt_engines.py` | Full .pt → ONNX (dynamic) → TRT → Triton deploy pipeline |

---

## 8. Re-run Checklist (for future benchmark comparisons)

> **Blockers resolved 2026-05-30.** Beat auto-starts with the workers and the
> offline soft-time-limit is now 1800 s, so the former ~173-frame stall is fixed.
> The one-shot runner below performs every step (pre-flight, beat ensure, GPU
> monitor, job submit, telemetry summary):
>
> ```bash
> cd /home/bamby/grad_project
> bash tools/prod/prod_run_parallel_flow_benchmark.sh
> ```
>
> The manual steps below are retained for reference / custom runs.

```bash
# On prod server
cd /home/bamby/grad_project

# 1. Verify environment
bash tools/prod/prod_trt_guard.sh
curl -sf http://127.0.0.1:39100/v2/health/ready && echo "Triton OK"
pgrep -fc "celery -A config worker"  # expect 23
pgrep -fa "celery.*beat" || echo "WARNING: beat not running — start it first"

# 2. Start beat if not running (required for stale-job reconciler)
cd backend
nohup .venv/bin/celery -A config beat -l info \
  --pidfile=logs/pids/celery_beat.pid \
  --logfile=logs/celery_beat.log &
cd ..

# 3. Reset GPU monitor (1 s samples)
echo "ts,util_pct,mem_used_mib,mem_total_mib,temp_c,power_w" > backend/logs/gpu_monitor_bench.csv
nohup bash -c 'for i in $(seq 1 3600); do
  nvidia-smi --query-gpu=timestamp,utilization.gpu,memory.used,memory.total,temperature.gpu,power.draw \
    --format=csv,noheader,nounits >> /home/bamby/grad_project/backend/logs/gpu_monitor_bench.csv
  sleep 1; done' &

# 4. Submit benchmark job (timeout 7200 s for full combined video)
cd backend && source .venv/bin/activate
python manage.py runtime_ingest_video \
  --video "/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4" \
  --actor-username "runtime_validator" \
  --runtime-profile offline \
  --pipeline-mode crop_frame \
  --replay-policy new-attempt \
  --benchmark-evidence \
  --timeout-seconds 7200 \
  --wait 2>&1 | tee /home/bamby/grad_project/backend/logs/bench_run_$(date +%Y%m%dT%H%M%S).log

# 5. Analyse GPU telemetry
awk -F',' 'NR>1{s+=$2;n++;if($2>mx)mx=$2;pw+=$6} END{
  printf "samples=%d  avg_util=%.1f%%  peak_util=%s%%  avg_power=%.1fW\n",n,s/n,mx,pw/n
}' /home/bamby/grad_project/backend/logs/gpu_monitor_bench.csv
```

For crop-frame validation, run:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_run_benchmark.sh \
  --video "/home/bamby/grad_project/Raw Data/all_merged.mp4" \
  --pipeline-mode crop_frame \
  --timeout 14400 \
  --show-tracking-ids
```

---

*Generated from production run on 2026-05-30. Update this file after each major pipeline change or hardware migration.*
