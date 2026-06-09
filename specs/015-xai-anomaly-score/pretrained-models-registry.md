# Pretrained Models Registry (Cycle 015)

**Date**: 2026-06-10
**Binding**: [no-ground-truth-doctrine.md](no-ground-truth-doctrine.md)
**Owner**: `apps.pipeline` (route capture) + `apps.anomalies` (consumption)

## Purpose

Catalog every **pretrained** model Cycle 015 may consume, with its governed role,
its **frozen** status, the signals it feeds, and its promotion constraint. This
registry exists because Cycle 015 builds an XAI + anomaly layer **on top of**
existing pretrained models and **does not train or fine-tune** any anomaly,
cheating, or normality target. The registry makes that boundary auditable.

## Governing Rules (binding)

- **All models are FROZEN.** Cycle 015 introduces no training/fine-tuning of any
  anomaly/cheating/normality target. Pretrained weights are used as-is.
- **Class A** models are existing **production signal sources**; they emit
  governed signals only (with source, unit, validity, missingness), never anomaly
  verdicts.
- **Class B** models are **frozen representation / one-class candidates** usable
  **only as `PROBE_ONLY` / `HYPOTHESIS_ONLY`**. They MUST NOT be promoted to a
  production behavioral decision, trained on assumed-normal data presented as
  truth, or used to fine-tune production behavior. They produce hypotheses for
  research comparison, never review-priority authority.
- Every consumed model is captured in `XAIModelRouteSnapshot`
  (model/version/artifact-digest lineage). A registry entry **without** a route
  snapshot is not production-valid.
- Any **future trainable** anomaly model requires a **separate governed plan**
  establishing a legitimate dataset, target semantics, evaluation authority, and
  promotion contract (per doctrine §"Permitted Pattern Methods").

## Class A — Production Signal Sources (frozen, deployed via Triton)

These are the real backbone of the anomaly layer: each turns pixels into a
governed, domain-robust **signal** consumed by the per-student observed-pattern
profiles.

| Model (Triton) | Frozen role | Signals it feeds | State |
|---|---|---|---|
| `person_detector` | detect/localize people | position, box geometry, motion/velocity/dwell, tracks | READY |
| `rtmpose_model` | 2D human pose / **skeleton** | pose kinematics, `head_yaw`, posture transitions, **skeleton sequences** | READY |
| `posture_model` | sitting/standing posture | posture state + transitions | READY |
| `gaze_horizontal_model` | horizontal gaze | gaze yaw, sustained-direction patterns | READY |
| `gaze_vertical_model` | vertical gaze | gaze pitch | READY |
| `gaze_depth_model` | gaze depth | gaze depth/target estimation | READY |
| `behavior_ensemble` | behavior semantics | behavior states, margins, entropy, reliability | READY |
| `osnet_ain_x1_0` | ReID embeddings | identity continuity, reconnect-gap quality | READY |
| `yoloe_scene_seg` | scene segmentation | scene-object distance/direction/overlap, seat/zone | READY |
| **COMBINED head (`yolo11m`)** *(new, Cycle-014 sibling)* | role + posture + gaze per person | `student/teacher`, `sitting/standing`, 6-way gaze — **domain-robust** per-student signals | training (prod) |

> The COMBINED head is documented in `docs/yolov11_combined_head/ARCHITECTURE.md`.
> Its gaze/posture outputs are *exactly* the structured, domain-invariant signals
> the deviation layer prefers over raw pixels.

## Class B — Frozen Representation / One-Class Candidates (`PROBE_ONLY`)

The strongest fit is **skeleton-based Video Anomaly Detection (SVAD)**, precisely
because `rtmpose_model` **already produces** the pose/skeleton sequences these
methods consume — so a probe needs **no new annotation** and **no behavioral
labels**, fully inside the doctrine.

| Candidate | Idea | Why it fits here | Constraint |
|---|---|---|---|
| **STG-NF** | lightweight **pose normalizing-flow** VAD | consumes `rtmpose` skeletons; very small; learns density of *normal* pose dynamics, scores low-likelihood as deviation | `PROBE_ONLY`, frozen-after-fit-on-assumed-normal |
| **DA-Flow** | dual-attention normalizing flow (recent SOTA skeleton VAD) | beats STG-NF/MoCoDAD/COSKAD/TrajREC with few params; same skeleton input | `PROBE_ONLY` |
| **Deep SVDD** | one-class deep boundary | already vetted in `research.md` | `PROBE_ONLY` |
| **PatchCore / PaDiM** | memory-bank / patch distribution | visual-anomaly references | `PROBE_ONLY` |
| **VideoMAE / V-JEPA** | self-supervised video representation | used **frozen** as a generic feature source only (never a verdict) | `PROBE_ONLY`, frozen, no fine-tune |

> "fit-on-assumed-normal" for a probe means: estimate a density/boundary from
> assumed-normal signal sequences **as a hypothesis**. Per the doctrine this can
> never be promoted, can never claim accuracy/AUROC, and can never mutate a
> production profile or decision.

## How this registry overcomes "no ground-truth"

1. **Class A is pure signal extraction** — frozen pretrained models turn video
   into governed signals; that is not labeling, so the doctrine permits it.
2. The **production decision** path stays the deterministic, per-student
   observed-pattern comparison (no learned anomaly model in the loop).
3. **Class B learned anomaly** is allowed only as a **frozen, `PROBE_ONLY`
   hypothesis** trained on *assumed-normal* signal sequences — never promoted,
   never an accuracy claim.
4. Because the best probes are **skeleton-based** and you already run
   `rtmpose_model`, you can stand up a credible unsupervised baseline **with the
   data you have and zero new labels**.

## Acceptance (consistent with doctrine)

- A model is registry-valid only with a captured `XAIModelRouteSnapshot` and a
  declared frozen status.
- Class B entries must carry `promotion_status = PROBE_ONLY` and emit
  `ground_truth_status = unavailable_for_anomaly_behavior`.
- No registry change may introduce a trainable anomaly/cheating/normality target
  or report label-based anomaly accuracy.

---

*Sources for the skeleton-VAD probe candidates:*
[STG-NF (pose normalizing flows)](https://orhir.github.io/STG_NF/) ·
[DA-Flow (dual-attention skeleton VAD)](https://arxiv.org/abs/2406.02976) ·
[Contracting skeletal kinematics for human-related VAD](https://www.sciencedirect.com/science/article/pii/S0031320324005685)
