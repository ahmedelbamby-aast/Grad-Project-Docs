# Fast-v1 Custom Model Loop for Scene Behavior Distillation

**Last updated:** 2026-05-11

## 1) Executive Summary

This rollout defines an **external Teacher→Student pipeline** where a self-hosted VLM generates pseudo-labels for scene-level behavior from your existing image corpus (about 3M images), and a fast YOLO classification student model is trained from those labels.

Only the **final trained student model** is integrated into this app.

For v1, the strategy is intentionally constrained to reduce complexity and accelerate delivery:

- **Single dominant label** per image/frame
- **Threshold-only** filtering on teacher confidence
- **Fixed small taxonomy** for behavior classes
- **Upload-first integration**, then live-stream rollout

---

## 2) Objectives and Non-Objectives

### 2.1 Objectives

- Produce a production-usable scene behavior student model that is much faster than the teacher VLM.
- Build a reproducible external loop for:
  - pseudo-label generation
  - dataset curation/versioning
  - training/evaluation/export
- Integrate the student model into the app inference stack with clear contracts and observability.
- Support future expansion toward complex/group behaviors and student-level fusion.

### 2.2 Non-Objectives (v1)

- No multi-label training in v1.
- No soft-target distillation in v1.
- No dynamic/evolving class ontology in v1.
- No in-app teacher labeling execution in v1.

---

## 3) Final v1 Decisions (Locked)

- Strategy: **Fastest v1**
- Teacher: **Self-hosted VLM stack**
- Labeling task: **Scene-level dominant behavior classification**
- Dataset basis: Existing image corpus (~3M images), with old detection labels removed/ignored
- Human QA: **Targeted QA** (low-confidence and disagreement buckets)
- Rollout: **Upload path first**, then live path

---

## 4) External Teacher→Student Pipeline Architecture

## 4.1 Stage A: Data Ingestion and Cleanup

- Input: image corpus from current storage sources.
- Remove/ignore all old object-detection labels for this pipeline.
- Build immutable manifest:
  - `image_id`
  - `source_path`
  - `source_video/session` (if available)
  - `capture_ts` (if available)
  - checksum/hash

Deliverables:
- `raw_manifest.parquet` (or equivalent)
- audit report for invalid/corrupt images

## 4.2 Stage B: Teacher Inference (Self-Hosted VLM)

- Run scene-level inference on each image.
- Enforce strict teacher output contract:
  - `image_id`
  - `dominant_label`
  - `confidence`
  - `teacher_model_name`
  - `teacher_model_version`
  - `prompt_version`
  - `inference_timestamp`
- Persist raw teacher responses for traceability.

Deliverables:
- `teacher_predictions_raw.jsonl`
- `teacher_predictions_normalized.parquet`

## 4.3 Stage C: Curation and Quality Filtering

- Apply confidence threshold policy:
  - retain only samples above configured threshold
  - drop below-threshold samples
- Apply class-balance pass:
  - cap heavy classes where needed
  - increase representation for minority classes
- Build curation report:
  - kept/dropped counts
  - per-class distributions
  - confidence histograms

Deliverables:
- `curated_dataset_manifest.parquet`
- `curation_report.md`

## 4.4 Stage D: Dataset Builder (Ultralytics Classification Format)

- Generate folder-based classification dataset:
  - `train/<class_name>/*.jpg`
  - `val/<class_name>/*.jpg`
  - `test/<class_name>/*.jpg`
- Keep teacher confidence + metadata in sidecar files (not in class-folder labels):
  - `metadata/train.jsonl`
  - `metadata/val.jsonl`
  - `metadata/test.jsonl`
- Enforce leakage prevention across splits by source grouping keys.

Deliverables:
- `dataset_vX/` in Ultralytics classify structure
- split integrity validation report

## 4.5 Stage E: Student Training

- Train YOLO classification model (`*-cls`) with fixed v1 class taxonomy.
- Track training artifacts:
  - config
  - seed
  - dataset version/hash
  - metrics
  - confusion matrix
- Choose best checkpoint by agreed validation criterion.

Deliverables:
- best model checkpoint
- training logs and metrics pack

## 4.6 Stage F: Evaluation and Promotion Gate

- Evaluate on held-out test split plus targeted human-reviewed subset.
- Gate for promotion:
  - minimum quality target met
  - acceptable class-wise behavior
  - no major taxonomy collapse

Deliverables:
- `evaluation_report_vX.md`
- promotion decision artifact

## 4.7 Stage G: Export and Runtime Packaging

- Export student model to required runtime formats:
  - ONNX
  - OpenVINO
  - TensorRT
