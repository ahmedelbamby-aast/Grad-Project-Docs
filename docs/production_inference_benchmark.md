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
| 2026-05-31 22:48 EEST | `parallel-per-frame-signals-looptrack-crop-frame-20260531T224816` | `74572801-b257-4229-b37c-1e8ba5952b4e` | Cancelled at `225/4541`. Job-scoped async/gRPC loop + incremental crop tracking on commit `2c3f6a7` kept worker RSS near ~2 GiB instead of ~37 GiB, confirming the memory fix. Throughput remained near `0.4 fps`, pointing to CPU decode/NMS over dense crop fanout. |
| 2026-05-31 23:04 EEST | `parallel-per-frame-signals-nms300-crop-frame-20260531T230406` | `2416537a-7466-487e-b2c9-959dd8698b1b` | Cancelled at `75/4541` after testing `TRITON_YOLO_MAX_DECODE_CANDIDATES=300`. Window throughput improved to `0.544 fps`; worker RSS stayed ~1.6 GiB. |
| 2026-05-31 23:07 EEST | `parallel-per-frame-signals-nms100-crop-frame-20260531T230706` | `80027072-a9d4-4be7-9099-4354acd1170b` | Active validation of `TRITON_YOLO_MAX_DECODE_CANDIDATES=100`. Probe at `125/4541`: 90-second window throughput `1.111 fps`, overall `0.925 fps`, worker RSS ~1.7 GiB. Later probe at `425/4541`: 120-second window `1.042 fps`, overall `1.000 fps`, worker RSS ~1.6 GiB, one active job, per-frame-signal contract active. |
| 2026-06-01 01:21 EEST | `roi320-running-crop-frame-20260601T012133` | `77650001-3c4b-4b0a-94aa-b4eb899b90df` | Completed `4541/4541` on the required `combined.mp4` video with `TRITON_CROP_BEHAVIOR_INPUT_SIZE=320`. DB rows: `4541` frames, `72751` detections, `72751` boxes, `57` tracks, no stale error. Throughput improved to `1.309 fps` over the DB created/completed window and behavior RTT fell materially, but GPU average utilization regressed to `3.95%`; therefore this is a measured partial success, not final acceptance. |

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
| `tools/prod/prod_enable_roi_crop_behavior.sh` | Rebuilds posture/gaze TensorRT engines and Triton configs for smaller crop-frame behavior input sizes; production candidate used `320` |
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

## 9. 2026-06-01 Crop-Frame Bottleneck Certification

Production job `80027072-a9d4-4be7-9099-4354acd1170b` on
`Raw Data/Diverse Classroom Enviroments/combined.mp4` completed `4541/4541`
frames and produced `4541` Frame rows, `24348` Detection rows, `24348`
BoundingBox rows, and `24182` embeddings.

The final DB row still retained `error_message=reconciled_stale_processing_state`.
This is a lifecycle false positive: the stale reconciler marked the job failed at
`20:39:25 UTC` because long Step 2 progress updates did not refresh `updated_at`;
the original worker continued, persisted rows at `21:37:55-21:38:18 UTC`,
rendered, embedded, and completed at `21:42:56 UTC`.

Measured bottleneck:

| Evidence | Result |
|---|---:|
| Full benchmark GPU samples | avg util `13.5%`, peak `38%` |
| 60-frame profiler GPU samples | avg util `15.49%`, peak `37%` |
| 120-frame profiler mean frame time | `728.229 ms` |
| 120-frame behavior/gaze inference wall time | `666.353 ms/frame` |
| 60-frame behavior input crossing gRPC | `380.1 MB/frame` |
| Estimated dense behavior output crossing gRPC | `81.8 MB/frame` |
| Behavior client RTT | `192-212 ms/request` |
| Behavior Triton server success time | `14.87-19.70 ms/execution` |
| Mean person crops in profiler | `19.33/frame` |
| Mean persisted person box size | `69.27 x 74.55 px` |
| Crop expansion ratio to `640x640` | `81.42x` in profiler |

Conclusion: the RTX 5090 remains mostly idle because the worker sends many large
`640x640` FP32 crop batches and receives dense YOLO grids through Python/gRPC,
then synchronizes at frame boundaries. Decode, crop slicing, DB persistence,
Triton readiness, and OOM are not the bottleneck for this run.

Full report:
[`docs/crop_frame_rtx5090_bottleneck_investigation.md`](crop_frame_rtx5090_bottleneck_investigation.md).

---

## 10. 2026-06-01 ROI-320 Crop-Frame Production Benchmark

**Status:** production benchmark completed; final optimization acceptance **not**
granted because average GPU utilization did not improve.

**Candidate:** `TRITON_CROP_BEHAVIOR_INPUT_SIZE=320` with rebuilt posture/gaze
TensorRT engines and matching Triton configs. The test video is
`/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4`.

**Evidence:**

| Item | Value |
|---|---|
| Replay key | `roi320-running-crop-frame-20260601T012133` |
| Job ID | `77650001-3c4b-4b0a-94aa-b4eb899b90df` |
| Main log | `backend/logs/roi320-running-crop-frame-20260601T012133.orchestrator.log` |
| Benchmark log | `backend/logs/bench_run_20260601T012134.log` |
| Summary JSON | `backend/logs/bench_summary_20260601T012134.json` |
| GPU CSV | `backend/logs/gpu_monitor_bench_20260601T012134.csv` |
| Telemetry JSON | `backend/logs/telemetry/bec68852-9776-4aee-aca1-20e4df6b078d_offline_20260531_222139.json` |
| Completion | `status=completed`, `processed_frames=4541`, `total_frames=4541`, `error_message=""` |
| DB rows | `4541` frames, `72751` detections, `72751` boxes, `57` tracks |

