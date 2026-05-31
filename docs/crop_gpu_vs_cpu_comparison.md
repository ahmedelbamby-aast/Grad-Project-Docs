# Crop-Frame Pipeline: GPU vs CPU Cropping — Comparison & Latency Strategy

> Scope: Phase 7 (`crop_frame` mode) of the
> [Inference Pipeline Parallelization Plan](inference_parallelization_plan.md).
> This document compares performing the **person-crop step** on the **CPU**
> (where it lives today) versus the **GPU**, and — because it is CPU-bound today —
> defines how to **measure and guarantee low latency / high throughput** across
> the CPU↔GPU boundary.
>
> Hardware authority (per `AGENTS.md` / constitution §1.7): native Linux, no
> Docker, RTX 5090 (32 GB), CUDA 12.8, Triton-only inference.

---

## 1. What "cropping" actually means in this pipeline

In `crop_frame` mode the per-frame work is:

1. **Detect** persons — `person_detector` runs on the full 640×640 frame (Triton, GPU).
2. **Crop** each person ROI from `frame[y1:y2, x1:x2]` using the detector boxes.
3. **Run secondary models** (posture, gaze×3, rtmpose) on the **small crops**
   instead of the full frame.

Step 2 is the subject of this document. Steps 1 and 3 are already GPU (Triton).

**Where it runs today:** CPU.
`apps/pipeline/cropper.py::FrameCropper.crop()` does:

```python
crop = frame[y1:y2, x1:x2]          # numpy slice on a host (CPU) ndarray
ok, encoded = cv2.imencode(".png", crop)   # CPU PNG encode
```

The frame is a host `numpy`/OpenCV array decoded by `cv2.VideoCapture` on the CPU,
so the crop is a host-memory view and the encode is a CPU codec call.

---

## 2. The real cost is the boundary, not the slice

A pixel slice (`frame[y1:y2, x1:x2]`) is essentially free on either device — it is
pointer arithmetic / a strided view. **The cost that matters is data movement and
re-encoding across the CPU↔GPU PCIe boundary**, plus per-call serialization to
Triton.

Per person, per frame, the current path pays:

| Cost | Where | Why it hurts |
|---|---|---|
| Full-frame decode → host RAM | CPU (`cv2.VideoCapture`) | unavoidable for file input |
| Box slice | CPU | cheap |
| **PNG/encode of each crop** | CPU | codec is expensive at scale (N persons × frames) |
| **Host→device upload of each crop** | PCIe | one H2D copy per crop per model |
| JSON serialization (if `TRITON_BINARY_TENSORS=0`) | CPU | `.tolist()` cost dominates |
| Triton network round-trip per model | localhost TCP | RTT × models × crops |

With ~10–30 persons/frame × 5 models, "cheap CPU slice" becomes **hundreds of
H2D copies + network calls per frame** — that is the latency wall, not the slice.

---

## 3. CPU cropping vs GPU cropping — head-to-head

| Dimension | **CPU crop (today: FrameCropper + cv2)** | **GPU crop (Triton BLS/ensemble, DALI, or CUDA shared-mem)** |
|---|---|---|
| Pixel-slice cost | Negligible | Negligible |
| Frame upload | Whole frame already on host; each crop copied H2D separately | Frame uploaded **once**; crops are device-tensor views — **0 extra H2D** |
| Re-encode (PNG/JPEG) | Per crop on CPU — significant at scale | None (raw device tensors stay on GPU) |
| PCIe traffic | N crops × H2D per model | 1 frame H2D total (or 0 with CUDA shared memory) |
| Network calls to Triton | 5–6 per crop (or batched) | Collapsible to **1 call/frame** via ensemble |
| Implementation effort | **Done today**; pure Python/OpenCV | High — needs a Triton **ensemble + Python BLS** crop step (Phase 7c) or DALI |
| Retraining required | No | No (same engines; only the graph changes) |
| Risk | Low | Medium (Triton-graph work, validated only on prod GPU) |
| Determinism / debuggability | High (host arrays inspectable) | Lower (device tensors, BLS scheduling) |
| VRAM | Flat (crops live in host RAM) | Slightly higher (frame + crops resident on GPU), but bounded by batch×instance |
| Best when | Few persons/frame, CPU headroom, simplicity | Many persons/frame, throughput-critical, GPU underutilized (our case: ~1% util) |

