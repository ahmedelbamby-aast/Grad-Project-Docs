# Chapter 2 (Literature Review) — Audit

**Last updated:** 2026-06-04
**Source PDF:** `C:\Users\Ahmed\Documents\grad projects docs\Chapter2_Literature_Review__.pdf`
**Audit method:** grep-verified against working tree at `9172549b`
**Severity:** wrong claims = **MEDIUM**, missing content = **MEDIUM**

## Wrong claims (contradicted by code AND by Chapter 1)

| # | Claim in PDF | Reality | Evidence |
|---|---|---|---|
| 1 | "The system recognizes **eight behavior classes** from visual input: looking right, looking left, looking forward, looking backward, looking up, looking down, sitting, and standing" (pages 3, 8, 9, 19) | Internally inconsistent with **Chapter 1's "ten classes"** (which adds `student` and `teacher`). The actual deployed model count is **five binary classifiers** — neither "eight" nor "ten" is the production reality. | `backend/config/settings/base.py:280-284` |
| 2 | "a classification model trained on a purpose-built annotated dataset" (page 19) — *singular* | Production deploys **five separate binary models**, each trained on its own pair (per Chapter 5), not "a classification model" singular. | five `.pt` files in `backend/models/{forward_backward,right_left,standing_sitting,student_teacher,up_down}/weights/` |

## Wrong reference-list entries (internal typos in the PDF itself)

| Issue | Where | Should be |
|---|---|---|
| Reference **#6** in §2.10 says **"SCP-Dataset"** | page 21 | Body text consistently uses **"SCB-Dataset"** (§2.7 title, pages 15-17). The arXiv link `arxiv.org/html/2304.02488v7` corresponds to **SCB-Dataset**. Reference-list typo. |
| Summary comparison table (page 20) — same row says **"SCP-Dataset"** | page 20 | Same SCB / SCP typo. |
| §2.10 Reference **#1** title: "...Suspicious Behavior of **the Student**" | page 21 | Body title uses plural **"of Students"** (§2.2 heading). Internal inconsistency. |
| Summary table date column shows **"Apr-23"** for Sumaiya | page 20 | Body says "Sumaiya et al. (2023)" without a more specific month. The "Apr-23" is unverified — not citable from the inline text. |
| Reference **#2** has **no author attribution** in the bibliography — only the title | page 21 | Every other ref has author names; this one omits "Sumaiya et al." |
| Reference **#5** title says "Cheating Detection in **Examination** Using ..." | page 21 | Body §2.6 title says "Cheating Detection in **Examinations** Using ..." (plural). |

## Missing

- No mention of how the **actual proposed system maps these 8 classes to 5 deployed binary models** (Chapter 5's pivot). Chapter 2 reads as if the system were a single 8-class classifier.
- No literature on Triton-based serving, TensorRT optimisation, or production GPU inference — which is **the dominant engineering work** in this repo.

## Correct claims

- "the system does not autonomously determine whether cheating has occurred; the final judgment remains with the human reviewer" (page 3) — consistent with the BSIL review-label design at `backend/apps/anomalies/bsil_review_labels.py`.

## Recommended fixes (helpers)

### Fix the reference list (§2.10)

```diff
- 1. Holi et al. (2025). Intelligent Offline Exam Monitoring System for Identifying Suspicious Behavior of the Student.
+ 1. Holi et al. (2025). Intelligent Offline Exam Monitoring System for Identifying Suspicious Behavior of Students.

- 2. Automated Invigilation System for Detection of Suspicious Activities During Examination.
+ 2. Sumaiya et al. (2023). Automated Invigilation System for Detection of Suspicious Activities During Examination.

- 5. Cheating Detection in Examination Using Improved YOLOv8 with Attention Mechanism.
+ 5. Zuo et al. (2024). Cheating Detection in Examinations Using Improved YOLOv8 with Attention Mechanism.

- 6. SCP-Dataset: A Dataset for Detecting Student and Teacher Classroom Behavior.
+ 6. Yang (2025). SCB-Dataset: A Dataset for Detecting Student and Teacher Classroom Behavior.
```

### Fix the summary table (page 20)

| Column | Current | Correct |
|---|---|---|
| Row 6, "Paper Title" | `SCP-Dataset: ...` | `SCB-Dataset: ...` |
| Row 2, "Date of Publish" | `Apr-23` | Replace with the actual publication date from the IRJET DOI, or drop the column for this row. |

### Add a paragraph on the deployed mapping

Add to §2.8 ("Closing Remarks") a sentence acknowledging the **architectural choice** the system actually makes — that the proposed eight behavioural states are realised as **five binary pair classifiers** rather than a single eight-class detector, which is the design pivot documented in Chapter 5.