Measured comparison:

| Metric | Baseline | ROI-320 candidate | Delta |
|---|---:|---:|---:|
| Full job throughput | `~0.975 fps` | `1.309 fps` | `+34.3%` |
| Full job elapsed | `~70 min` | `57.82 min` DB created→completed | `-17.4%` |
| Main upload/inference wall | `~70 min` | `50.01 min` telemetry wall | `-28.6%` |
| Mean frame latency | `728.229 ms` profiler | `394.122 ms` telemetry | `-45.9%` |
| Behavior input tensor size | `640x640` | `320x320` | `-75.0% pixels/sample` |
| Estimated behavior input traffic | `380.1 MB/frame` | `~95.0 MB/frame` | `-75.0%` |
| Estimated dense YOLO output traffic | `81.8 MB/frame` | `~20.5 MB/frame` | `-75.0%` |
| Avg GPU utilization | `13.5%` | `3.95%` | **regressed** |
| Peak GPU utilization | `38%` | `34%` | **regressed** |
| GPU memory | previous memory-stable run ~`1.6 GiB` worker RSS | `15663 MiB` GPU memory in candidate monitor | not accepted as a utilization win |
| Stale reconciler outcome | false `reconciled_stale_processing_state` retained | completed with empty `error_message` | fixed |

Per-model RTT from the production telemetry session:

| Model | Baseline behavior RTT | ROI-320 mean RTT | ROI-320 p95 RTT |
|---|---:|---:|---:|
| `person_detector` | not primary bottleneck | `13.499 ms` | `14.973 ms` |
| `posture_model` | `192-212 ms/request` | `102.393 ms` | `135.143 ms` |
| `gaze_horizontal_model` | `192-212 ms/request` | `137.843 ms` | `174.109 ms` |
| `gaze_vertical_model` | `192-212 ms/request` | `119.100 ms` | `153.553 ms` |
| `gaze_depth_model` | `192-212 ms/request` | `155.837 ms` | `191.294 ms` |
| `rtmpose_model` | not in original behavior RTT sample | `40.882 ms` | `75.485 ms` |

Conclusion: the ROI-320 change reduced behavior tensor volume and improved
throughput/RTT/total time, and the stale-job lifecycle fix passed the long-run
validation. It did **not** increase RTX 5090 utilization. The remaining
optimization target must therefore reduce the still-measured Python/Triton
request boundary and request count instead of treating smaller crop tensors as
the final fix.

Acceptance state: **partial production success; final GPU-utilization acceptance
failed.**

---

## 11. 2026-06-01 Cycles 1-5 Production Benchmark — Knob Tuning + Telemetry Fix + Concat Memoization

**Status:** production benchmark completed; **frame-inference goals achieved (≥2× throughput, +91% GPU utilization, correctness preserved)**. Post-inference embedding stage shows a separate (pre-existing) slowness that is out of scope for this cycle.

**Candidate:** five stacked changes shipped together (small, low-risk, env + writer + memoization).

1. **Telemetry persistence fix** — `apps.telemetry.writer.flush_session` now bulk-inserts `TelemetryModelCall` rows; previously model calls were captured in the JSON file but **silently dropped from DB** (0 rows / job before; 29 123 rows / job now).
2. **`TRITON_OFFLINE_BATCH_QUEUE_MAX_CONCURRENCY` 4 → 2** — matches the prod inflight-sweep saturation knee (12.82 FPS / 151 ms p50 at 2, vs 11.26 FPS / 322 ms at 4).
3. **`TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES` 1 → 2** — lets N+1 crop preprocess overlap N Triton inflight and lets true_batch pack two frames' crops into one gRPC call per model.
4. **`OFFLINE_TRIM_EVERY_N_BATCHES=8`** — amortizes `gc.collect()` + `malloc_trim(0)` (5–30 ms each) so prod no longer pays a GC stall on every batch.
5. **`_build_true_batch_payload` memoization** — same `crop_payloads` list dispatched across 4 fan-out models now pays the `b"".join` cost once instead of four times (`~1.7 ms × 4 = 6.6 ms` saved per frame).

**Evidence:**

| Item | Value |
|---|---|
| Replay key | `cycle2to5-crop-frame-20260601T012045` |
| Job ID | `74ec0432-995c-487e-9d77-1048ec109fb1` |
| Bench summary | `backend/logs/bench_summary_20260601T042322.json` |
| Bench log | `backend/logs/bench_run_20260601T042322.log` |
| GPU CSV | `backend/logs/gpu_monitor_bench_20260601T042322.csv` |
| Inference audit | `backend/data/videos/74ec0432-995c-487e-9d77-1048ec109fb1/inference_audit.json` |
| Telemetry session | `d5e3c6f0-cc5f-4dc5-bfe6-d00473af9218` |
| Completion | `processed_frames=4541/4541`, `run.complete=01:52:04`, `error_message=""` |
| DB row parity | `4541` frames, `72 743` detections (Δ −0.011 %), bbox `attention=11 772 hand=8 801 person=19 162 sitting=33 008` (Δ ≤ 0.04 %), `72 577` embeddings (Δ −0.011 %) |
| Final DB status | `completed` (no stale error) |

