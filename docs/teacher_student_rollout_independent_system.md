# Teacher-Student Scene Behavior Rollout (Independent System)

**Last updated:** 2026-05-11

## 1) Executive Summary

This document defines a standalone **Teacher→Student training system** for scene behavior understanding:

- A **self-hosted VLM teacher** labels images/frames with a dominant scene behavior and confidence.
- A **YOLO classification student** is trained on curated teacher-labeled data to produce fast behavior inference.
- The system is designed as an independent data + model factory, with strict versioning, quality gates, and repeatable retraining cycles.

v1 constraints are intentionally simple:
- single dominant label per sample
- threshold-only filtering on teacher confidence
- fixed small class taxonomy
- targeted human QA only on risky buckets

---

## 2) Scope and Objectives

### 2.1 Objectives

- Build a production-grade standalone cycle for:
  - pseudo-label generation
  - dataset curation and versioning
  - student training and evaluation
  - model export and artifact registry
- Use the existing large image corpus (~3M images), while treating images as unlabeled for this task.
- Produce fast student models in ONNX/OpenVINO/TensorRT-ready form.
- Preserve traceability from every trained model back to teacher outputs and dataset versions.

### 2.2 Non-Objectives (v1)

- No multi-label behavior training.
- No soft-target distillation.
- No evolving taxonomy during a training run.
- No end-to-end video temporal reasoning in v1 (frame/image-level dominant behavior only).

---

## 3) Locked v1 Decisions

- Teacher: self-hosted VLM stack
- Label target: scene-level dominant behavior class
- Learning mode: pseudo-label supervised classification
- Confidence policy: threshold-only filtering
- QA mode: targeted QA (low-confidence/disagreement slices)
- Taxonomy: fixed small behavior class list (versioned)

---

## 4) System Architecture (Independent)

## 4.1 Core Subsystems

- **Data Catalog Service**
  - indexes raw media, computes checksums, tracks provenance.
- **Teacher Inference Worker Pool**
  - performs batched VLM labeling jobs.
- **Curation and Validation Engine**
  - applies confidence threshold, class-balance controls, and quality checks.
- **Dataset Builder**
  - emits Ultralytics classification dataset structure and sidecar metadata.
- **Training Orchestrator**
  - runs student training jobs with reproducible configs and seed control.
- **Evaluation and Gatekeeper**
  - validates model quality against hard thresholds and generates promotion decisions.
- **Export and Artifact Registry**
  - exports runtime formats and stores model lineage metadata.

## 4.2 Storage Domains

- `raw_media/` immutable source images
- `teacher_outputs/` raw and normalized pseudo-labels
- `curation/` filtered manifests and QA outcomes
- `datasets/` train/val/test packaged datasets
- `training_runs/` logs, checkpoints, metrics
- `exports/` ONNX/OpenVINO/TensorRT artifacts
- `registry/` model cards + lineage graph

---

## 5) End-to-End Pipeline Stages

## 5.1 Stage A: Data Ingestion and Cleanup

- Input all candidate images from source repositories.
- Ignore/remove legacy object-detection label files for this loop.
- Validate image readability, remove duplicates, record checksums.
- Build immutable manifest with:
  - `image_id`
  - `source_path`
  - `source_group_key` (video/session/source bucket)
  - `capture_ts` (if available)
  - `sha256`

Deliverables:
- `raw_manifest.parquet`
- `data_hygiene_report.md`

## 5.2 Stage B: Teacher Inference (VLM)

- Run teacher prompt for dominant scene behavior classification.
- Normalize each response to strict schema:
  - `image_id`
  - `dominant_label`
  - `confidence`
  - `teacher_model_name`
  - `teacher_model_version`
  - `prompt_version`
  - `inference_ts`
  - `raw_response_ref`
- Persist both raw response and normalized output for auditability.

Deliverables:
- `teacher_predictions_raw.jsonl`
- `teacher_predictions_normalized.parquet`

## 5.3 Stage C: Curation and Quality Filtering

- Apply global confidence threshold and drop low-confidence samples.
- Run class distribution checks:
  - maximum cap per class
  - minimum required samples per class
- Build candidate review set for targeted human QA:
  - near-threshold samples
  - random slice per class
  - disagreement slice (if secondary teacher available later)
- Finalize curated manifest.

Deliverables:
- `curated_manifest_vX.parquet`
- `curation_summary_vX.md`
- `targeted_qa_queue_vX.parquet`

## 5.4 Stage D: Dataset Packaging (Ultralytics Classification)

- Build folder layout:
  - `dataset_vX/train/<class_name>/*.jpg`
  - `dataset_vX/val/<class_name>/*.jpg`
  - `dataset_vX/test/<class_name>/*.jpg`
- Enforce split integrity by grouping key to prevent leakage.
- Persist sidecar metadata with teacher confidence and lineage:
  - `metadata/train.jsonl`
  - `metadata/val.jsonl`
  - `metadata/test.jsonl`

