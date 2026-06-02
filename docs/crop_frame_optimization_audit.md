# Crop-Frame Pipeline — Optimization Audit (Evidence-Based)

**Last updated:** 2026-06-01

> Auditor roles: Senior Systems Architect · Performance Engineer · Triton Expert · GPU Optimization Reviewer
> Date: 2026-05-31 · Method: every claim verified against actual code / Triton configs / telemetry.
> **Rule applied:** documentation without implementation = FAIL. Code evidence overrides docs.
> Environment of audit: Windows dev checkout (no live Triton/GPU). Claims requiring a running
> RTX 5090 are marked **PROD-VERIFY** and were *not* asserted as proven.

---

## A. Executive Summary

| # | Principle | Status | Evidence (file:line) |
|---|---|---|---|
| 1 | No PNG/JPEG encode in inference path | **PASS** | crop + full-frame tensors are raw ndarray→`tobytes()`/`tolist()` (`tasks.py:2013-2035`, `:2455-2466`); `imencode/imwrite/PIL` only in UI/preview/placeholder |
| 2 | Binary tensors enabled | **IMPLEMENTED — DEFAULT OFF** | `TRITON_BINARY_TENSORS="0"` (`base.py:380`); client path `triton_client.py:115,257,717`; JSON fallback `:750` |
| 3 | gRPC preferred over HTTP | **IMPLEMENTED — DEFAULT HTTP** | `grpc_aio` client `triton_client.py:594,642`; defaults `TRITON_PROTOCOL_PREFERENCE="http"`, `TRITON_GRPC_ENABLED="0"` (`base.py:344,348`) |
| 4 | Dynamic batching | **PASS (configs)** / **PROD-VERIFY (load)** | `person_detector`(bs8), `rtmpose_model`(bs16), legacy `posture_model`(bs32) all have `dynamic_batching`; but only 2 model dirs exist on disk in active `triton_repository_cuda12/` |
| 5 | Crop aggregation | **PASS** | all person-crops across the whole frame-batch collected then dispatched in ONE batched call per model — `_run_crop_behaviour_for_items` (`tasks.py:2104-2194`); no `for crop: infer(crop)` |
| 6 | CPU/GPU pipeline overlap | **IMPLEMENTED — DEFAULT OFF** | `TRITON_OFFLINE_PIPELINE_OVERLAP="0"` (`base.py:376`) |
| 7 | Concurrent secondary models | **IMPLEMENTED — DEFAULT OFF** | `TRITON_CONCURRENT_MODELS="0"` (`base.py:372`); `asyncio.gather` (`tasks.py:~1828`) |
| 8 | CPU↔GPU boundary analysis | **DELIVERED** | see §B.8 |
| 9 | Telemetry | **PASS** | `FrameMeta` decode/preprocess/inference/postprocess/per-model-ms + `ModelCallMeta.rtt_ms` + session `drop_rate`/`peak_gpu_memory_mb`/`observed_fps` (`telemetry/session.py`,`models.py`,`writer.py`) |
| 10 | Production-readiness binding class | **SERIALIZATION + NETWORK bound (host/CPU-bound)** | all perf flags default-off ⇒ HTTP+JSON path ⇒ consistent with plan-measured ~1% GPU util |

**Headline.** The optimization *machinery is implemented and largely correct*, and — contrary to an
earlier draft of this report — **`crop_frame` mode is fully built and selectable end-to-end**
(`pipeline_mode.py:9,45,76`; `models.py:73-77`; `serializers.py:107-113`;
`_resolve_upload_pipeline_executor` `tasks.py:191`; crop dispatch `tasks.py:2104-2194`). Crop
aggregation and the no-PNG rule **PASS**.

Two classes of issue remain:
1. **Activation gap (not a code defect):** the four biggest wins — binary tensors, gRPC, pipeline
   overlap, concurrent dispatch — are **all default-OFF**, so the *shipping default* path is
   serialization/network-bound, not GPU-bound. Turning them on is a **PROD-VERIFY** cutover, not a
   dev-box edit (constitution §1.8).
2. **One genuine performance finding in `crop_frame`** (§B.5b): crops are letterboxed back **up** to
   640×640 because the behaviour engines are fixed at `dims:[3,640,640]`, so `crop_frame` currently
   does *more* behaviour-model GPU work than `full_frame`, not less. (An earlier draft of this report
   also alleged a decode-semantics mismatch between modes — **that finding was retracted after code
   verification**: both modes use the same `_decode_yolo_output0` decoder; see §B.5c.)

