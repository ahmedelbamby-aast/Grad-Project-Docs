# Adding YOLOE + Depth Anything v2 — When, Not Whether

**Last updated:** 2026-06-01

**Status:** Decision recommendation, 2026-06-01. The two models are not yet in the repository or Triton repo; this document only addresses *when* to land them.
**Inputs to the decision:** current cycle-8 prod numbers, the projected cycle-9 → cycle-13 trajectory in `docs/cycles_9_to_12_implementation_playbook.md`, the SLA in `docs/runtime_sla_video_plus_5min.md`.

---

## TL;DR

**Defer both models until after the SLA target is met** (or until Cycle 13a's BLS architecture is shipped, whichever comes first). Adding them now would:

1. Make the optimization problem we are actively solving **measurably harder**.
2. Invalidate the per-stage numerical projections that drive the Cycle 9–12 plan.
3. Almost certainly need to be **redone** once Cycle 13a moves per-frame fan-out server-side.

The right insertion point is **after Cycle 12 lands ~5 FPS** and either:
- The SLA target is genuinely hit (so we have headroom to spend on new features), or
- Cycle 13a ships the BLS server-side graph (so YOLOE / Depth Anything v2 cost one extra server-side step rather than two more full gRPC round-trips).

---

## 1. What each model would cost the pipeline

Both are inference-heavy models. Concrete numbers below assume the prod RTX 5090 and the same `combined.mp4` benchmark (4 541 frames, ~17 persons/frame).

### 1.1 YOLOE (Ultralytics 2024 vision-language YOLO)

Open-vocabulary detector with text or visual prompts. Different from the current `person_detector` (which is a single-class fine-tune of a closed-vocab YOLO).

**If it replaces `person_detector`:**
- Input shape ≈ 640×640 — same as today.
- TensorRT compute on this GPU: probably **2–4× heavier** than the current `student_teacher.pt`-derived detector (the published YOLOE-S is ~10 M params vs the current detector at similar; YOLOE-L is ~80 M).
- Per-call server time: 4–8 ms today → **10–25 ms** with YOLOE-L.
- Detection cadence stays every-5th-frame, so amortized cost rises but it's not catastrophic.

**If it's added alongside `person_detector`:**
- Per detection-cadence frame: +1 full-frame gRPC call (~30–60 MB H2D transfer + 10–25 ms compute).
- Over 4 541 frames at every-5th cadence (908 frames): **~9–23 s** of new step-2 wall.

**If used for behavior/attribute prompts on cropped persons:**
- That's the typical use case. Per-crop dispatch like the existing 4 behavior models.
- 17 crops/frame × 4 541 frames = 77 k crop dispatches, server compute ~10–20 ms each batched: **150–300 s** of new step-2 wall.

### 1.2 Depth Anything v2 (Lihe Yang et al., 2024)

Monocular depth estimation. ViT-based encoder, dense regression head.

**Variants and typical compute on RTX 5090:**

| Variant | Params | Per-frame at 518×518 (typical input) | 4 541-frame total |
|---|---:|---:|---:|
| Small (ViT-S) | ~25 M | **~20–35 ms / frame** | **90–160 s** |
| Base (ViT-B) | ~97 M | ~45–70 ms / frame | 200–320 s |
| Large (ViT-L) | ~335 M | ~110–160 ms / frame | 500–725 s |

**Critical**: Depth Anything is **per-frame, not per-crop**. Even the small variant adds the equivalent of 1.5–2.7 minutes of new GPU work to a video whose total SLA budget is 7.5 min.

It also has a **different input shape** (typically 518×518) and outputs a dense `[H, W]` depth map — not something that can join the behavior ensemble in Cycle 9.

### 1.3 Combined impact on the SLA gap

We are currently 14.4 min over the 7.5-min SLA target after Cycle 8. The projected trajectory through Cycle 12 closes that to **+7.6 min over**:

| State | Total wall | FPS | Over SLA |
|---|---:|---:|---:|
| Cycle 8 (now) | 21.87 min | 3.46 | +14.4 min |
| + Cycle 12 (projected) | 15.17 min | 4.99 | +7.6 min |
| **+ YOLOE-L alongside** | +0.4 min | | +8.0 min |
| **+ Depth Anything v2-S per-frame** | +2.5 min | | +10.1 min |
| **+ both** | | | **+10.5 min** |

Adding the two models even with the smallest variants **moves the SLA target ~3 min further away** at exactly the moment we are trying to close the gap. The current FPS would regress from 3.46 to ~2.6–2.8.

---

## 2. Why "after the SLA" is the right ordering

### 2.1 The optimization cycles depend on a stable workload baseline

Every cycle in `docs/cycles_9_to_12_implementation_playbook.md` projects a numerical gain against the current 6-model workload. Cycle 9's "−18 to −27 % Step 2 wall" is a calculation against today's 4 behavior calls per frame. If we add YOLOE per-crop, Cycle 9 has to ensemble 5 models, the projected gain changes, and the regression unit tests have to cover the 5-model contract. We'd be re-measuring instead of optimizing.

### 2.2 The cycles 9–12 changes will not benefit the new models

- **Cycle 9** (Triton ensemble) — Depth Anything has a different input shape and **cannot** join the behavior ensemble. YOLOE *might* join if it's prompted on crops, but only if its input contract matches `[N, 3, 320, 320]` exactly.
- **Cycle 10** (pose parallelization) — irrelevant to either new model.
- **Cycle 11** (behavior input 320→256) — both new models keep their own input sizes.
- **Cycle 12** (parallel render + COPY persistence) — independent.

So the in-flight optimization work does not pre-emptively make YOLOE or Depth Anything cheaper.

### 2.3 Cycle 13a — the right insertion point

The architectural change in Cycle 13a (BLS server-side fan-out) moves crop preprocess + behavior ensemble + pose + embedding-vector compute behind **one gRPC call per frame**. Adding YOLOE and Depth Anything to that BLS graph costs **one extra server-side step each**, not two more full per-frame gRPC round-trips. That is roughly a 3–4× cheaper insertion point than today's architecture.

If we add them now: 2 weeks of integration work + accuracy validation + benchmark — most of which we'll **redo** when Cycle 13a re-routes the dispatch. If we add them after Cycle 13a: the same models, one config change in the BLS graph, far less risk.

### 2.4 The constitution discipline argument

Per the same discipline cycles 6–8 followed — one change per benchmark cycle, measured before/after, accepted only on evidence — adding two new heavyweight models is **not "one change."** It's two new models, two new accuracy gates, two new failure modes, all stacked on top of the optimization cycles we have not finished. The right size for each commit is the size that preserves clear before/after attribution.

---

## 3. When to add each — concrete checkpoints

| Checkpoint | What is true at that point | Add YOLOE? | Add Depth Anything v2? |
|---|---|---|---|
| After **Cycle 12** | ~5 FPS, +7.6 min over SLA | **No** — still over SLA | **No** — would push it ~3 min further over |
| After **Cycle 13a** (BLS server-side fan-out) | 8–10 FPS, near or at SLA | **Yes** if input shape can join the BLS graph (1-step add) | **Yes**, but pick the small variant first; verify it can run alongside the BLS graph on the same GPU |
| After **Cycle 13b** (multi-process video sharding) | 15–18 FPS, comfortably under SLA | **Yes** — there is headroom | **Yes** — pick variant based on observed remaining headroom |

The earliest sane checkpoint to start integration work is **right after Cycle 12 lands and Cycle 13a is in design**. That lets the cycle-13a design include the YOLOE / Depth Anything inputs from the start instead of grafting them on later.

---

## 4. What to do *now* if those features are roadmap-critical

If business priority says YOLOE / Depth Anything must land before the SLA work is finished, the honest options are:

1. **Pause the SLA work** at Cycle 8 and budget cycles 9–12 *after* the new-model integration. Total runtime will get worse before it gets better.
2. **Add them on a separate feature branch** without touching `master` or the optimization branch — keep the SLA work clean and re-base the model integration onto the post-SLA tree later.
3. **Add only the lightest variant of each** (YOLOE-S, Depth Anything v2 Small) and run them on a frame-stride > 1 (every 3rd or 5th frame) so the SLA gap doesn't widen as much.

Option 2 is the safest if the feature roadmap is firm — it parallelizes engineering effort without contaminating the benchmark baseline.

---

## 5. Open questions I cannot answer alone

- Which **YOLOE variant**? S / M / L? Open-vocab text prompts, or visual prompts? Used for what — replacing person_detector, attribute tagging on crops, scene classification, something else?
- Which **Depth Anything v2 variant**? S, B, L? Used for what — per-frame distance estimate, ROI-only depth on detected persons, depth-aware tracking, 3-D scene reconstruction?
- Are these **per-frame requirements** or **per-detected-person**? That alone is the difference between ~150 s and ~50 s of new wall time.
- Is the SLA target a **hard deadline** or a stretch goal? If hard, the answer is unambiguous (defer). If stretch, the option-2 branch-now-merge-later path is reasonable.

I'm putting these in writing so the decision happens with the trade-off visible, not buried.

---

## 6. Recommendation, restated

**Add YOLOE and Depth Anything v2 after Cycle 12 at the earliest, ideally after Cycle 13a.** Use the time between now and then to:

1. Decide which **variant** of each model is actually required (this changes the cost by 2–10×).
2. Decide where each output **feeds into** downstream — without a downstream consumer the model is dead weight.
3. Prepare the **accuracy parity gates** for both models in advance (the existing prod helper `tools/prod/prod_enable_roi_crop_behavior.sh` and the parity-probe pattern in `docs/cycles_9_to_12_implementation_playbook.md` §4 are reusable templates).

Adding them now would directly slow the SLA work and create technical debt that Cycle 13a will pay to undo. The cost of waiting 4 cycles (≈ 2–3 weeks of focused work) is small compared to the cost of re-doing the integration twice.