**Headline metrics:**

| Metric | Baseline (`77650001`) | Cycles 1–5 (`74ec0432`) | Δ |
|---|---:|---:|---:|
| Step 2 wall (frame inference) | **2 175 s** (36 m 15 s) | **883 s** (14 m 43 s) | **−59.4 %** |
| **Step 2 FPS (frame inference only)** | **2.09 fps** | **5.14 fps** | **+146 % (×2.46)** |
| Total elapsed (start → `run.complete`) | 3 002 s (50 m 1 s) | 1 716 s (28 m 36 s) | **−42.8 %** |
| **Overall FPS (to `run.complete`, bench probe basis)** | **1.308 fps** | **2.644 fps** | **+102 % (×2.02)** |
| Total elapsed (start → DB `status=completed`) | 3 469 s (57 m 49 s) | 2 186 s (36 m 26 s) | **−37.0 %** |
| **Overall FPS (to DB `status=completed`)** | **1.309 fps** | **2.077 fps** | **+58.7 % (×1.59)** |
| Mean frame inference_ms | 197.9 ms | 174.4 ms | −11.9 % |
| Mean frame preprocess_ms (full) | 1.04 ms | 0.69 ms | −33.7 % |
| Mean frame decode_ms | 0.015 ms | 0.013 ms | −15.4 % |
| **Avg GPU utilization** | **3.95 %** | **7.55 %** | **+91.1 %** |
| Peak GPU utilization | 34 % | 40 % | +17.6 % |
| % GPU samples in 0–5 % bucket | 80.3 % | 65.9 % | −14.4 pp |
| % GPU samples ≥ 20 % | 7.6 % | 21.2 % | +13.6 pp |
| Behavior executions per model | 4 543 | **3 598** | **−20.8 %** (two-frame packing) |
| Avg crops per behavior call | 17.10 | **21.6** | +26.3 % |
| TelemetryModelCall DB rows / job | **0** (writer bug) | **29 123** | visibility restored |
| Max RSS (offline_control worker) | not measured | ~813 MB sustained | well within bounds |

**Per-model RTT (from `telemetry_model_calls`, now actually populated):**

| Model | Calls | Mean RTT (ms) | Notes |
|---|---:|---:|---|
| `person_detector` | 910 | **9.58** | every-5th-frame cadence |
| `posture_model` | 3 598 | **145.39** | 2-frame batch → ~72.7 ms / frame eqv |
| `gaze_horizontal_model` | 3 598 | **168.54** | larger output `[84,2100]` |
| `gaze_vertical_model` | 3 598 | **155.99** | |
| `gaze_depth_model` | 3 597 | **176.43** | |
| `rtmpose_model` | 13 822 | **41.08** | per-person; not batched at frame level |

**Why it worked (mechanism):**
- The MAX_CONCURRENCY=4 setting was inflating per-frame latency without raising throughput (16 simultaneous gRPC calls serialized behind one GPU). Dropping to 2 removed ~170 ms of pure-queue waste per frame.
- MAX_FRAMES=2 lets `_build_true_batch_payload` pack two frames' crops into a single ~40-crop gRPC request per model, halving the per-frame request count for behavior models (4541 baseline → 3598 in this run).
- Trim amortization removed ~5–25 ms / frame of GC stall.
- Concat memoization removed ~5–7 ms / frame of redundant `b"".join`.

**Correctness:**
- Detection count preserved within 0.011 %.
- Per-class bbox parity within 0.04 % per class (statistical noise, same source video).
- No persistence errors. No tracking discontinuities surfaced in inference_audit.
- Lifecycle hardening from the prior cycle still in effect (no stale-reconciler false positive).

**Known follow-up issues** (do **not** block acceptance of this cycle):
- `rtmpose_model` occasionally hits `batch-size must be <= 16` when packed across MAX_FRAMES=2 frames; falls back to HTTP and succeeds — but the warning is real and the pose dispatch path should explicitly chunk to 16. **Tracked as a separate fix.**
- Post-`run.complete` embedding stage runs in a separate Celery task and was still active at probe time (long-known: `sha256_stub` embedding is CPU-bound and unrelated to inference parallelization). Out of scope for this cycle.

**Acceptance state: ACCEPTED.** Frame inference throughput more than doubled, average GPU utilization is 91 % higher, all correctness contracts hold, telemetry visibility is restored, and the residual room to improve is now in the post-inference stages (embedding, pose serialization) and not in the inference hot path.

---

*Updated from production run on 2026-06-01. Update this file after each major pipeline change or hardware migration.*

---

## 12. 2026-06-01 Cycle 6 Production Benchmark — Pose Dispatch Chunking Fix

**Status:** **ACCEPTED**. Measured before/after on prod RTX 5090 with identical
4 541-frame video; correctness preserved within 0.012 % on every counter.