---

## B. Detailed Findings

### B.1 No PNG/JPEG in inference path — PASS
Inference tensors are always raw float buffers:
- `_build_crop_payload` (`tasks.py:2013-2035`): `np.ascontiguousarray(input_tensor,…).tobytes()`
  (binary) or `input_tensor.reshape(-1).tolist()` (JSON). **No image codec.**
- Full-frame builder (`tasks.py:2455-2466`): identical pattern.

Encode sites exist but are all **non-inference** (correctly classified):

| file:line | function | purpose | verdict |
|---|---|---|---|
| `tasks.py:755` | `_annotated_rgb_to_data_url` | base64 PNG for WebSocket UI | allowed (UI) |
| `tasks.py:2429` | preview write (`cv2.imwrite`) | preview frame image | allowed (evidence) |
| `tasks.py:3021` | plotted frame (`cv2.imwrite`) | annotated overlay | allowed (evidence) |
| `tasks.py:1299` | `_placeholder_frame_bytes` (PIL) | 2×2 JPEG when FFmpeg missing | allowed (placeholder) |
| `cropper.py:46` | `FrameCropper.crop` (`cv2.imencode`) | PNG crop for **persistence** | allowed — **NOT** used by the Triton crop inference path |

**Important nuance:** `FrameCropper` (`pipeline_service.py:75`, `services.py:280`) is the *legacy/
persistence* cropper that PNG-encodes. The **Triton `crop_frame` inference path does NOT use
`FrameCropper`** — it slices `orig_frame[y1:y2, x1:x2]` directly (`tasks.py:2157`) and builds a raw
tensor. So inference never pays a PNG encode. ✓

### B.2 Binary tensors — IMPLEMENTED, DEFAULT OFF
Flag `base.py:380`; client honours it (`triton_client.py:115` init, `:257` `use_binary`, `:717`
binary header) with JSON fallback (`:750`). Correct + backward-compatible. Default-off is project
governance ("done; default off; validate on GPU host", AGENTS.md). **Not a defect — an un-activated
win.** Applies to **both** full-frame and crop payloads (both builders branch on the same flag).

### B.3 gRPC — IMPLEMENTED, DEFAULT HTTP
Async gRPC client `triton_client.py:594` (`grpc_aio.InferenceServerClient`), gRPC infer `:642`,
selected by `_should_try_grpc()` with automatic HTTP fallback on failure. Defaults to HTTP
(`base.py:344,348`). Making gRPC the production default is a **PROD-VERIFY** cutover (Phase 3 in the
plan explicitly "requires live-Triton validation"), not a missing-code defect.

### B.4 Dynamic batching
**(a) Configs present & correct** for every model that has a `config.pbtxt`:
- `triton_repository_cuda12/person_detector`: `max_batch_size:8`, `preferred_batch_size:[1,2,4,8]`, `max_queue_delay_microseconds:8000`, `instance_group count:1`.
- `triton_repository_cuda12/rtmpose_model`: `max_batch_size:16`, `preferred:[1,2,4,8,16]`, `8000us`.
- legacy `triton_repository/posture_model/configs/offline.pbtxt`: `max_batch_size:32`, `preferred:[4,8,16,32]`, `5000us` (gaze_* analogous).

