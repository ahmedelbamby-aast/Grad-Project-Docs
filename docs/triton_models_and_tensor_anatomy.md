# Triton Models — Inventory, Roles, and Dense-Tensor Anatomy

**Last updated:** 2026-06-01

**Status:** Reference document. Captured from live Triton on the production
Linux RTX 5090 server on 2026-06-01. Re-run the `curl` commands at the end
to refresh; the byte-count math is structural and only changes when the
underlying engines are re-exported with new shapes.

This document answers two questions that come up repeatedly when reasoning
about the optimization cycles:

1. *How many models are loaded in Triton, and what is each one's role?*
2. *What are the "dense input/output tensors" the cycles keep referring to,
   and what do they actually contain?*

---

## 1. What is loaded in Triton right now

Live model index from prod (`POST http://127.0.0.1:39100/v2/repository/index`):

```
behavior_ensemble       v1   READY
gaze_depth_model        v1   READY
gaze_horizontal_model   v1   READY
gaze_vertical_model     v1   READY
person_detector         v1   READY
posture_model           v1   READY
rtmpose_model           v1   READY
```

That is **7 entries READY**, but only **6 are real TensorRT engines** —
the 7th (`behavior_ensemble`) is a server-side scheduling wrapper added by
Cycle 9, has no `.plan` file, allocates no GPU memory of its own, and
routes a single gRPC call through models 2–5 in parallel.

### 1.1 The 6 real models (each a distinct TensorRT engine with its own weights)

| # | Model | Platform | Input shape | Output | When it runs | What its output feeds |
|---|---|---|---|---|---|---|
| 1 | **`person_detector`** | `tensorrt_plan`, `max_batch=8` | `[N, 3, 640, 640]` full frame, FP32 | `[14, 8400]` YOLO grid (4 bbox + 10 class scores per anchor) | Every 5th frame (sparse detect cadence `OFFLINE_DETECT_EVERY_N_FRAMES=5`); reused on intermediate frames | Person bboxes → tracker (ByteTrack / BoT-SORT) → crops fed into the 4 behavior models and into `rtmpose_model` |
| 2 | **`posture_model`** | `tensorrt_plan`, `max_batch=32` | `[N, 3, 320, 320]` letterboxed **person crop**, FP32 | `[14, 2100]` YOLO grid | Every frame, once per detected person crop | Sitting / standing classification → `sitting_standing` Detection rows in DB |
| 3 | **`gaze_horizontal_model`** | `tensorrt_plan`, `max_batch=32` | `[N, 3, 320, 320]` person crop, FP32 | `[84, 2100]` YOLO grid (wider output channel) | Every frame, per detected person | Left / right gaze direction → `attention_tracking` Detection rows; **input to the C1–C4 constraints of the staged LPM layer (Cycle 10)** |
| 4 | **`gaze_vertical_model`** | `tensorrt_plan`, `max_batch=32` | `[N, 3, 320, 320]` person crop, FP32 | `[14, 2100]` YOLO grid | Every frame, per detected person | Up / down gaze direction → `attention_tracking` rows; **LPM scope** |
| 5 | **`gaze_depth_model`** | `tensorrt_plan`, `max_batch=32` | `[N, 3, 320, 320]` person crop, FP32 | `[14, 2100]` YOLO grid | Every frame, per detected person | Forward / backward gaze direction → `attention_tracking` rows; **LPM scope** |
| 6 | **`rtmpose_model`** | `tensorrt_plan`, `max_batch=16` | `[N, 3, 256, 192]` re-cropped, different aspect, FP32 | Two heads: `simcc_x` `[17, 384]` + `simcc_y` `[17, 512]` (17 COCO keypoints × bin scores) | Every frame, per detected person | 17 body keypoints → pose video render + the `head_yaw_deg` used as a **read-only** C4 input to LPM; never modified by LPM |

### 1.2 The 7th entry — `behavior_ensemble` (NOT a model, a scheduling layer)

| # | Entry | Platform | Role |
|---|---|---|---|
| 7 | **`behavior_ensemble`** | `ensemble` (no `.plan` file, no weights, no GPU memory of its own) | A Triton scheduling layer added by Cycle 9. One gRPC call to this entry fans the same `[N, 3, 320, 320]` input out to models 2–5 (posture + 3 gaze) in parallel and returns all four outputs in one response. Flag-gated by `TRITON_BEHAVIOR_ENSEMBLE` — **Cycle 9 did not meet its acceptance gate so the ensemble is STAGED but not on the SLA path.** See `docs/cycle_9_results.md`. |