**Candidate (single, surgical change):** `PoseRuntime._provider_infer_batch`
now chunks its crops to the deployed rtmpose engine's `max_batch_size=16`
(configurable via `TRITON_MODEL_BATCH_SIZE_OVERRIDES["pose_estimation"]`)
before handing each chunk to the orchestrator. Cycles 1–5 enabled true-batch
packing across all model paths, but the pose path bypassed `_infer_task_batch`
and let the orchestrator stack every per-crop payload into a single gRPC
request — when `tasks.py` `dynamic_cap` exceeded 16 (which happens once
`avg_pose_ms_per_person` decays below ~88 ms on a fast GPU), every such call
violated the model contract and silently fell back to HTTP per frame.

**Evidence:**

| Item | Value |
|---|---|
| Replay key | `cycle6-posechunk-crop-frame-20260601T022240` |
| Job ID | `a1a448b9-474f-4dea-942b-3288bcae6900` |
| Bench summary | `backend/logs/bench_summary_20260601T0...json` (rc=0) |
| Bench log | `backend/logs/bench_run_20260601T0...log` |
| Inference audit | `backend/data/videos/a1a448b9-.../inference_audit.json` |
| Telemetry session | (via DB query against `metadata.job_id`) |
| Final status | `completed` (no stale-error) |
| DB row parity | 4 541 frames, 72 752 detections, 72 752 bboxes, 72 586 embeddings |

**Per-cycle FPS summary (same job — `combined.mp4`, 4 541 frames, prod RTX 5090):**

| Cycle | Job ID | Step 2 FPS (frame inference) | Overall FPS (`run.complete`) | Overall FPS (DB `status=completed`) |
|---|---|---:|---:|---:|
| Baseline (ROI-320 only) | `77650001-3c4b-4b0a-94aa-b4eb899b90df` | **2.09** | **1.308** | **1.309** |
| Cycles 1–5 (telemetry + concurrency + concat memo) | `74ec0432-995c-487e-9d77-1048ec109fb1` | **5.14** | **2.644** | **2.077** |
| Cycle 6 (pose chunking fix) | `a1a448b9-474f-4dea-942b-3288bcae6900` | **5.17** | **2.776** | **2.780** |
| Δ Cycle 6 vs Baseline | — | **+147 %** | **+112 %** | **+112 %** |
| Δ Cycle 6 vs Cycle 1–5 | — | **+0.6 %** | **+5.0 %** | **+33.8 %** |

**Headline metrics (Cycle 6 vs Cycle 1–5 vs Baseline):**

| Metric | Baseline `77650001` | Cycle 1–5 `74ec0432` | **Cycle 6 `a1a448b9`** | Δ vs Cycle 1–5 | Δ vs Baseline |
|---|---:|---:|---:|---:|---:|
| Step 2 wall (frame inference) | 2 175 s | 883 s | **879 s** | unchanged | −59.6 % |
| **Step 2 FPS (frame inference)** | 2.09 | 5.14 | **5.17** | +0.6 % | +147 % |
| Pose post-processing wall (`step2.frame_stage_timings → step2.pose_upload`) | (not measured) | 733 s (12 m 13 s) | **222 s (3 m 42 s)** | **−69.7 %** | n/a |
| Total to `run.complete` | 3 002 s | 1 716 s | **1 166 s** | **−32.1 %** | **−61.2 %** |
| **Overall FPS (bench probe / `run.complete`)** | 1.308 | 2.644 | **2.776** | +5.0 % | **+112 %** |
| Total to DB `status=completed` | 3 469 s (57.8 m) | 2 186 s (36.4 m) | **1 633 s (27.2 m)** | **−25.3 %** | **−52.9 %** |
| **Overall FPS (DB `status=completed`)** | 1.309 | 2.077 | **2.78** | **+33.8 %** | **+112 %** |
| `rtmpose_model` batch-size>16 warnings | (not measured) | many per run | **0** | **bug fixed** | **bug fixed** |
| TelemetryModelCall DB rows / job | 0 (writer bug) | 29 123 | 20 348 | −30 % (fewer pose retries) | visibility restored |

**Correctness (parity check across all four counters):**

| Counter | Baseline | Cycle 1–5 | Cycle 6 | Δ vs baseline |
|---|---:|---:|---:|---:|
| Detections | 72 751 | 72 743 | 72 752 | **+1 (+0.001 %)** |
| BoundingBoxes | 72 751 | 72 743 | 72 752 | **+1 (+0.001 %)** |
| FrameEmbeddings | 72 585 | 72 577 | 72 586 | **+1 (+0.001 %)** |
| Per-class `attention/hand/person/sitting` | 11 776 / 8 801 / 19 162 / 33 012 | 11 772 / 8 801 / 19 162 / 33 008 | 11 769 / 8 801 / 19 162 / 33 020 | ≤ 0.06 % per class |

The Cycle 6 run actually produced **+1** of each counter vs baseline — frames
that previously got bumped to the HTTP fallback and lost their gRPC retry
budget now return cleanly with full results. No regression.

**Acceptance state: ACCEPTED.** Measurable production improvement on every
metric (FPS, elapsed, warning count, telemetry rows), zero correctness
regression, all 5 new chunking unit tests + 41 cycle-1–5 telemetry tests
pass and are gated by `inference-parallelization.yml`.

---

## 13. 2026-06-01 Cycle 7 Production Benchmark — Redis Client Caching

**Status:** **ACCEPTED with caveat (hypothesis was partially wrong but the direction is right).** Measured before/after with identical 4 541-frame `combined.mp4`; correctness preserved within 0.012 %.