### Verdict for this project
- **Short term (now):** keep cropping on the **CPU** with `FrameCropper`. It is
  correct, already wired, needs no retraining, and is the safe default for
  `crop_frame` mode. The win of `crop_frame` over `full_frame` comes from feeding
  **small** crops to the secondary models (less GPU compute + activation VRAM),
  **not** from where the slice happens.
- **Long term (Phase 7c, optional):** move the crop onto the GPU via a **Triton
  ensemble + BLS** (`preprocess → detector → crop → {posture, gaze×3, rtmpose}`)
  so the frame is uploaded once, lives on the GPU, and 5–6 calls collapse into
  **one**. This is the durable path to GPU saturation — but it is an optimization
  *on top of* the CPU-crop mode, not a prerequisite.

---

## 4. Since cropping is on the CPU: how to guarantee low latency & high speed

The goal is to make the **CPU crop step + the CPU↔GPU handoff** so cheap and so
overlapped that the GPU never waits on the CPU. Five levers, all already present
or planned in the parallelization plan:

### 4.1 Move bytes, not pictures — kill the re-encode
- For the **inference path**, do **not** PNG-encode crops. Slice the ROI and hand
  the **raw `np.ndarray`** straight to the Triton preprocessing (letterbox/normalize
  → tensor). `cv2.imencode(".png", …)` is only needed for *persisting* a crop
  image for the UI/evidence, not for inference.
- Keep the PNG encode only on the persistence/render branch, and **offload it**
  (Phase 4 `OFFLINE_OFFLOAD_POST_STAGES`) so it never blocks inference.

### 4.2 Binary tensors instead of JSON  (`TRITON_BINARY_TENSORS=1`)
- JSON + `.tolist()` is the single biggest CPU tax per call. Binary tensors send
  the raw buffer. Already implemented (Phase 1a), default-off, JSON fallback.
- **Native to gRPC** — see 4.3.

### 4.3 gRPC transport  (`TRITON_PROTOCOL_PREFERENCE=grpc`)
- Lower per-call RTT than HTTP and binary by default. This is the main RTT-bound
  win (Phase 3). Confirm `mean_rtt_per_model_ms` drops sharply in telemetry.

### 4.4 Batch the crops — one big call, not N small ones
- All crops for a frame (and across frames) go through the **same**
  `_process_batch_items` / batch queue, so Triton **dynamic batching** coalesces
  them into large GPU batches (Phase 5, `preferred_batch_size: [8,16,32]`).
- This is the key reason `crop_frame` can be *faster* than `full_frame`: many
  small same-size crops batch beautifully; full frames do not.

### 4.5 Overlap CPU and GPU — never serialize the stages
- **Pipeline overlap** (`TRITON_OFFLINE_PIPELINE_OVERLAP=1`, Phase 2):
  a producer thread decodes + crops frame *N+1* on the CPU while the GPU infers
  frame *N*. The crop cost is then **hidden behind** GPU time.
- **Concurrent model dispatch** (`TRITON_CONCURRENT_MODELS=1`, Phase 1b):
  posture/gaze/pose for the crops fire in one `asyncio.gather`, so per-batch
  latency is `max(model RTT)` not `sum`.
- **Eliminate the copy entirely (advanced):** `--cuda-shared-memory` /
  CUDA pinned-memory pools (already configured in `prod_start_triton.sh` via
  `--pinned-memory-pool-byte-size`) make H2D copies cheap and recyclable; a full
  CUDA-shared-memory path removes the host→device copy for the frame.

---

## 5. How to *determine* (measure) the CPU↔GPU latency — not guess it

Per constitution §1.6 / §7.1, every claim must be probe-backed. The telemetry
layer (`backend/apps/telemetry/`) already records the right signals; use them to
prove the crop boundary is not the bottleneck.

### 5.1 Per-stage frame timing (already emitted)
`on_frame_inferred(...)` reports, per frame:
`decode_ms`, `preprocess_ms` (← crop + letterbox + tensor build),
`inference_ms`, `postprocess_ms`. **The crop cost lives in `preprocess_ms`.**

> **Acceptance rule:** for `crop_frame` to be healthy, `preprocess_ms` must stay
> **well below `inference_ms`** (CPU work hidden behind GPU). If
> `preprocess_ms ≳ inference_ms`, the CPU crop/encode/serialize path is the wall —
> apply 4.1–4.5.

### 5.2 Per-model RTT (already emitted)
`telemetry_model_calls.rtt_ms` per `model_name` isolates network+GPU time from
CPU crop time. Compare `mean_rtt_per_model_ms` HTTP+JSON vs gRPC+binary.