### 1.3 How the per-frame data flow uses them

```
input frame
   │
   ▼
person_detector  ──► person bboxes ──► tracker (BoT-SORT) ──► per-person crops
                                                                  │
                                            ┌─────────────────────┼─────────────────────┐
                                            ▼                                           ▼
                              behavior_ensemble (4 sub-models                     rtmpose_model
                              or 4 direct calls when flag off)                    (256×192 crop)
                                            │                                           │
                              posture / gaze_h / gaze_v / gaze_d                  17 keypoints
                                            │                                           │
                                            ▼                                           ▼
                              Detection rows + bboxes                          pose render +
                              (sitting/standing, attention_*)              head_yaw → LPM C4 input
                                            │
                                            ▼
                              [Cycle 10 — staged, OFF in prod] LPM filter
                              (C1–C4 on the 3 gaze models only — never touches
                               person_detector, rtmpose, or sitting/standing)
                                            │
                                            ▼
                              fd.boxes → persistence → render → embedding
```

### 1.4 Honest notes

- **Only 6 of the 7 entries have weights or do GPU compute.** The
  `behavior_ensemble` entry has an empty `1/` version directory by design —
  Triton accepts that for ensemble configs.
- The 4 behavior models (`posture`, 3 × `gaze_*`) share input shape
  `[N, 3, 320, 320]`, which is what made them ensembleable in the first
  place. `person_detector` (640²) and `rtmpose_model` (256×192) deliberately
  do **not** join the ensemble because their tensors are different shapes.
- The LPM layer (Cycle 10) is **scope-restricted to the 3 gaze models only**
  by design (`docs/logical_path_matrix_spec.md` §2.1). It reads RTMPose
  keypoints for the head-yaw signal in constraint C4 but is forbidden from
  mutating any RTMPose / person_detector / posture output.
- Cumulative size on disk varies by precision (FP16 engines): person_detector
  ~10 MB, each behavior model ~5–10 MB, rtmpose ~25 MB. VRAM occupied at
  runtime: ~15.6 GiB per the prod `nvidia-smi` (most of which is workspace
  and cuDNN, not the engines themselves).

---

## 2. What "dense tensor" means in our context

The phrase keeps coming up because dense output transfer is the **central
inefficiency** in the post-Cycle-8 pipeline and the target of Cycle 9b
Option 1.

- **"Dense"** in our context means *every spatial anchor in the model's
  output grid carries a prediction, regardless of whether there's a real
  object there*. Most of the bytes are filtered out by NMS post-processing
  on the client side — they are paid for, transferred, parsed, and then
  discarded.
- **"Compact"** is what we *would like* to send instead — only the
  post-NMS survivors as small tuples like `(class_id, score, x1, y1, x2, y2)`.

---

## 3. Inputs — what we send TO Triton on every call

| Model | Shape (FP32) | Bytes per item | What each cell contains |
|---|---|---:|---|
| `person_detector` | `[N, 3, 640, 640]` | **4.69 MB / frame** | Full video frame, letterboxed to 640×640, gray-padded to keep aspect ratio, normalized to `[0, 1]` (raw uint8 / 255). 3 channels = R, G, B. |
| `posture_model`, `gaze_h`, `gaze_v`, `gaze_d` | `[N, 3, 320, 320]` | **1.17 MB / crop** | One person crop (slice of the original frame inside the person bbox), letterboxed to 320×320, same `[0, 1]` normalization. |
| `rtmpose_model` | `[N, 3, 256, 192]` | **576 KB / crop** | A *separately* re-cropped person region using rtmpose-specific aspect handling. Taller-than-wide because human bodies are vertical. |

### Per-frame input traffic with 17 person crops (the prod combined.mp4 average)

