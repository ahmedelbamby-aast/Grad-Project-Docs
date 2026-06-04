# Chapter 1 (Introduction) — Audit

**Last updated:** 2026-06-04
**Source PDF:** `C:\Users\Ahmed\Documents\grad projects docs\chapter 1.pdf`
**Audit method:** grep-verified against working tree at `9172549b`
**Severity:** wrong claims = **HIGH**, missing content = **HIGH**

## Wrong claims (contradicted by code)

| # | Claim in PDF | Reality in code | Evidence |
|---|---|---|---|
| 1 | "The proposed framework recognizes **ten distinct classes** — student, teacher, sitting, standing, looking left, looking right, looking upward, looking downward, looking forward, looking backward" (pages 2, 9, 10) | The system does **NOT** have a single ten-class detector. It is **five independent binary classifiers** plus a person/pose pipeline. The ten labels are spread across 5 separate models, never co-emitted as a single ten-class output. | `backend/config/settings/base.py:280-284`; model files: `forward_backward.pt`, `right_left.pt`, `standing_sitting.pt`, `student_teacher.pt`, `up_down.pt` |
| 2 | "Several YOLO architectures including **YOLOv8, YOLOv9, YOLOv10, YOLOv11, YOLOv12, and YOLO-NAS** were evaluated" (page 2) | Only **YOLOv8-style decode** is wired into production code paths. No YOLOv9 / v10 / v11 / v12 / YOLO-NAS / RF-DETR / Roboflow 3.0 reference appears in `backend/**/*.py` as a deployed runtime. (YOLO26 exists only as a planned integration for **pose**.) | grep `yolov9|yolov10|yolov11|yolov12` across `backend/**/*.py` = 0 hits. Only `# Decode YOLOv8-style output tensor` at `backend/apps/video_analysis/tasks.py:1571,1706`. |
| 3 | "Recognizes ten classes ... through **continuous frame-by-frame analysis**" as a single classifier output (page 2) | Production pipeline emits **per-task** outputs: `attention_tracking`, `hand_raising`, `person_detection`, `sitting_standing`, `pose_estimation` — not a unified 10-class taxonomy. | `backend/apps/video_analysis/tasks.py:3234, :5304` |
| 4 | "continuous frame-by-frame analysis" as the primary mechanism (page 2) | Production uses Triton ensemble routing (`behavior_ensemble`, `behavior_ensemble_gaze_slice`, `behavior_ensemble_gaze_slice_topk`), tracking with BoT-SORT/ByteTrack, and BSIL temporal sequence analysis. The pipeline is far more elaborate than "frame-by-frame analysis". | `backend/apps/pipeline/services/triton_client.py:43-49`; `backend/apps/behavior/` (BSIL); `backend/apps/tracking/` |

## Missing (real system components not mentioned)

- **Triton Inference Server**, **TensorRT engines** (`backend/models/triton_repository_cuda12/`), **server-side ensemble routing**, the **Top-K adapter chain** — none of this exists in Chapter 1's "framework" description.
- **BSIL (Behavioral Semantic Inference Layer)** — `backend/apps/behavior/` (10+ ORM tables, `episodes/`, `graph/`, `memory.py`, `temporal_*`) — entirely absent.
- **WHEP / WebRTC live streaming bridge** via go2rtc + mediamtx (`backend/config/settings/base.py:220-223`) — the chapter says "CCTV streams are processed continuously" but never specifies the actual transport.
- **Celery + Redis** queue architecture, `process_video_upload`, `run_live_stream_inference`.
- **`apps.anomalies`** (BSIL drift, review labels, baselines) — the alert path Chapter 1 promises is implemented here.
- **The pivot story**: Chapter 5 documents that the "ten-class" approach was tried, failed (20–22 % mAP@50), and was **replaced** by 5 binary pairs. Chapter 1 still describes the abandoned design as if it were the system.

## Correct claims (verified against code)

- Live + offline modes both supported — confirmed by Celery tasks `process_video_upload` (`backend/apps/video_analysis/tasks.py:3079`) and `run_live_stream_inference` (`tasks.py:4548`).
- Web-based dashboard — confirmed by `frontend/src/pages/DashboardPage.tsx` and 19 other lazy-loaded pages (`frontend/src/App.tsx:21-51`).
- Real-time alerts via WebSocket — confirmed: `backend/apps/video_analysis/consumers.py`, `backend/apps/anomalies/consumers.py`.

## Recommended fix (helper)

Re-write §1.1, §1.4, §1.5 around the **actual** detection topology:

> The framework deploys **five independent binary classifiers** (`student/teacher`, `sitting/standing`, `looking-left/right`, `looking-up/down`, `looking-forward/backward`) plus a **pose estimator** (RTMPose-M) and a **person detector**. At serving time, the classifiers and pose estimator are wrapped behind a single **NVIDIA Triton Inference Server** ensemble (`behavior_ensemble_gaze_slice_topk`), which applies a server-side slice + Top-K adapter chain to reduce per-frame output traffic by ~95 %. Per-frame detections feed a **Behavioral Semantic Inference Layer (BSIL)** that records temporal sequences, drift baselines, and review-label evidence — the system never autonomously concludes cheating; human reviewers issue the final judgment via the dashboard.

Architectures actually evaluated and shipped: YOLOv8-derived binary classifiers (per Chapter 5 benchmark campaign) + RTMPose-M (pose). Drop YOLOv9 / v10 / v11 / v12 / YOLO-NAS from the "deployed" framing — they were evaluated in Project I (Chapter 5) but did not ship to production.