### 5.3 GPU utilization & VRAM (benchmark CSV)
`tools/prod/prod_run_benchmark.sh` records `utilization.gpu`, `memory.used`.
- If **GPU util is low while throughput is low** → CPU/boundary bound → the crop
  path is starving the GPU. Fix with overlap + batching + binary/gRPC.
- If **GPU util is high** → you are GPU-bound; cropping is no longer the issue.

### 5.4 The concrete measurement procedure (prod RTX 5090)
```bash
# 1. Baseline: full_frame mode
tools/prod/prod_run_benchmark.sh --pipeline-mode full_frame  --timeout-seconds 3600

# 2. Candidate: crop_frame mode (CPU crop), same clip
tools/prod/prod_run_benchmark.sh --pipeline-mode crop_frame  --timeout-seconds 3600

# 3. Compare via the API (per constitution: independent baseline vs candidate)
curl -s http://127.0.0.1:8011/api/v1/video-analysis/jobs/<id>/benchmark-comparison/ | python3 -m json.tool
```
Then read from `telemetry_sessions` / `telemetry_frames` / `telemetry_model_calls`:
- p50/p95 `preprocess_ms` (crop cost), `inference_ms`, total frame latency;
- `mean_rtt_per_model_ms` per model;
- `peak_gpu_memory_mb`, drop rate, fps.

### 5.5 Pass/fail gates for "low latency, high speed" (state in the job's plan)
| Metric | Source | Target |
|---|---|---|
| p95 `preprocess_ms` (crop) | telemetry_frames | < 30–50% of p95 `inference_ms` |
| GPU utilization | benchmark GPU CSV | trending up vs `full_frame`, not ~1% |
| fps (crop_frame vs full_frame) | telemetry_sessions | ≥ full_frame on a multi-person clip |
| `peak_gpu_memory_mb` | telemetry_sessions | ≤ full_frame (crops are smaller) |
| drop rate | telemetry_sessions | 0 (offline must not silently drop, §8.3) |

> The numbers above are *operational targets to be validated*, not constitutional
> constants — they must be confirmed on representative classroom footage on the
> prod GPU before `crop_frame` is accepted (constitution §3.5, §7.3).

---

## 6. Decision summary

- **Keep cropping on the CPU now.** It is correct, retraining-free, low-risk, and
  the benefit of `crop_frame` comes from *smaller GPU inputs*, not from GPU-side
  slicing.
- **Guarantee speed by hiding the CPU boundary**, not by moving the slice: raw
  ndarrays (no inference-path PNG), binary tensors, gRPC, crop batching, and
  CPU/GPU pipeline overlap — all already flag-gated in the plan.
- **Measure, don't assume:** `preprocess_ms` (crop) must stay under `inference_ms`,
  and GPU util must rise; both are already in telemetry and the benchmark runner.
- **Only escalate to GPU-side cropping (Triton ensemble + BLS, Phase 7c)** if, after
  4.1–4.5, telemetry still shows the CPU↔GPU boundary starving a multi-person GPU.
  That step uploads the frame once, keeps crops on-device, and collapses 5–6 calls
  into one — still **no retraining**.

---

## 7. References (code & plan)

- Crop step (CPU, today): `backend/apps/pipeline/cropper.py::FrameCropper`
- Full-frame Triton dispatch: `backend/apps/video_analysis/tasks.py::_run_triton_frame_level_inference`
- Shared batch dispatch (crops reuse this): `_process_batch_items`, `_infer_task_batch`
- Mode routing: `apps/tracking/pipeline_mode.py`, `_resolve_upload_pipeline_executor`
- Optimisation flags: `TRITON_BINARY_TENSORS`, `TRITON_CONCURRENT_MODELS`,
  `TRITON_OFFLINE_PIPELINE_OVERLAP`, `TRITON_PROTOCOL_PREFERENCE`,
  `OFFLINE_OFFLOAD_POST_STAGES`
- Telemetry: `backend/apps/telemetry/` (`telemetry_frames`, `telemetry_model_calls`, `telemetry_sessions`)
- Benchmark: `tools/prod/prod_run_benchmark.sh`, `docs/production_inference_benchmark.md`
- Plan: [`docs/inference_parallelization_plan.md`](inference_parallelization_plan.md) Phase 7 (7a CPU crop, 7c GPU ensemble)