| Path | Calculation | Total |
|---|---|---:|
| Without ensemble (4 separate calls) | 17 crops × 1.17 MB × 4 models | **79.6 MB / frame** |
| With Cycle 9 ensemble (1 call) | 17 crops × 1.17 MB × 1 ensemble | **19.9 MB / frame** |
| rtmpose (separate) | 17 crops × 576 KB | 9.8 MB / frame |
| `person_detector` (every 5th frame, amortized) | 4.69 MB / 5 | ~0.94 MB / frame |

That is why Cycle 9 cut input bytes 4× — it eliminated the duplicate
uploads of the same crop tensor. (Cycle 9 still didn't reduce Step 2 wall;
see `docs/cycle_9_results.md` for the post-mortem.)

---

## 4. Outputs — and this is where "dense" matters

The four behavior models are YOLO detection heads. YOLO emits a fixed-size
grid:

```
shape = [ 4 + num_classes ,  num_anchors ]
         ^^^^^^^^^^^^^^^^^   ^^^^^^^^^^^^
         per-anchor channel  spatial anchors (every possible detection location)
```

For each anchor (= one possible detection location), the channel dimension
holds:

- 4 box coordinates (`cx, cy, w, h` in pixels)
- N class scores (probabilities for each class the model was trained on)

### 4.1 Where `num_anchors` comes from

`num_anchors` is determined by the input size and YOLO's 3-scale feature
pyramid (strides 8, 16, 32). For 320×320 input:

```
anchors = (320/8)² + (320/16)² + (320/32)²
        = 40²      +  20²      +  10²
        = 1600     +  400      +  100
        = 2100 anchors per crop
```

For 640×640 (`person_detector`) it's 80² + 40² + 20² = **8400 anchors**.

### 4.2 What each model's output tensor actually contains

| Model | Output shape | Channels = `4 + num_classes` | What the model was trained to detect | Bytes per crop |
|---|---|---|---|---:|
| `person_detector` | `[14, 8400]` | 4 box + **10 classes** (legacy multi-class YOLO; we only filter for the `person` class downstream) | persons in the full frame | **459 KB / call** |
| `posture_model` | `[14, 2100]` | 4 box + 10 classes (sitting / standing / various postures) | sitting vs. standing per cropped person | **114.8 KB / crop** |
| `gaze_horizontal_model` | `[84, 2100]` | 4 box + **80 classes** (trained on a wider taxonomy that includes left/right gaze states alongside other attributes; only the gaze classes are consumed) | left vs. right gaze | **689 KB / crop** ← largest |
| `gaze_vertical_model` | `[14, 2100]` | 4 box + 10 classes | up vs. down gaze | **114.8 KB / crop** |
| `gaze_depth_model` | `[14, 2100]` | 4 box + 10 classes | forward vs. backward gaze | **114.8 KB / crop** |
| `rtmpose_model` | 2 heads: `simcc_x [17, 384]` + `simcc_y [17, 512]` | 17 COCO keypoints × bin scores (SimCC representation, not YOLO) | 17 body keypoints per cropped person | **59.5 KB / crop** |

### 4.3 Why this is wasteful — concrete math for one frame, 17 person crops

| Output | Per crop | × 17 crops | Per frame |
|---|---:|---:|---:|
| posture | 114.8 KB | | 1.91 MB |
| gaze_horizontal | 689 KB | | **11.4 MB** |
| gaze_vertical | 114.8 KB | | 1.91 MB |
| gaze_depth | 114.8 KB | | 1.91 MB |
| **Total dense behavior output / frame** | | | **17.1 MB** |
| rtmpose (`simcc_x` + `simcc_y`) | 59.5 KB | | ~1 MB |
| person_detector (every 5th frame) | 459 KB | every 5th frame | ~92 KB amortized |
| **Total dense output / frame across all models** | | | **~18.2 MB** |

After NMS the typical real survivors per crop per behavior model are
**1 detection box** — sometimes zero. So ~5 boxes × 6 floats × 4 bytes ≈
**120 bytes** of actually-useful data per crop.

**~17.1 MB of dense behavior output → ~2 KB of post-NMS survivors. That is
~99.99 % discard rate per frame.**

That entire 17 MB / frame still has to:

1. cross PCIe back from GPU to CPU on the Triton side,
2. travel over the gRPC wire (localhost but real bytes),
3. be parsed by `result.as_numpy()` in Python,
4. be reshaped to `[14, 2100]` / `[84, 2100]`,
5. run through `_decode_yolo_output0` (anchor-by-anchor argmax + NMS) **per crop per model**,
6. only then collapse to ~5 detection boxes.

