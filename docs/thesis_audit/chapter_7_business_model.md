# Chapter 7 (Business Model & Market Entry Strategy) — Audit

**Last updated:** 2026-06-04
**Source PDF:** `C:\Users\Ahmed\Documents\grad projects docs\Chapter_7_Revised.pdf`
**Audit method:** grep-verified against working tree at `9172549b`
**Severity:** wrong claims = **HIGH**, missing content = **HIGH**

> This is the most-divergent-from-code of the four PDFs. The product **as
> marketed** in this chapter differs substantially from what the codebase
> actually implements.

## Wrong claims (contradicted by code)

| # | Claim in PDF | Reality | Evidence |
|---|---|---|---|
| 1 | "**Sentinel AI**, an AI-powered surveillance system" is the product name (§1, 2.1, 3, 4, 5.1, 8) | The repo identifies itself as **"Exam Monitoring Dashboard"** (`README.md:1`). "Sentinel" appears in code **only as a Python sentinel-object pattern** (`backend/apps/video_analysis/frame_pipeline.py:34,37` — `_DONE = object()` and `_DECODE_ERROR = object()`) and as a **negation** ("No Sentinel/Cluster needed" referring to **Redis Sentinel** — `specs/001-exam-monitor-dashboard/spec-addendum.md:844`). No file uses "Sentinel" as a product/brand name. |
| 2 | "five detection modules: (i) object detection, (ii) pose and gaze tracking, (iii) **facial recognition**, (iv) **audio monitoring**, (v) behavioural pattern analysis" (§3.1) | The repo implements (i), (ii), and (v). It does **NOT** implement (iii) facial recognition or (iv) audio monitoring. | grep `face_recognition\|facial_recognition\|face_embedding` returned 0 hits; grep `audio\|microphone\|speech\|voice\|whisper` in `backend/**/*.py` returned 0 hits. |
| 3 | "Integration compatibility with established LMS platforms, including **Moodle** and **Open edX**" (§3) | No Moodle integration. No Open edX integration. No LMS adapter code. | grep `Moodle\|edx\|LMS\|lms_integration` in `backend/**/*.py` returned 0 hits (the only `backend/uv.lock` occurrence is unrelated). |
| 4 | "**Edge-based processing** that reduces latency and minimises dependency on stable internet connectivity" + Phase 1 "prototype hardware assembly" (§3, §7) | The production runtime is **a single Linux RTX 5090 server** with native Triton, **not edge hardware**. No Jetson, Orin, Raspberry Pi, or edge-compute orchestration in the code. `AGENTS.md:995` explicitly states "Production runs do not depend on Docker or sudo" and targets a centralised GPU server. | grep `edge_comput\|jetson\|orin\|raspberry` in `backend/**/*.py` returned 0 hits. |
| 5 | "**On-premises data storage**" as a differentiator | True in a trivial sense (self-hosted Django + PostgreSQL + Redis stack), but the PDF implies it's a **deliberate product design choice for institutional governance**, whereas the codebase reality is that it's simply the development topology. There is no multi-tenant data-isolation layer. | No tenant isolation visible in `backend/apps/accounts/models.py` or the migration history. |
| 6 | "Real-time detection of … **unauthorised object use**" (§3) | The deployed person/posture/gaze models do **NOT** detect unauthorised objects (mobile phones, books, notes). That's a separate class taxonomy not present in any deployed model. | None of the five binary classifiers have phone/book/note labels — they only emit pair labels. |
| 7 | "**audio monitoring** for detecting verbal communication" (§3.1) | Not implemented. No audio pipeline, no microphone permissions handling, no speech model. | See #2 above. |
| 8 | "AI models optimised for **Egyptian classroom environments**" as a strength (§5.1) | No region-specific model training, dataset, or tuning visible in the repo. EDUNET dataset (cited in Chapter 5) is not Egypt-specific in the chapter's description either. | No Egypt-region tagged training data, no per-country model variants. |
| 9 | "Phase 3 ... additional planned features include **automated attendance tracking, student engagement analytics, and campus security monitoring**" | None of these features are in the current codebase. They are speculative. | grep `attendance\|engagement_analytics\|campus_security` returned no production app. |

## Missing (real system capabilities Chapter 7 doesn't mention)

- The **actual streaming transports supported**: RTSP, RTSPS, and WHEP/WebRTC bridged via `go2rtc` and `gst-mediamtx` (`backend/config/settings/base.py:220-223`, `backend/apps/cameras/services.py:16-22`).
- **Behavioral Semantic Inference Layer (BSIL)** — the actual behavioral-intelligence product surface, with 10+ ORM tables, ontology versioning, semantic-pose-state, temporal sequence records, episode lifecycle, interaction graphs (`backend/apps/behavior/`).
- **Anomaly review workflow** with explicit human-in-the-loop review labels (`backend/apps/anomalies/bsil_review_labels.py`) — this is the real "human reviewer keeps final judgment" feature.
- **Triton-based GPU inference** with Top-K adapters, server-side slicing, ensemble routing — the real engineering differentiator.
- **Audit logging** of every operator action (`backend/apps/audit/`).
- **Exam-room reference data and per-session lifecycle** (`backend/apps/exams/`, `backend/apps/sessions/`).

## Correct claims (verified)

- Web-based dashboard, Django backend, React frontend, real-time WebSocket updates — all confirmed (`README.md:5-7`).
- Hybrid Hardware-plus-SaaS is a business-model claim, not a code claim — not contradicted.

## Recommended fixes (helpers)

### Choose one product name

The current text alternates between "Sentinel AI" (Chapter 7) and "AI Surveillance System for Exam Behaviour Monitoring" (subtitle). The Chapter 5 PDF filename is `Chapter5_ApexGuard_final.pdf`, suggesting "ApexGuard" was an earlier candidate. **Pick one** and update Chapters 1, 2, 5, 7 to match.

### Remove the four false capability claims

Delete or amend the following in §3 and §3.1:

```diff
- (iii) facial recognition for identity verification
+ (iii) [REMOVED — not implemented; identity is currently inferred from student/teacher class]

- (iv) audio monitoring for detecting verbal communication
+ (iv) [REMOVED — not implemented; system is vision-only]

- Integration compatibility with established LMS platforms, including Moodle and Open edX.
+ [REMOVED — no LMS integration exists. If planned, move to §7 "Future Work".]

- Edge-based processing that reduces latency and minimises dependency on stable internet connectivity.
+ Centralised GPU-server architecture (single NVIDIA RTX 5090 host running NVIDIA Triton Inference
+ Server natively on Linux) with a low-latency local network to camera endpoints. No internet dependency
+ for inference; data residency is on-premises.
```

### Rewrite §3.1 around the actual detection modules

```text
The system incorporates four detection modules:
  (i) person detection (YOLOv8-derived `student_teacher` binary classifier)
  (ii) posture classification (`standing_sitting` binary classifier)
  (iii) gaze direction (three binary classifiers: `right_left`, `up_down`, `forward_backward`)
  (iv) pose keypoint estimation (RTMPose-M, 17 keypoints per detected person)
Detections are aggregated by the Behavioral Semantic Inference Layer (BSIL),
which tracks per-student temporal sequences and surfaces sustained or
repeated behaviours for human review.
```

### Add streaming transports to §3

The system supports RTSP, RTSPS, and WHEP/WebRTC ingestion via `go2rtc` and `gst-mediamtx` bridges — this is a genuine competitive feature against pure-browser proctoring solutions and should be highlighted in §3 (Value Proposition).