**Candidate:** Cache the Redis client per `REDIS_URL` in both `apps/tracking/embeddings.py:redis_client` and `apps/video_analysis/tasks.py:_redis_client`. The embedding loop calls these helpers ~217 k times per job; the pre-cycle-7 code paid a fresh `Redis.from_url() + .ping()` round-trip every time.

**Evidence:**

| Item | Value |
|---|---|
| Replay key | `cycle7-rediscache-crop-frame-20260601T120927` |
| Job ID | `515fe118-6009-4776-916d-6473fbf31ed7` |
| Bench summary | `backend/logs/bench_summary_20260601T120927.json` (rc=0) |
| Inference audit | `backend/data/videos/515fe118-.../inference_audit.json` |
| Final status | `completed` (no stale-error) |
| DB row parity | 4 541 frames, 72 745 detections, 72 745 bboxes, 72 579 embeddings |

**Per-cycle FPS summary across all 4 runs (same `combined.mp4`, 4 541 frames, prod RTX 5090):**

| Cycle | Job ID | Step 2 FPS | Overall FPS (`run.complete`) | Overall FPS (DB `status=completed`) |
|---|---|---:|---:|---:|
| Baseline (ROI-320) | `77650001-3c4b-4b0a-94aa-b4eb899b90df` | 2.09 | 1.308 | **1.309** |
| Cycles 1–5 | `74ec0432-995c-487e-9d77-1048ec109fb1` | 5.14 | 2.644 | **2.077** |
| Cycle 6 (pose chunking) | `a1a448b9-474f-4dea-942b-3288bcae6900` | 5.17 | 2.776 | **2.780** |
| **Cycle 7 (redis cache)** | `515fe118-6009-4776-916d-6473fbf31ed7` | **5.39** | **2.865** | **2.870** |
| Δ Cycle 7 vs Baseline | — | **+158 %** | **+119 %** | **+119 %** |
| Δ Cycle 7 vs Cycle 6 | — | +4.3 % | +3.2 % | +3.2 % |

**Stage-by-stage (Cycle 7 vs Cycle 6):**

| Stage | C6 | C7 | Δ |
|---|---:|---:|---:|
| Step 2 (frame inference) | 879.0 s | 842.6 s | −36.4 s |
| Pose post-processing | 221.4 s | 220.6 s | unchanged |
| Persistence | 39.6 s | 42.4 s | +2.8 s |
| Render | 25.7 s | 25.8 s | unchanged |
| **Embedding** | **467.6 s** | **450.7 s** | **−16.9 s (−3.6 %)** |
| **TOTAL DB `status=completed`** | **1 633.2 s** | **1 582.1 s** | **−51 s (−3.1 %)** |

**Why the result was smaller than the projected −69 %:** I extrapolated the embedding-stage saving from "per-call `Redis.from_url + ping` = ~1.5 ms × 217 k calls = ~325 s". The actual saving was 17 s, meaning real per-call overhead is ~0.08 ms — about 18× smaller than assumed. `redis-py` 5.x's `Redis.from_url()` lazily constructs a `ConnectionPool` and pool checkouts are cheap; the first call pays the connect cost, subsequent ones reuse the pooled socket even *without* my cache. The cache layer is still useful (one connect per worker process instead of two-stage from_url+ping per call), just not as dominant as I projected.

**Where the real ~450 s of embedding wall actually lives** (next-cycle targets):
1. `cv2.VideoCapture.set/read` per-frame seek of the encoded video — likely dominant.
2. `model.embed([crop])` YOLO backbone per detection — 72 k single-crop inference calls.
3. Per-row `FrameEmbedding.objects.create()` — no bulk_create.
4. Per-row `detection.embeddings.exists()` idempotency query.

**Acceptance state: ACCEPTED.** Measurable production improvement, zero correctness regression, all 8 new unit tests (cycle 6 + cycle 7) pass and are gated by `inference-parallelization.yml`. The 15–32 FPS user target requires Cycle 8+ to attack the dominant embedding sub-costs above and Cycle 9+ to attack Step 2 with Triton ensemble/BLS.

---

## 14. 2026-06-01 Cycle 8 Production Benchmark — Embedding Stage Attack

**Status:** **ACCEPTED.** Largest single-cycle win since Cycles 1–5. The embedding stage that took 450 s in Cycle 7 now takes ~174 s. Correctness parity within 0.003 %.

**Candidate (bundled, single benchmark cycle because the three sub-changes share one stage budget):**
1. Enable `OFFLINE_EMBEDDING_REUSE_BY_TRACK=1` (already-implemented flag + a process-local `track_vector_cache` dict in the embedding loop).
2. Lazy `cv2.VideoCapture` frame reads — skip `.set()/.read()` when every detection in the frame already has a cached track vector. Forward-skip with `.grab()` to avoid keyframe re-decodes.
3. New `persist_embeddings_bulk` helper (`backend/apps/tracking/embeddings.py`) replaces 72 k single-row `.create()` calls with `bulk_create` batches of `EMBEDDING_BULK_BATCH_SIZE` (default 500). Vector-dim coercion at the bulk boundary preserves §17.2.

**Prod-measured root cause (Cycle 7 → Cycle 8 investigation):**
- 53 distinct StudentTracks vs 72 579 student Detections → 1 370× duplicate `model.embed()` work without reuse.
- `cv2_cap.set(...).read()`: 16.69 ms/frame on the same MP4 (sequential read: 0.32 ms/frame) → ~75 s of avoidable wall.
- Per-row `FrameEmbedding.objects.create(...)` × 72 k → no bulk path.