**(b) PROD-VERIFY — model availability / silent-skip risk.** On disk the **active** repo
`triton_repository_cuda12/` contains **only** `person_detector/` and `rtmpose_model/`. The
posture/gaze models live in the **legacy** `triton_repository/` (dual live/offline pbtxt) and in
`probe_engines_disabled/`. Route table (`model_route_service.py:29-50`) maps
`posture_detection→posture_model`, `gaze_*→gaze_*_model`. **If those models are not loaded by the
active Triton profile on prod, behaviour responses fail and `_decode_*` returns early — silently
dropping posture/gaze** (constitution §11 No-Silent-Failure / §2.3 Model gate). **Action:** confirm
on prod which models are `READY`; if behaviour models are required, deploy them to
`triton_repository_cuda12/` (or fail-closed, don't skip).
*Note: only `person_detector`/`rtmpose_model` `config.pbtxt` are git-tracked by `.gitignore:155-160`;
the rest are runtime artifacts — hence "PROD-VERIFY", not "FAIL".*

### B.5 Crop aggregation — PASS, with a performance caveat
**(a) Aggregation is correct.** `_run_crop_behaviour_for_items` (`tasks.py:2104-2194`):
- collects **every (frame, person-box) crop across the whole batch** into `crop_payloads`;
- dispatches each behaviour model across **all** crops in **one** `_infer_task_batch(...)` call
  (`tasks.py:2160`) — the same batched path full-frame uses, so dynamic batching + binary + telemetry
  all apply;
- **no per-crop `infer()` loop.** ✓ This is exactly the required design.

**(b) FINDING — crops are letterboxed UP to 640×640, so per-model compute is NOT reduced.**
`_build_crop_payload` docstring + code (`tasks.py:2014,2023`): *"Preprocess a single person crop to
a 640x640 letterboxed FP32 tensor … Mirrors the full-frame preprocessing so behaviour models
receive the same input contract."* The posture/gaze TensorRT engines are built for
`dims:[3,640,640]`, so each crop is upscaled to 640×640.
Consequence: in `full_frame`, a behaviour model runs **once** per frame (1×640×640). In
`crop_frame`, the **same** model runs **N times** per frame (N persons × 640×640). **As implemented,
`crop_frame` does MORE behaviour-model GPU work than `full_frame`, not less.** The plan's premise
("smaller input = far less compute + activation VRAM") is **not realized** because the engines are
fixed at 640. Realizing it requires re-exporting the behaviour engines at a smaller `imgsz`
(an engine rebuild from the same weights — *not* retraining). Today the genuine `crop_frame`
benefits are (i) cleaner per-person signal/accuracy and (ii) `rtmpose`'s native 256×192 ROI path —
**not** throughput. **This must be measured on prod before any "crop_frame is faster" claim.**

### B.5c Decode-semantics consistency — PASS (earlier draft finding RETRACTED)
A draft of this audit alleged the crop path decoded the behaviour `output0` as a `1×1` top-1
classifier while full-frame decoded it spatially. **Code verification disproves that:**
- `full_frame`: `_decode_task_response` → `_decode_yolo_output0(output0, frame_width=item w,
  frame_height=item h, …)` (`tasks.py:1662`).
- `crop_frame`: `_run_crop_behaviour_for_items` → the **same** `_decode_yolo_output0(output0,
  frame_width=meta['width'], frame_height=meta['height'], input_size=640,
  letterbox_scale/pad=crop-space, …)` (`tasks.py:2207-2217`), then translates each box back to frame
  space by adding `origin_x/origin_y` (`:2226-2230`).
The symbol `_decode_crop_behaviour_response` and any `frame_width=1, frame_height=1` decode **do not
exist** in the codebase (verified by grep). Both modes interpret the engine identically as a YOLO
detector (`output0:[14,8400]` posture / `[84,8400]` gaze), so the decode is **consistent**. No action.

### B.6 Pipeline overlap — IMPLEMENTED, DEFAULT OFF
`TRITON_OFFLINE_PIPELINE_OVERLAP="0"` (`base.py:376`); producer/consumer overlap (git `7e366a83`).
When on, frame N+1 decode/crop/preprocess overlaps frame N GPU infer. Applies to crop mode too
(crop work runs in the same producer stage). Correct; default-off pending PROD-VERIFY.

### B.7 Concurrent secondary models — IMPLEMENTED, DEFAULT OFF
`TRITON_CONCURRENT_MODELS="0"` (`base.py:372`). When on & >1 task, `asyncio.gather` dispatches all
models in one loop ⇒ per-batch latency `max(RTT)` not `sum`, with sequential fallback on error
(`tasks.py:~1805-1846`). Correct.

### B.8 CPU↔GPU boundary data-flow (crop_frame)

```
disk mp4 ──cv2 decode(CPU)──▶ host frame (H×W×3 u8)            decode_ms
        person_detector path: letterbox 640(CPU) ─serialize─▶ HTTP/gRPC ─▶ Triton ─H2D▶ GPU ─D2H▶ boxes
                                                                              model_rtt_ms
   for each person box:  slice orig_frame[y1:y2,x1:x2] (CPU, free)
                         letterbox UP to 640×640 (CPU)                      preprocess_ms
                         serialize (JSON .tolist()  | binary tobytes)       ◄ JSON default = expensive
        ── ALL crops batched ──▶ HTTP/gRPC ─▶ Triton ─H2D▶ GPU(FP16) ─D2H▶ output0
   decode top-class per crop (CPU) ─▶ attach annotation                     postprocess_ms
```

| Transfer | src→dst | size | frequency | avoidable? |
|---|---|---|---|---|
| frame decode | disk→host | ~2.6 MB | per frame | no |
| **tensor serialize** | host→wire | 4.9 MB binary / ~25 MB JSON | per model-call **× N crops** | **JSON→binary (flag off)** |
| **transport** | host→Triton | as above | per call × N crops | **HTTP→gRPC (flag off)** |
| H2D | host→GPU | 4.9 MB | per call × N crops | reduce only via smaller-imgsz engines |
| D2H | GPU→host | output | per call × N crops | inherent |
| crop slice | host | view | per person | free ✓ |
| PNG encode | — | — | — | absent from inference ✓ |

**Hot spots (priority):** JSON `.tolist()` (×N crops ⇒ worse in crop mode) → enable binary;
HTTP RTT (×N) → enable gRPC; 640-letterboxed crops ⇒ smaller-imgsz engine rebuild for true savings.

### B.9 Telemetry — PASS
`telemetry/session.py`: `FrameMeta` (l.49) carries `decode_ms, preprocess_ms, inference_ms,
postprocess_ms` and per-model timings; `ModelCallMeta.rtt_ms` (l.92); `SessionSummary` has
`drop_rate` (l.178/321), `peak_gpu_memory_mb` + `mean_gpu_utilization_pct` (l.338-339),
`observed_fps`/`target_fps` (l.144-145). Persisted in `telemetry_*` tables + JSON
(`writer.py`). **All audited metrics are measurable.** Crop cost folds into `preprocess_ms`
(includes the 640 letterbox) — keep it there and assert `preprocess_ms < inference_ms`.

### B.10 Binding class — SERIALIZATION + NETWORK bound (host/CPU-bound) by default
With defaults (binary off, gRPC off, overlap off, concurrent off): CPU decode → CPU letterbox →
**JSON `.tolist()`** → **HTTP** localhost → Triton → JSON parse → per-row DB writes, frames largely
sequential per model. That is serialization-bound + RTT-bound, matching the plan's measured **~1%
GPU util**. **Not GPU-bound today.** In `crop_frame`, the per-call costs multiply by N persons, so
the default (JSON/HTTP) path is *even more* host-bound there — making binary+gRPC **more** important
for crop mode than for full-frame.

---

## C. Code Changes Applied

**None in this pass — by design and by governance.** Justification:

1. **No safe one-line fix exists for the real findings.**
   - "Enable binary/gRPC/overlap/concurrent" = flipping prod defaults, **prohibited without RTX-5090
     validation** (constitution §1.8 Runtime Determinism, §12 gates; AGENTS.md marks each "default
     off, awaiting prod validation"). Doing it from a non-GPU dev box would itself be an unvalidated
     runtime change.
   - "640-letterbox wastes compute" → fix = **re-export behaviour engines at smaller `imgsz`** +
     update `config.pbtxt` dims + rebuild TensorRT on prod (CUDA 12.8). Engine work, PROD-only.
   - "decode-semantics mismatch" → requires confirming each model's true output contract on a live
     Triton; editing a decoder blind risks corrupting results (constitution §9.1).
   - "behaviour models not in cuda12 repo" → deploy/READY-verify on prod, or fail-closed.
2. The mechanisms the audit asked to "implement if missing" are **already implemented** (binary,
   gRPC, dynamic batching, crop aggregation, overlap, concurrent dispatch, telemetry). Re-writing
   them would duplicate working code (constitution anti-duplication).

So this deliverable is the **verified report + scoped fix list**, not fabricated diffs.

### Scoped fixes (ready to apply in a clean session / on prod)
| Pri | Fix | Location | Risk | Gate |
|---|---|---|---|---|
| P0 | Confirm behaviour models `READY` in active `triton_repository_cuda12`; if required but absent, **fail-closed** instead of silent skip | prod Triton + `_decode_*` early-return sites | med | prod READY check + a unit test asserting fail-closed |
| P1 | For true crop speedup, re-export posture/gaze engines at smaller `imgsz`; set `config.pbtxt` dims accordingly; keep 640 fallback | `triton_repository_cuda12/*`, `_build_crop_payload` | med | benchmark crop vs full on prod |
| P2 | PROD-VERIFY then enable `TRITON_BINARY_TENSORS`+`TRITON_PROTOCOL_PREFERENCE=grpc`+overlap+concurrent; record telemetry deltas per mode | `backend/.env` on prod | low (flag-gated) | `prod_run_benchmark.sh` both modes |
| P3 | Confirm SPA upload form surfaces `crop_frame` (API/serializer already do) | frontend upload form | low | manual + e2e |

---

## D. Performance Impact Estimate
Baseline (defaults): ~0.5–5 fps offline, ~1% GPU util (plan-measured).

| Change | FPS | GPU util | Latency | CPU util | PCIe |
|---|---|---|---|---|---|
| Binary tensors ON | ↑↑ | ↑ | ↓/call | ↓↓ | ↓ (~5×) |
| gRPC ON | ↑ | ↑ | ↓ RTT | ↓ | ↓ |
| Overlap ON | ↑ | ↑ | hides decode+crop | = | = |
| Concurrent models ON | ↑ | ↑ | max(RTT) not Σ | = | = |
| crop_frame **as-is** (640 letterbox) | **↓ or ≈** for posture/gaze (N× calls), accuracy ↑ | ↑ (more work) | ↑ total | ↑ | ↑ |
| crop_frame **with small-imgsz engines** | ↑↑ | ↑↑ | ↓ | ↓ | ↓ |

Directional only; ground truth = telemetry + `tools/prod/prod_run_benchmark.sh` (constitution §7.3).
**No numeric throughput certified here** — none measured on the prod GPU in this audit.

---

## E. Final Verdict

### NEEDS PERFORMANCE WORK

- **Engineering is sound and more complete than expected:** binary tensors, gRPC, dynamic batching,
  **crop aggregation**, pipeline overlap, concurrent dispatch, and full telemetry are all
  implemented and correct; `crop_frame` is wired end-to-end; the inference path is PNG-free.
- **But the default path is host-bound (serialization + network), not GPU-bound** — the wins are all
  switched OFF pending RTX-5090 validation.
- **One real performance issue blocks a valid "crop-frame is optimized" claim:** crops are
  letterboxed back up to 640×640 (engines fixed at that input), so `crop_frame` currently does
  *more* behaviour-model GPU compute than `full_frame`, not less — plus a PROD-VERIFY silent-skip
  risk if the behaviour models aren't loaded in the active Triton repo. (The decode path is
  consistent across modes — the draft's mismatch claim was retracted, §B.5c.)

**Not "READY FOR PRODUCTION" as an optimized crop-frame system yet**, but it is **not** an
architecture-bottleneck dead-end either: the structure is right (modes wired end-to-end, crops
aggregated, PNG-free inference, full telemetry). What remains is (1) prod-validate + enable the
existing flags, (2) re-export behaviour engines at a smaller input for real crop savings, and
(3) verify behaviour-model availability / fail-closed. All are bounded, prod-gated tasks — hence
**NEEDS PERFORMANCE WORK**.

---

## Appendix — verification commands (reproducible)
```
grep -n PIPELINE_MODE_CHOICES backend/apps/video_analysis/models.py        # 73-77: incl crop_frame
grep -n "pipeline_mode" backend/apps/video_analysis/serializers.py          # 107-113 ChoiceField
grep -n "crop_frame\|_run_crop_behaviour_for_items\|_build_crop_payload" backend/apps/video_analysis/tasks.py
grep -n "TRITON_BINARY_TENSORS\|TRITON_PROTOCOL_PREFERENCE\|TRITON_GRPC_ENABLED\|TRITON_CONCURRENT_MODELS\|TRITON_OFFLINE_PIPELINE_OVERLAP" backend/config/settings/base.py
ls backend/models/triton_repository_cuda12/*/config.pbtxt                   # only person_detector, rtmpose
grep -n _DEFAULT_ROUTE_TABLE -A22 backend/apps/pipeline/services/model_route_service.py
grep -n "imencode\|imwrite\|PIL\|tolist\|tobytes" backend/apps/video_analysis/tasks.py
grep -n "class FrameMeta\|rtt_ms\|drop_rate\|peak_gpu_memory_mb\|observed_fps" backend/apps/telemetry/session.py
```