Deliverables:
- `dataset_vX/`
- `split_integrity_report_vX.md`

## 5.5 Stage E: Student Training

- Train YOLO classification (`*-cls`) on taxonomy version `TAXONOMY_vX`.
- Capture full reproducibility metadata:
  - code commit hash
  - training config
  - seed
  - dataset version hash
  - hardware/runtime info
- Save checkpoints and final candidate models.

Deliverables:
- checkpoints
- training logs
- metrics artifacts

## 5.6 Stage F: Evaluation and Promotion Gate

- Evaluate on:
  - held-out test split
  - targeted QA reviewed subset
- Required outputs:
  - class-wise precision/recall/F1
  - confusion matrix
  - calibration view for confidence reliability
- Promotion decision:
  - `promote` / `reject` / `retrain_with_adjustments`

Deliverables:
- `evaluation_report_vX.md`
- `promotion_decision_vX.json`

## 5.7 Stage G: Export and Artifact Registration

- Export promoted model to:
  - ONNX
  - OpenVINO
  - TensorRT
- Run export smoke inference checks.
- Register model card with full lineage:
  - model version
  - taxonomy version
  - teacher model/version
  - prompt version
  - confidence threshold
  - dataset version
  - training run ID

Deliverables:
- runtime artifacts
- model registry entries

---

## 6) Data Contracts and Schemas

## 6.1 Teacher Output Contract

Required fields:
- `image_id`
- `dominant_label`
- `confidence` (float in `[0,1]`)
- `teacher_model_name`
- `teacher_model_version`
- `prompt_version`
- `inference_ts`

## 6.2 Curated Sample Contract

Required fields:
- `image_id`
- `label`
- `confidence`
- `split` (`train|val|test`)
- `taxonomy_version`
- `dataset_version`

## 6.3 Student Prediction Contract (Offline Evaluation)

Required fields:
- `image_id`
- `predicted_label`
- `predicted_confidence`
- `top_k` (optional)
- `student_model_version`
- `runtime_backend`

---

## 7) Taxonomy Governance

- Maintain explicit taxonomy files per version:
  - class IDs
  - class names
  - class definitions
  - positive/negative examples
- Taxonomy is frozen per run.
- Any taxonomy change requires:
  - new taxonomy version
  - new dataset package
  - new training run lineage chain

---

## 8) Operational Model

## 8.1 Scheduling

- Batch teacher inference jobs by shard.
- Run curation and dataset packaging as deterministic post-processing jobs.
- Trigger training only when curation gates pass.

## 8.2 Retry and Failure Policy

- Teacher inference retries on transient failures with capped attempts.
- Failed samples move to quarantine manifest.
- Training job failures produce resumable checkpoints and incident logs.

## 8.3 Monitoring

- Track:
  - teacher throughput and failure rates
  - confidence distribution drift
  - class imbalance
  - training convergence and overfit indicators
  - export success rates

---

## 9) Quality Assurance and Testing

## 9.1 Data QA

- image integrity and dedup tests
- schema conformance tests
- split leakage tests
- class distribution thresholds

## 9.2 Model QA

- reproducibility checks with fixed seeds
- class-wise metric minimums
- confidence calibration checks
- robustness sampling on hard slices

## 9.3 Export QA

- ONNX/OpenVINO/TensorRT load tests
- inference parity smoke checks against reference outputs

---

## 10) Rollout Phases (Independent System)

### Phase 0: Foundation
- lock taxonomy v1
- define teacher prompt contract
- build manifests and storage conventions

### Phase 1: First End-to-End Cycle
- teacher labeling
- curation and QA
- dataset packaging
- student training + evaluation
- export + registration

### Phase 2: Stabilization
- tune threshold and curation policy
- improve class balance and hard-slice coverage
- harden reliability and observability

### Phase 3: Continuous Cycle
- scheduled incremental relabeling and retraining
- drift monitoring and periodic taxonomy review

---

## 11) Risks and Mitigations

- **Teacher label noise**
  - threshold filtering + targeted QA
- **Class imbalance**
  - curation caps/floors and controlled sampling
- **Data leakage across splits**
  - strict grouping-key split policy and automated checks
- **Taxonomy instability**
  - explicit taxonomy versioning and freeze-per-run
- **Export/runtime incompatibility**
  - mandatory smoke tests before artifact promotion

---

## 12) Future Extensions (Post-v1)

- multi-label behavior outputs
- confidence-weighted or soft-target distillation
- teacher ensemble consensus labeling
- temporal sequence-aware student models
- automated drift-triggered retraining

---

## 13) Assumptions and Defaults

- Existing image corpus is sufficient to bootstrap v1.
- Teacher outputs include usable confidence values.
- v1 dominant-label simplification is acceptable for first production cycle.
- Human review capacity exists for targeted QA buckets.