That entire pipeline is what Cycle 9b Option 1 (server-side compact
post-processing / BLS) eliminates — the NMS runs *inside* Triton and Python
only ever sees the ~2 KB compact result.

### 4.4 Why `gaze_horizontal_model` is the largest output

The 84-channel output is a leftover from how the original model was
exported — it was trained on a COCO-derived 80-class taxonomy and the gaze
direction was added as classes inside that taxonomy rather than as a
separate head. The pipeline only uses the gaze-related class scores; the
other ~76 channels are pure waste.

**Trimming `gaze_horizontal_model`'s output to just the gaze classes
(re-export with a 2-class head) would save 11.4 MB → ~0.27 MB per frame on
its own** — that is Cycle 9b Option 2 (output fusion) in compact form,
specifically for this one model.

---

## 5. What the dense outputs feed into (client-side Python work per frame)

`_decode_yolo_output0(...)` in `backend/apps/video_analysis/tasks.py` does
this per `[C, 2100]` tensor:

1. Split the 4 box channels (cx, cy, w, h) and the `C-4` class channels.
2. For each anchor, compute `score = max(class_probs)` and
   `cls_id = argmax(class_probs)`.
3. Keep only anchors with
   `score ≥ TRITON_OFFLINE_BEHAVIOR_SCORE_THRESHOLD` (default ~0.25).
4. Run NMS on the survivors.
5. Cap at `TRITON_YOLO_MAX_DECODE_CANDIDATES = 100` (set by Cycle 1–5).
6. Return `[(cls_id, score, [x1, y1, x2, y2]), …]` — typically 0–3 items
   per crop.

Steps 1–4 cost real Python CPU time per crop, multiplied by 17 crops × 4
behavior models = **68 dense-grid decodes per frame**.

---

## 6. One-glance visual

```
       what we send                          what we get back
┌─────────────────────────────┐    ┌──────────────────────────────────────┐
│  17 person crops            │    │  [14, 2100]   posture       (1.91 MB)│
│  packed into one tensor     │    │  [84, 2100]   gaze_horizontal(11.4 MB)│
│  [17, 3, 320, 320] FP32     │ ─► │  [14, 2100]   gaze_vertical (1.91 MB)│
│  ≈ 19.9 MB                  │    │  [14, 2100]   gaze_depth    (1.91 MB)│
└─────────────────────────────┘    │  TOTAL dense output ≈ 17.1 MB        │
                                   │                                      │
                                   │  After NMS: ~5 detections / frame    │
                                   │  ≈ 120 bytes of actually-useful data │
                                   │  Discard rate ≈ 99.99 %              │
                                   └──────────────────────────────────────┘
```

That discard rate is the single biggest remaining inefficiency. It's what
the constitution-driven *"Cycle 9b Option 1 (server-side compact
postprocessing) is the highest-impact continuation"* recommendation in
`docs/cycle_9_results.md` is pointing at.

---

## 7. Reproduce the numbers from prod

```bash
# 1. Live model index
ssh prod-grad 'curl -s -X POST http://127.0.0.1:39100/v2/repository/index | python3 -m json.tool'

# 2. Per-model shape extract
ssh prod-grad 'for m in person_detector posture_model gaze_horizontal_model \
                    gaze_vertical_model gaze_depth_model rtmpose_model behavior_ensemble; do
  echo === $m ===
  head -3 /home/bamby/grad_project/backend/models/triton_repository_cuda12/$m/config.pbtxt
  grep -E "^\s*(dims|data_type|max_batch_size)" \
       /home/bamby/grad_project/backend/models/triton_repository_cuda12/$m/config.pbtxt | head -8
  echo
done'
```

The byte-count math in §§ 3–4 is structural (it falls out of the input
shape and output shape directly) so it only changes when the underlying
engines are re-exported. The most likely future change is `gaze_horizontal_model`
shrinking from `[84, 2100]` to `[14, 2100]` (Cycle 9b Option 2) or
`[(class+4), 2100]` shrinking to a compact post-NMS tuple per crop
(Cycle 9b Option 1 / BLS Python backend / Cycle 10 LPM Phase 2 server-side
migration).
