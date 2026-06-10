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
  research comparison, never review-priority authority. A Class B model may
  graduate to a production-mandatory **signal/representation** role **only**
  through the evidence-gated promotion lifecycle (see *Promotion* below), and
  **never** into a behavioral decision authority.
- Every consumed model is captured in `XAIModelRouteSnapshot`
  (model/version/artifact-digest lineage). A registry entry **without** a route
  snapshot is not production-valid.
- Any **future trainable** anomaly model requires a **separate governed plan**
  establishing a legitimate dataset, target semantics, evaluation authority, and
  promotion contract (per doctrine §"Permitted Pattern Methods").

## Reproducible Acquisition Contract

Cycle 015 now carries a manifest-driven acquisition contract so the model packs
needed for reproducible local/dev setup are explicit and scriptable:

- manifest:
  `specs/015-xai-anomaly-score/pretrained-model-acquisition-manifest.json`
- download helpers:
  `scripts/models/download-xai-registry-models.py`,
  `scripts/models/download-xai-registry-models.sh`,
  `scripts/models/download-xai-registry-models.ps1`
- verifier:
  `scripts/ci/verify_pretrained_models.py`

The manifest declares:

- runtime pack archive names (`pt.zip`, `onnx.zip`, `openvino.zip`,
  `tensorrt.zip`);
- environment-variable names that provide source URIs and expected SHA-256
  digests for those packs; and
- a per-registry-model mapping from registry entry → required runtime packs →
  reproducibility artifact paths.

This keeps model acquisition reproducible without hardcoding secrets or private
artifact URLs into the repo. A registry change is incomplete unless the manifest
and verifier remain consistent with the registry tables.

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

## Class B (graph) — Student-Interaction Graph Models (`PROBE_ONLY`)

The **production** student-interaction graph (FR-035) is **deterministic** — a
rule-based, identity-gated consolidation of existing Class A relational signals
(SRVL geometry, `InteractionEdge`, gaze/`head_yaw` orientation, the COMBINED
head, scene segmentation). **No pretrained model is required for production.**
The models below are **frozen, `PROBE_ONLY` research candidates** for comparing a
*learned* graph representation against the deterministic graph — they consume the
`rtmpose` skeletons / the deterministic graph you already build, so they need
**no new labels**.

| Candidate | Idea | Why it fits here | Constraint |
|---|---|---|---|
| **ST-GCN family** (ST-GCN, 2s-AGCN, CTR-GCN, MS-G3D) | spatio-temporal graph conv over skeleton joints, extendable to multi-person | consumes existing `rtmpose` skeletons; learns relational dynamics | `PROBE_ONLY`, frozen-after-fit-on-assumed-normal |
| **GAT / GraphSAGE** | learned node/edge embeddings over the interaction graph | embeds the deterministic graph for hypothesis comparison | `PROBE_ONLY`, frozen, no production authority |
| **ARG / GroupFormer** | actor-relation / group-activity graphs | reference for multi-person interaction structure | `PROBE_ONLY` |
| **StrGNN / TADDY** | structural-temporal / transformer **dynamic-graph anomaly** | unsupervised low-likelihood edge/subgraph scoring on the temporal interaction graph | `PROBE_ONLY`, no accuracy/AUROC claim |

> All four rows are **`PROBE_ONLY` / `HYPOTHESIS_ONLY`**. They MUST NOT drive a
> production `review_priority_score` or `pattern_state`, MUST NOT be trained as a
> collusion/cheating target, and MUST NOT report anomaly/collusion accuracy. The
> production graph and its per-student signals stay deterministic.

## Promotion: From `PROBE_ONLY` To Production-Mandatory

A probe does not become a production requirement by opinion — it earns it with
**evidence, a benchmark, and computed metrics**. This mirrors how the industry
promotes models: a **gated offline check → shadow → canary → production** rollout
with a **champion/challenger** comparison, a **governed approver**, and
**automated rollback**; the **ML Test Score** four-area production-readiness
rubric; staged **model-registry** promotion (`dev → staging → pre-production →
production`) with role-based approval; **model cards** for documentation; and the
**NIST AI RMF** Measure/Manage functions. We adapt those gates to our
no-ground-truth constraint: a probe may graduate only into a governed
**signal/representation** role, judged on **serving quality** — never on
behavioral validity. Because we have no ground-truth labels, every promotable
candidate is **unsupervised or self-supervised** (skeleton normalizing-flow VAD,
one-class boundaries, dynamic-graph anomaly); promotion certifies its serving
quality as a signal, not any labeled accuracy.

