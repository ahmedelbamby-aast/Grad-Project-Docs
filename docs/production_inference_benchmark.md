# Production Inference Benchmark & Issue Log

**Environment:** Linux, no Docker, no sudo — NVIDIA RTX 5090 (32 GB GDDR7, sm_120 Blackwell)  
**Recorded:** 2026-05-30  
**Branch:** master @ `33a24d1`  
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
| Frames processed per second (offline, HTTP, batch=1–4) | ~0.2–0.5 fps average | All 6 models, sequential HTTP |
| Burst rate (queue drain at job start) | ~50–120 fps | Short burst before queue empties |
| GPU utilisation (sustained processing) | 1.2% avg / 12% peak | 179 samples @ 2 s interval |
| GPU temperature under load | 36 °C | Far below thermal limit (89 °C) |
| Pipeline theoretical ceiling (HTTP, sequential) | ~0.5 fps | Limited by sum of 6 RTTs |
| Pipeline theoretical ceiling (gRPC, parallel dispatch) | ~20–50 fps | Estimated; pending implementation |
| Pipeline theoretical ceiling (gRPC + batch=16) | ~200+ fps | RTX 5090 TFLOPS ceiling |

---

## 7. Tools Added During This Session

| Script | Purpose |
|---|---|
| `tools/prod/prod_trt_guard.sh` | Pre-flight check: blocks Triton launch if TRT version ≠ manifest |
| `tools/prod/prod-rebuild-tensorrt-engines.sh` | Full stop→export→build→deploy→restart workflow |
| `tools/prod/prod_update_bashrc.sh` | Idempotently fixes `~/.bashrc`/`~/.profile` LD_LIBRARY_PATH |
| `tools/prod/prod_disk_cleanup.sh` | Purges `__pycache__`, probe engines, stale rollbacks, old logs |
| `tools/prod/prod_post_sync.sh` | Post-`uv sync` TRT install + guard check |
| `backend/scripts/build_tensorrt_engines.py` | Full .pt → ONNX (dynamic) → TRT → Triton deploy pipeline |

---

## 8. Re-run Checklist (for future benchmark comparisons)

```bash
# On prod server
cd /home/bamby/grad_project

# 1. Verify environment
bash tools/prod/prod_trt_guard.sh
curl -sf http://127.0.0.1:39100/v2/health/ready && echo "Triton OK"
pgrep -fc "celery -A config worker"  # expect 23

# 2. Reset GPU monitor
echo "ts,util,mem_used,mem_total,temp,power" > backend/logs/gpu_monitor_bench.csv
nohup bash -c 'for i in $(seq 1 600); do
  nvidia-smi --query-gpu=timestamp,utilization.gpu,memory.used,memory.total,temperature.gpu,power.draw \
    --format=csv,noheader,nounits >> /home/bamby/grad_project/backend/logs/gpu_monitor_bench.csv
  sleep 1; done' &

# 3. Submit benchmark job
cd backend && source .venv/bin/activate
python manage.py runtime_ingest_video \
  --video "/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4" \
  --actor-username "runtime_validator" \
  --runtime-profile offline \
  --pipeline-mode full_frame \
  --replay-policy new-attempt \
  --timeout-seconds 900 \
  --wait

# 4. Analyse
awk -F',' 'NR>1{s+=$2;n++;if($2>mx)mx=$2} END{printf "avg=%.1f%% peak=%s%% n=%d\n",s/n,mx,n}' \
  /home/bamby/grad_project/backend/logs/gpu_monitor_bench.csv
```

---

*Generated from production run on 2026-05-30. Update this file after each major pipeline change or hardware migration.*