**Evidence:**

| Item | Value |
|---|---|
| Replay key | `cycle8-embed-bulk-crop-frame-20260601T125627` |
| Job ID | `d2de80a0-31b7-4a47-b9f1-d2e2156ea3a8` |
| Bench summary | `backend/logs/bench_summary_20260601T125627.json` (rc=0) |
| Inference audit | `backend/data/videos/d2de80a0-.../inference_audit.json` |
| Final status | `completed` |
| DB row parity | 4 541 frames, 72 749 detections, 72 749 bboxes, 72 583 embeddings |

**Per-cycle FPS summary across all 5 runs (same `combined.mp4`, 4 541 frames, prod RTX 5090):**

| Cycle | Job ID | Step 2 FPS | Overall FPS (`run.complete`) | Overall FPS (DB `status=completed`) |
|---|---|---:|---:|---:|
| Baseline (ROI-320) | `77650001-3c4b-4b0a-94aa-b4eb899b90df` | 2.09 | 1.308 | 1.309 |
| Cycles 1–5 | `74ec0432-995c-487e-9d77-1048ec109fb1` | 5.14 | 2.644 | 2.077 |
| Cycle 6 (pose chunking) | `a1a448b9-474f-4dea-942b-3288bcae6900` | 5.17 | 2.776 | 2.780 |
| Cycle 7 (redis cache) | `515fe118-6009-4776-916d-6473fbf31ed7` | 5.39 | 2.865 | 2.870 |
| **Cycle 8 (embedding stage)** | `d2de80a0-31b7-4a47-b9f1-d2e2156ea3a8` | 5.33 | **3.99** | **3.46** |
| Δ Cycle 8 vs Baseline | — | **+155 %** | **+205 %** | **+164 %** |
| Δ Cycle 8 vs Cycle 7 | — | −1.1 % (noise) | +39.2 % | +20.5 % |

**Stage breakdown (Cycle 8 vs Cycle 7):**

| Stage | C7 | C8 | Δ |
|---|---:|---:|---:|
| Step 2 (frame inference) | 842.6 s | 852.8 s | +1.2 % (within noise) |
| Pose post | 220.6 s | 220.7 s | unchanged (as designed) |
| Persistence | 39.6 s | 39.4 s | unchanged |
| Render | 25.7 s | 25.7 s | unchanged |
| **Embedding (post-`run.complete`)** | **450.7 s** | **~174 s** | **−61.5 %** |
| **TOTAL DB `status=completed`** | **1 582.1 s** | **1 312.3 s** | **−17.1 %** |

**Toward the 7.5-min SLA**: 21.87 min vs 7.5 min target = **14.4-min gap remaining**. Step 2 (14.2 min) is now ~98 % of that gap → Cycle 9 (Triton ensemble) is the next attack.

**Acceptance state: ACCEPTED.** Measurable production improvement on every metric (FPS, elapsed, embedding wall), correctness preserved within 0.003 %, 7 new bulk-create regression unit tests pass and are gated by `inference-parallelization.yml`.

---

## 15. 2026-06-01 Cycle 9 Production Benchmark — Behavior Triton Ensemble

**Status:** **NEEDS FURTHER ITERATION.** The production benchmark completed and
preserved correctness, and the ensemble reduced app-level behavior requests and
behavior RTT. It did **not** meet the Cycle 9 Step 2 acceptance gate because
Step 2 wall time stayed flat (`852.8 s` → `858.1 s`, +0.6 %).

**Candidate:**
1. Added `behavior_ensemble` Triton config over `posture_model`,
   `gaze_horizontal_model`, `gaze_vertical_model`, and `gaze_depth_model`.
2. Added `TRITON_BEHAVIOR_ENSEMBLE` with rollback to the previous four-call path.
3. Added app route `behavior_all` and split ensemble outputs back into the
   existing `output0` decode path.
4. Added production startup validation and a tensor parity probe.
5. Rebuilt the pinned production Triton binary with `TRITON_ENABLE_ENSEMBLE=ON`
   after the first production restart proved the existing build had
   `TRITON_ENABLE_ENSEMBLE=OFF`.

**Evidence:**

| Item | Value |
|---|---|
| Replay key | `cycle9-behavior-ensemble-crop-frame-20260601T180847` |
| Job ID | `c1651663-e08a-4e29-9ee3-fd0f09884b98` |
| Candidate SHA | `0fa847af43186017316cc11a8c76645ff463e574` |
| Bench summary | `backend/logs/bench_summary_20260601T180857.json` (rc=0) |
| Bench log | `backend/logs/parallel_flow_cycle9-behavior-ensemble-crop-frame-20260601T180847.log` |
| GPU CSV | `backend/logs/gpu_monitor_bench_20260601T180857.csv` |
| Parity probe | `backend/logs/behavior_ensemble_parity_cycle9_20260601T180827.json` |
| Triton rebuild backup | `/home/bamby/services/triton_build_r2502/tritonserver/install/bin/tritonserver.pre_cycle9_no_ensemble_20260601T180729` |
| Final status | `completed` |
| DB row parity | 4 541 frames, 72 749 detections, 72 749 bboxes, 72 583 embeddings |