### Promotion lifecycle (`promotion_status`)

| Stage | `promotion_status` | Runs in prod? | Outputs used? |
|---|---|---|---|
| Offline hypothesis | `PROBE_ONLY` / `HYPOTHESIS_ONLY` | No | No |
| Shadow — receives a copy of real inputs, logs outputs | `SHADOW` | Yes | No (discarded) |
| Canary — limited, reviewer-visible | `CANARY` | Yes | Advisory only |
| Production-required signal/representation | `MANDATORY` | Yes | Yes — as a governed signal |
| Retired / reverted | `ARCHIVED` / `ROLLED_BACK` | No | No |

### Promotion gates (all required; bounded, evidence-driven)

- **G1 — Doctrine cap.** The promotion target is a governed **signal/representation**,
  not a behavioral verdict. No anomaly/cheating/normality accuracy, AUROC, or
  AUPRC is claimed. A behavioral-decision promotion is out of scope and needs the
  separate governed ground-truth program.
- **G2 — Reproducibility & determinism.** Same inputs → same outputs; exact
  model/version/artifact digest captured in `XAIModelRouteSnapshot`; idempotent.
- **G3 — Serving SLOs (benchmark).** A native RTX 5090 stride-1 benchmark meets
  the declared latency/throughput/VRAM/CPU budgets; a minimum shadow duration is
  observed; the serving-output distribution is **equal or better** than the
  incumbent (champion/challenger).
- **G4 — Signal quality & stability.** Stable produced signal, explicit
  missingness, calibration where task-appropriate evidence exists, drift
  behavior, and — where the model feeds explanations — XAI fidelity/sanity.
- **G5 — Monitoring & rollback.** Live drift/alert wiring plus **proven automated
  rollback** to the prior state, with runtime reconciliation convergence.
- **G6 — Governance & documentation.** A model card (purpose, inputs, metrics,
  limits), a benchmark-ledger entry, an immutable evidence manifest, security and
  retention compliance, and **sign-off by a designated governed approver**
  (role-gated). No self-promotion.

> **Not overly strict.** G1–G6 are a **bounded minimum bar**, not an open-ended
> obstacle course. Once the recorded benchmark + computed serving metrics +
> rollback proof meet the bar and the approver signs off, the model is promoted
> from probe to a fixed production requirement. Every transition is recorded in a
> `ModelPromotionRecord`, is reversible, and is logged in the benchmark ledger.

### What promotion still may NOT do

- Turn a learned model into a behavioral/cheating/normality decision authority.
- Report anomaly/behavioral accuracy/AUROC, or relabel assumed-normal data as
  ground truth.
- Skip the benchmark, the computed metrics, the rollback proof, or the approver.

## Probe Fine-Tuning Lane (corpus adaptation of Class B COPIES)

During corpus ingestion the system MAY take **copies** of Class B probe models
and adapt them on the supported videos. The **original registry entry stays
frozen and untouched**; every adapted model is a NEW registry row with
`lineage = FINETUNED_COPY(parent_digest)`, its own artifact digest, and
`promotion_status = PROBE_ONLY` until it independently passes the G1-G6
promotion lifecycle above. The governed adaptation options:

| Option | Method | How it works here | Industry/literature basis |
|---|---|---|---|
| **A — Filtered pseudo-label self-training** | teacher-student / Noisy-Student-style | probe copy infers on the corpus; every inference is screened by the **accepted standard deterministic methods** (robust observed-pattern statistics, cross-model agreement, identity gating, temporal consistency, confidence/entropy gates — all `learned`/`configured` provenance) and each one is **accepted / refused / edited** (e.g. temporal smoothing) with a recorded decision; the copy is fine-tuned on the accepted/edited inference signals only | Noisy Student (arXiv:1911.04252); pseudo-label filtering for continual TTA (arXiv:2406.02609); source-free video DA from noisy labels (arXiv:2311.18572) |
| **B — Frozen vs fine-tuned champion/challenger** | side-by-side comparison | the original frozen probe and the Option-A copy run on held-out corpus videos; compared on serving metrics, signal stability, drift behavior, and **agreement with the deterministic accepted baseline** — recorded as champion/challenger evidence for the promotion lifecycle | champion/challenger + shadow evaluation (registry promotion practice above) |
| **C — Self-supervised continued pretraining (DAPT)** | label-free objective continuation | continue the probe's own self-supervised objective on the corpus (normalizing-flow likelihood fit for STG-NF/DA-Flow; masked reconstruction for VideoMAE-style) — **no pseudo-labels at all**; the safest adaptation | domain-adaptive pretraining; SSL video representation literature |
| **D — Test-time adaptation (ephemeral)** | TENT-style entropy minimization / BN-stat adaptation | adapts activation statistics during inference with filtered pseudo-labels; **never persisted** as new weights — a persisted variant must go through Option A + the promotion lifecycle | TENT (arXiv:2006.10726); CTTA filtering (arXiv:2406.02609) |
| **E — Cross-model distillation from governed signals** | knowledge distillation | the copy is trained to match governed deterministic fast-path signals or frozen-ensemble outputs (machine-generated targets with full lineage, never behavioral ground truth) | self-training + distillation practice (e.g. arXiv:2507.15709) |

