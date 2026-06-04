# Chapter 5 (Simulation & Performance Evaluation) — Audit

**Last updated:** 2026-06-04
**Source PDF:** `C:\Users\Ahmed\Documents\grad projects docs\Chapter5_ApexGuard_final.pdf`
**Audit method:** grep-verified against working tree at `9172549b`
**Severity:** wrong claims = **LOW–MEDIUM**, missing content = **HIGH**

> This chapter is **closest to the production reality** of all four PDFs.
> Most architectural claims match the deployed model files.

## Correct claims (verified)

| Claim | Evidence |
|---|---|
| **Five binary pairs** as the deployed architecture | `backend/config/settings/base.py:280-284` — exactly `student_teacher.pt`, `standing_sitting.pt`, `right_left.pt`, `forward_backward.pt`, `up_down.pt`. |
| Pair mapping (Forward/Backward = paper vs rotated; Left/Right = lateral; Up/Down = concealed-notes axis; Sitting/Standing = posture; Student/Teacher = identity) | Matches the env-key mapping at `backend/config/settings/base.py:283` (`forward_backward` → `DEPTH_GAZE`), `:282` (`right_left` → `HORIZONTAL_GAZE`), `:284` (`up_down` → `VERTICAL_GAZE`), `:281` (`standing_sitting` → `POSTURE`), `:280` (`student_teacher` → `PERSON_DETECTOR`). |
| Training is done **outside this repo** (Lightning AI); only trained weights are checked in | No training scripts present under `backend/` — only inference + deploy. |

## Wrong claims (contradicted by code)

| # | Claim | Reality | Evidence |
|---|---|---|---|
| 1 | "**YOLO26** ... the most recent addition to the YOLO family evaluated in this study" — used as a **behavior classifier** trained on all 5 pairs (§5.4) | YOLO26 in this repo exists **only as a planned integration for POSE estimation** (`YOLO26-Pose Medium`), not as a behavior classifier. There is **no YOLO26 classifier checkpoint** in `backend/models/{forward_backward, right_left, standing_sitting, student_teacher, up_down}/`. | `docs/yolo26_pose.md:3-12`; `Glob backend/models/**/*.pt` returns only YOLOv8-style classifier checkpoints. |
| 2 | "**EDUNET dataset**, providing 4,126 training images and 1,631 validation images" | The literal token `EDUNET` (case-insensitive) does **not appear in any tracked file** under `backend/`, `frontend/`, `scripts/`, or `tools/`. Dataset wiring is undocumented in production config. | `Grep EDUNET --type all` returned 0 hits in the repo. |
| 3 | "YOLO-NAS … excluded from further consideration" (§5.2.2) | Not contradicted, but unverifiable — no YOLO-NAS checkpoint or training artifact in the repo. Project I evaluation lives entirely outside the production codebase. | Filesystem inspection of `backend/models/` |
| 4 | "RF-DETR Medium ... Roboflow 3.0 ... YOLOv12 ... YOLOv11 ... YOLO-NAS" benchmark numbers (§5.2.2) | No artifacts, training logs, or inference benchmark JSONs for these architectures exist in `backend/logs/` or `backend/data/videos/`. Only the deployed YOLOv8-style classifiers are reachable from code. | Filesystem inspection of `backend/models/` and `backend/logs/` |

## Missing (real system components not mentioned)

- The **Triton ensemble layer** that wraps the 5 binary models at serving time (`behavior_ensemble`, `behavior_ensemble_gaze_slice`, `behavior_ensemble_gaze_slice_topk`) — **none of these are described**, yet they are the actual production inference path. (`backend/models/triton_repository_cuda12/` + `backend/apps/pipeline/services/triton_client.py:43-49`)
- The **Top-K adapter chain** (Cycle 9b B.2.c, currently the production-accepted route, job `be4ba9ee`) that prunes dense YOLO outputs to 100 anchors before serialisation.
- The **server-side slice adapter** (`gaze_horizontal_slice_model`, Cycle 9b B.2.b) that pruned the gaze-horizontal output `[84,2100] → [6,2100]`.
- The **OpenVINO and TensorRT exports** of each binary model — visible in `backend/models/{name}/weights/{name}_openvino_model/` and `{name}.engine` — never mentioned in the chapter.
- The **pose model** (`rtmpose-m.onnx`, `rtmpose-m.engine`) — Chapter 5 talks only about the 5 binary classifiers, but pose estimation is a sixth deployed model. (`backend/models/rtmpose/rtmpose_manifest.json`)

## Minor consistency issues inside Chapter 5

- §5.5.2 says "YOLOv9m … val/cls_loss collapse at epoch 125 invalidated the final weights" — unverifiable from the repo (no training logs).
- §5.5.5 reports "YOLOv11m and YOLOv11l both achieved 0.4334" as "strong evidence that performance is data-bounded" — claim is plausible, but no checkpoints exist to verify.
- §5.5.4 reports "YOLOv9m and YOLOv9c were not yet available at time of writing" for Left/Right yet §5.5.3 reports specific YOLOv9m/c numbers for Forward/Backward — internally consistent only if these runs completed at different times; flag for thesis review.

## Recommended fixes (helpers)

### YOLO26 framing

Either remove YOLO26 from §5.4's "eleven model variants" list, or split the sentence:

> The team collectively trained **ten** classifier variants spanning YOLOv8/9/10/11/12 (m + l sizes per family) on the five binary pairs. **YOLO26 was evaluated separately and only for pose estimation** (per [`docs/yolo26_pose.md`](../yolo26_pose.md)); its NMS-free inference architecture is a candidate for the future pose-runtime upgrade, not a sixth classifier.

### EDUNET dataset attribution

Either add a dataset card to `backend/data/datasets/edunet/` (with the SHA-256 and the source URL) so the citation is reproducible, or weaken the wording:

> All Project II models were trained on a **clean, manually curated 5,757-image subset** (4,126 train / 1,631 val) drawn from publicly available classroom-behaviour datasets after the merge-source quality failure in Project I.

### Add §5.8 — Production Serving Layer

Add a new short section after §5.7 covering what was actually shipped to production:

> The five trained classifiers are **not directly invoked by the application**. Instead, they are wrapped behind an **NVIDIA Triton ensemble** (`behavior_ensemble_gaze_slice_topk`, current production-accepted route per production benchmark job `be4ba9ee`) that applies a server-side slice adapter to the gaze-horizontal output and Top-K adapters to all four behaviour children, reducing per-frame behavior output traffic from ~6.85 MB to ~0.33 MB (a ~95 % reduction). This serving layer is documented in [`docs/entity/cycles/cycle_9b_b2c_topk.md`](../entity/cycles/cycle_9b_b2c_topk.md).