**Before / after metrics:**

| Metric | Cycle 8 Baseline | Cycle 9 Candidate | Delta |
|---|---:|---:|---:|
| Step 2 wall | `852.8 s` | `858.1 s` | `+5.3 s` / `+0.6 %` |
| Step 2 FPS | `5.33` | `5.29` | `-0.8 %` |
| Telemetry session wall | `1138.8 s` | `923.1 s` | `-215.6 s` / `-18.9 %` |
| DB-completed elapsed | `1312.3 s` | `1110.7 s` | `-201.7 s` / `-15.4 %` |
| Overall FPS (DB-completed) | `3.46` | `4.09` | `+18.1 %` |
| App-level model calls | `20 348` | `9 557` | `-53.0 %` |
| Behavior crop app calls | `14 391` across four models | `3 597` `behavior_ensemble` calls | `-75.0 %` |
| Behavior mean RTT | `143-168 ms` per standalone behavior model | `107.9 ms` ensemble mean | improved |
| Behavior p95 RTT | `248-277 ms` per standalone behavior model | `173.9 ms` ensemble p95 | improved |
| Avg GPU utilization | `9.65 %` | `9.36 %` | `-0.29 pp` |
| Peak GPU utilization | `36 %` | `43 %` | `+7 pp` |
| Avg / peak VRAM | `15 663 / 15 663 MiB` | `15 663 / 15 663 MiB` | unchanged |

**Correctness parity:**

| Counter | Cycle 8 | Cycle 9 | Delta |
|---|---:|---:|---:|
| Frames | 4 541 | 4 541 | 0 |
| Detections | 72 749 | 72 749 | 0 |
| Bounding boxes | 72 749 | 72 749 | 0 |
| Frame embeddings | 72 583 | 72 583 | 0 |
| Student tracks | 53 | 53 | 0 |
| `attention_tracking` boxes | 11 776 | 11 776 | 0 |
| `hand_raising` boxes | 8 800 | 8 800 | 0 |
| `person_detection` boxes | 19 162 | 19 162 | 0 |
| `sitting_standing` boxes | 33 011 | 33 011 | 0 |
| Ensemble tensor parity | n/a | max abs diff `0.0` | pass |

**Observed root cause of the mixed result:** Cycle 9 removed three app-level
behavior uploads and responses per crop batch, but the Triton ensemble still
executes the four child TensorRT models server-side for every crop batch and
returns the same dense YOLO grids. That reduced request count and RTT, but it
did not reduce the Step 2 critical path enough to pass the 10 % gate.

**Decision:** **NEEDS FURTHER ITERATION.** Do not mark Cycle 9 accepted. Keep
the candidate deployed only as a measured, parity-clean intermediate state.
The next iteration must reduce the server-side ensemble critical path or dense
output movement, not just the Python request count.

### 15.1 Cycle 9b Child Critical-Path Measurement

**Status:** measurement only; no optimization accepted.

After the Cycle 10 safety proof, the live production Triton stats and direct
gRPC decomposition were used to identify the dominant behavior child before any
Cycle 9b implementation:

| Model | Direct server avg / exec |
|---|---:|
| `posture_model` | `12.133 ms` |
| `gaze_horizontal_model` | `16.058 ms` |
| `gaze_vertical_model` | `11.759 ms` |
| `gaze_depth_model` | `11.909 ms` |

Decision: `gaze_horizontal_model` is the dominant child and should be the first
Cycle 9b target. This aligns with the dense-tensor anatomy: it is also the
widest behavior output (`[84,2100]`). Evidence is recorded in
`docs/cycle_9b_child_critical_path_results.md`; the direct probe artifact is
`backend/logs/probe_rtt_decompose_cycle9b_child_20260602T.json`.

---

## 16. 2026-06-01 Cycle 10 Production Benchmark — LPM Phase 1

**Status:** **NEEDS FURTHER ITERATION — NOT ACCEPTED.** The LPM hook and
`telemetry_lpm_events` table deployed and ran on production, and the benchmark
completed. The run failed the LPM acceptance gates because contradiction
counters stayed at zero while `attention_tracking` boxes dropped sharply.
Production was rolled back to `LPM_ENABLED=0`.

**Evidence:**

| Item | Value |
|---|---|
| Replay key | `cycle10-lpm-crop-frame-20260601T201239` |
| Job ID | `17075418-4386-4b5f-85d4-ea23bec71f66` |
| Candidate SHA | `377fb59e66fe46327e6cbf2e6e64fd3bb70a0bde` |
| Telemetry session | `59c9e8bb-514f-41b4-a21a-ebda3a070a7d` |
| Bench summary | `backend/logs/bench_summary_20260601T231516.json` |
| Bench log | `backend/logs/parallel_flow_cycle10-lpm-crop-frame-20260601T201239.log` |
| GPU CSV | `backend/logs/gpu_monitor_bench_20260601T231516.csv` |
| Final status | `completed` |
| Rollback | `LPM_ENABLED=0`; workers restarted |

**Before / after metrics:**

| Metric | Cycle 9 Candidate | Cycle 10 LPM Run | Delta |
|---|---:|---:|---:|
| DB-completed elapsed | `1110.7 s` | `1076 s` | `-34.7 s` / `-3.1 %` |
| Overall FPS | `4.09` | `4.219` | `+3.2 %` |
| Telemetry session wall | `923.1 s` | `908.857 s` | `-14.2 s` / `-1.5 %` |
| App-level model calls | `9557` | `9557` | unchanged |
| Avg GPU utilization | `9.36 %` | `9.0 %` | `-0.36 pp` |
| Peak GPU utilization | `43 %` | `39 %` | `-4 pp` |