### Binding rules for every option

- **Copies only.** The parent frozen model is never mutated; the copy is a new
  registry row + artifact digest + `XAIModelRouteSnapshot` lineage.
- Accepted/refused/edited inference decisions are **model-derived training
  signals**, never ground truth; the filter policy and every threshold carry
  `learned`/`configured` parameter provenance, and the full accept/refuse/edit
  ledger is persisted with the run.
- Reviewer feedback may only **exclude/quarantine** corpus windows from a
  fine-tune set; it never supplies labels or training targets (doctrine).
- Evaluation compares the fine-tuned copy against the **standard accepted
  baseline methods** (deterministic production path) and the frozen parent —
  serving metrics, signal stability, agreement, drift — with **no behavioral
  accuracy/AUROC claim**.
- A fine-tuned copy reaches `SHADOW`/`CANARY`/`MANDATORY` only through the same
  G1-G6 evidence-gated lifecycle; improvements are claimed as measured
  benchmark deltas, never as "better cheating detection".

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
5. The **general baseline** (population/context tiers, corpus-ingested) is **not a
   pretrained model** — it is the same deterministic robust-statistics machinery
   at population scale. Deriving bounds from it (quantile envelopes, robust scale)
   is statistical estimation, not anomaly training; such values carry `learned`
   provenance, while the remainder carry `configured` provenance from a
   fingerprinted `.env`. **No operational value is hardcoded.**

## Acceptance (consistent with doctrine)

- A model is registry-valid only with a captured `XAIModelRouteSnapshot` and a
  declared frozen status.
- Class B entries must carry `promotion_status = PROBE_ONLY` and emit
  `ground_truth_status = unavailable_for_anomaly_behavior`.
- No registry change may introduce a trainable anomaly/cheating/normality target
  or report label-based anomaly accuracy.
- The general baseline is deterministic statistics, not a registry model; any
  operational value it derives is `learned` provenance and every other value is
  `configured` from a fingerprinted `.env` — no hardcoding.
- A model reaches `MANDATORY` only with a complete `ModelPromotionRecord`
  (native benchmark, computed serving metrics, model card, governed-approver
  sign-off, and rollback proof); `PROBE_ONLY`/`SHADOW`/`CANARY` outputs never
  enter a production `review_priority_score` or `pattern_state`, and no promotion
  reports behavioral accuracy/AUROC.

---

*Sources for the skeleton-VAD probe candidates:*
[STG-NF (pose normalizing flows)](https://orhir.github.io/STG_NF/) ·
[DA-Flow (dual-attention skeleton VAD)](https://arxiv.org/abs/2406.02976) ·
[Contracting skeletal kinematics for human-related VAD](https://www.sciencedirect.com/science/article/pii/S0031320324005685)

*Sources for the interaction-graph probe candidates:*
[ST-GCN (spatio-temporal graph conv)](https://arxiv.org/abs/1801.07455) ·
[Actor Relation Graph (group activity)](https://arxiv.org/abs/1904.10117) ·
[StrGNN (structural temporal graph anomaly)](https://arxiv.org/abs/2005.07427) ·
[Deep Graph Anomaly Detection survey (TKDE 2025)](https://github.com/mala-lab/Awesome-Deep-Graph-Anomaly-Detection)

*Sources for the promotion gates (industry practice):*
[ML Test Score — production-readiness rubric (Breck et al., IEEE Big Data 2017)](https://research.google/pubs/pub46555/) ·
[MLflow Model Registry — staged promotion](https://mlflow.org/docs/latest/ml/model-registry) ·
[Champion/Challenger + shadow/canary rollout (DataRobot)](https://www.datarobot.com/blog/introducing-mlops-champion-challenger-models/) ·
[Model Cards for model reporting](https://arxiv.org/abs/1810.03993) ·
[NIST AI RMF (Measure/Manage)](https://www.nist.gov/itl/ai-risk-management-framework)