- Reuse existing export/tooling patterns where possible.
- Register artifact metadata:
  - taxonomy version
  - teacher model/version used
  - confidence threshold used
  - dataset version
  - training run ID

Deliverables:
- runtime model artifacts + registry metadata

---

## 5) Integration Plan in Current App

## 5.1 New Inference Capability

- Add new task key for scene behavior, for example:
  - `scene_behavior_dominant`
- Extend model route/runtime config to support this task across backends.

## 5.2 Inference Output Contract

- Canonical response shape:
  - `label`
  - `confidence`
  - `top_k` (optional)
  - `model_name`
  - `model_version`
  - `runtime_backend`
  - `timestamp/frame_index`

## 5.3 Upload-First Rollout

- Execute scene-behavior inference in upload job path first.
- Persist behavior timeline per job/session:
  - frame index
  - behavior label
  - confidence
  - runtime metadata
- Expose results in existing analysis payloads and dashboards.

## 5.4 Live Rollout (Phase 2)

- Enable live path behind feature flag after upload stability proves acceptable.
- Publish live behavior updates through websocket channels.

---

## 6) Data Contracts, Governance, and Versioning

## 6.1 Teacher Prediction Contract (External)

Required fields:
- `image_id`
- `dominant_label`
- `confidence`
- `teacher_model_name`
- `teacher_model_version`
- `prompt_version`
- `timestamp`

## 6.2 Student Prediction Contract (Internal)

Required fields:
- `session_or_job_id`
- `frame_index` (or image index)
- `dominant_behavior_label`
- `confidence`
- `runtime_backend`
- `model_version`
- `taxonomy_version`

## 6.3 Taxonomy Governance

- Fixed small taxonomy for v1 (explicit class list locked before training).
- Every prediction/event must carry `taxonomy_version`.
- Taxonomy change implies new model version and migration note.

---

## 7) Quality, Testing, and Validation

## 7.1 External Loop Tests

- Dataset hygiene tests:
  - detection label remnants are excluded
  - image integrity and dedupe checks
- Teacher contract tests:
  - schema validation
  - confidence field integrity
- Split integrity tests:
  - no train/val/test leakage by grouping key
- Training reproducibility:
  - run determinism band with fixed seed/config
- Export compatibility:
  - ONNX/OpenVINO/TensorRT load + smoke inference

## 7.2 In-App Integration Tests

- Upload-path contract tests for persisted behavior timeline.
- API/websocket contract compatibility tests (if live enabled).
- Runtime fallback behavior tests.
- Performance checks: added scene model must not violate upload SLA.

## 7.3 Promotion Criteria for Live Enablement

- Offline quality threshold reached on held-out set.
- Stable error/timeout/fallback rates.
- Acceptable latency footprint for selected runtime backend.

---

## 8) Rollout Phases

### Phase 0: Prep
- Lock taxonomy v1
- Lock threshold and QA policy
- Prepare manifests and storage layout

### Phase 1: External Pipeline v1
- Generate pseudo-labels
- Curate + build dataset
- Train + evaluate + export student model

### Phase 2: Upload Integration
- Add model route and inference adapter
- Persist and surface dominant behavior timeline
- Run canary on upload jobs

### Phase 3: Live Integration (Flagged)
- Enable on selected sessions/cameras
- Monitor latency/error impact
- Expand rollout if stable

### Phase 4: Stabilize and Iterate
- Improve taxonomy and thresholds
- Prepare v2 features (weighted/soft targets, richer behavior classes, student-level fusion)

---

## 9) Risks and Mitigations

- **Teacher noise risk**
  - Mitigation: threshold filtering + targeted QA buckets
- **Class imbalance risk**
  - Mitigation: curation balancing and capped class sampling
- **Taxonomy instability risk**
  - Mitigation: fixed v1 taxonomy and strict versioning
- **Integration latency risk**
  - Mitigation: upload-first rollout and backend-specific benchmarking
- **Runtime mismatch risk**
  - Mitigation: export smoke tests across ONNX/OpenVINO/TensorRT before promotion

---

## 10) Future Extensions (Not v1)

- Multi-label behavior outputs
- Confidence-weighted/soft-target distillation
- Complex/group behavior ontology expansion
- Fusion of scene-level behavior with per-student pose/tracking signals
- Continuous retraining loop with drift monitoring

---

## 11) Assumptions and Defaults

- External Teacher→Student loop remains outside this app codebase.
- Only final student artifacts are integrated into the app.
- Existing image corpus is sufficient to start v1.
- Fixed small behavior taxonomy is acceptable for first release.
- Upload-first rollout is mandatory before live path enablement.