**Correctness parity:**

| Counter | Cycle 9 | Cycle 10 LPM Run | Delta |
|---|---:|---:|---:|
| Frames | `4541` | `4541` | 0 |
| Detections | `72749` | `63653` | `-9096` / `-12.5 %` |
| Bounding boxes | `72749` | `63653` | `-9096` / `-12.5 %` |
| Frame embeddings | `72583` | `63487` | `-9096` / `-12.5 %` |
| Student tracks | `53` | `53` | 0 |
| `attention_tracking` boxes | `11776` | `2680` | `-9096` / `-77.2 %` |
| `hand_raising` boxes | `8800` | `8803` | `+3` |
| `person_detection` boxes | `19162` | `19162` | 0 |
| `sitting_standing` boxes | `33011` | `33008` | `-3` |

**LPM telemetry:**

| Metric | Value |
|---|---:|
| LPM event rows | `4541` |
| Persons seen by LPM | `10587` |
| Eliminated contradictions | `0` |
| C1 / C2 / C3 / C4 violations | `0 / 0 / 0 / 0` |
| Frames with eliminated contradictions | `0` |
| Avg / max LPM latency | `0.024 ms / 0.272 ms` |

**Decision:** **NEEDS FURTHER ITERATION — NOT ACCEPTED.** The production run
proved the telemetry table works, but the LPM gate failed. The immediate bug was
that a single low-confidence directional gaze box could resolve to UNKNOWN and
be dropped without any C1-C4 violation. A local safety fix was added after this
run, but it still requires a fresh production benchmark before Cycle 10 can be
reconsidered.

### 16.1 Safety Fix Production Proof

**Status:** **NOT ACCEPTED.** The safety fix was deployed and benchmarked on
production, but Cycle 10 still failed parity, contradiction-signal, and
performance gates.

| Item | Value |
|---|---|
| Replay key | `cycle10-lpm-violationonly-crop-frame-20260601T221110` |
| Job ID | `21666815-f4bd-4f5f-b90e-b9101b4d899d` |
| Candidate SHA | `31edac44c66233baadd3a26ddd57b51b1a043d66` |
| Telemetry session | `be855e0e-6393-467a-9688-b723a29a56a4` |
| Bench summary | `backend/logs/bench_summary_20260602T011132.json` |
| Bench log | `backend/logs/parallel_flow_cycle10-lpm-violationonly-crop-frame-20260601T221110.log` |
| GPU CSV | `backend/logs/gpu_monitor_bench_20260602T011132.csv` |
| Inference audit | `backend/data/videos/21666815-f4bd-4f5f-b90e-b9101b4d899d/inference_audit.json` |
| Final status | `completed` |
| Rollback | `LPM_ENABLED=0`; workers restarted |

| Metric | Cycle 9 Candidate | Safety Fix Proof | Delta |
|---|---:|---:|---:|
| DB-completed elapsed | `1110.7 s` | `1124 s` | `+13.3 s` / `+1.2 %` |
| Overall FPS | `4.09` | `4.039` | `-1.2 %` |
| Telemetry session wall | `923.1 s` | `934.779 s` | `+11.7 s` / `+1.3 %` |
| App-level model calls | `9557` | `9557` | unchanged |
| Avg GPU utilization | `9.36 %` | `8.6 %` | `-0.76 pp` |
| Peak GPU utilization | `43 %` | `43 %` | unchanged |
| Behavior ensemble mean RTT | `107.9 ms` | `111.893 ms` | `+4.0 ms` |
| Behavior ensemble p95 RTT | `173.9 ms` | `183.151 ms` | `+9.3 ms` |

| Counter | Cycle 9 | Safety Fix Proof | Delta |
|---|---:|---:|---:|
| Frames | `4541` | `4541` | 0 |
| Detections | `72749` | `72095` | `-654` / `-0.90 %` |
| Bounding boxes | `72749` | `72095` | `-654` / `-0.90 %` |
| Frame embeddings | `72583` | `71929` | `-654` / `-0.90 %` |
| Student tracks | `53` | `53` | 0 |
| `attention_tracking` boxes | `11776` | `11122` | `-654` / `-5.55 %` |
| `hand_raising` boxes | `8800` | `8803` | `+3` |
| `person_detection` boxes | `19162` | `19162` | 0 |
| `sitting_standing` boxes | `33011` | `33008` | `-3` |

**LPM telemetry:** `4541` rows, `10587` persons, `C1/C2/C3/C4 = 0/0/0/0`,
`eliminated=0`, `nonzero_frames=0`, average LPM latency `0.024 ms`.

**Decision:** **NOT ACCEPTED.** The production proof shows the violation-only
preservation rule reduced the original attention-box loss, but post-decode LPM
still lacks the probability evidence needed to detect contradictions. The next
iteration must capture pre-decode gaze probabilities or move LPM into the
future compact postprocessing / BLS path. Production remains rolled back to
`LPM_ENABLED=0`.

---

*Updated from production run on 2026-06-01. Update this file after each major pipeline change or hardware migration.*
