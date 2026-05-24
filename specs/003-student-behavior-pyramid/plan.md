# Implementation Plan: Student Behavior Detection Pyramid

**Branch**: `003-student-behavior-pyramid` | **Date**: 2026-04-24 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/003-student-behavior-pyramid/spec.md`

## Summary

Build the multi-layered behavior detection pyramid that processes classroom video frames through five stages: (1) person detection via YOLOv8, (2) student/teacher classification and filtering, (3) per-student cropping from the original frame, (4) parallel binary behavior classification across four specialized YOLO models (posture, horizontal gaze, depth gaze, vertical gaze), and (5) result merging into unified per-student behavior records. The existing pipeline app (`backend/apps/pipeline/`) has stub layers that will be replaced with real ultralytics model inference while preserving the established `BasePyramidLayer` interface and downstream integrations (rule engine, tracker, WebSocket predictions, frontend `PredictionsPanel`).

Add a complete model lifecycle and benchmarking track (US5): scan `models/` for available artifacts (`.pt`, `.onnx`, TensorRT `.engine`/`.trt`, OpenVINO IR), detect runtime capabilities (OS, CPU, GPU, CUDA/TensorRT/OpenVINO availability), export missing formats from `.pt` sources via background Celery jobs, execute benchmarks per format-target combination in Dockerized containers, persist metrics (latency, throughput, memory/VRAM, stability, error rate), generate deployment matrices with ranked recommendations per model family, produce benchmark visualizations (comparison bars, trend lines, heatmaps), and generate explainability artifacts (attribution heatmaps) tied to inference samples. Use `Raw Data/` decompressed video folders to simulate camera-buffer streams for workload testing. Triton Inference Server is mandatory for production-path inference orchestration.

## Technical Context

**Language/Version**: Python 3.13 (backend), TypeScript ~5.9 (frontend)
**Primary Dependencies**: Django 5.1.5, Django Channels 4.2.2, DRF 3.15.2, Celery 5.4.0, ultralytics 8.3.61 (YOLOv8), onnxruntime-gpu 1.17, tensorrt 8.6 (optional), openvino 2024.0 (optional), Pydantic 2.10.6, React 19.2, Zustand 5, Vite 7.3, Daphne (ASGI)
**Storage**: PostgreSQL (via psycopg 3.2.5), Redis (Channels layer + Celery broker + benchmark result cache)
**Testing**: pytest, ruff, Docker (for export/benchmark isolation)
**Target Platform**: Windows + Linux (cross-platform, per constitution §VI); CPU (universal), NVIDIA GPU (CUDA 12.x, TensorRT optional), Intel CPU/GPU (OpenVINO optional)
**Project Type**: Web application (Django ASGI backend + React SPA frontend) with ML inference pipeline and model lifecycle management
**Performance Goals**: Full pyramid ≤3 seconds for 30 students per frame (SC-005); ≥15 FPS inference on single stream (constitution §VI); UI interactions ≤200ms; benchmark collection ≤5 minutes per format-target pair
**Constraints**: Pre-trained `.pt` model files provided externally (not trained in this feature); lazy model loading; GPU/CUDA when available with CPU fallback; Dockerized execution required for export/benchmark flows; Triton research/integration mandatory; Windows GPU support detection may be limited (fallback to CPU)
**Scale/Scope**: Classrooms of up to 30 students; 5 YOLO models (1 person detector + 4 behavior classifiers); each frame processed independently (no cross-frame tracking in scope); `Raw Data/` video folders for replay testing; `models/` directory scan with multi-format artifact support

### Model Lifecycle Requirements

**Discovery & Inventory**:
- Scan `models/` for `.pt`, `.onnx`, `.engine`, `.trt`, OpenVINO IR (`.xml`+`.bin`)
- Validate checksums, extract version tags from filenames
- Flag incomplete model families missing required formats

**Capability Detection**:
- Detect OS, CPU type, GPU presence, CUDA availability
- Detect TensorRT runtime availability (may fail on Windows even with NVIDIA GPU)
- Detect OpenVINO runtime availability
- Persist capability snapshot per host

**Export Orchestration**:
- Trigger background export jobs via Celery for missing formats
- Sequential export to minimize resource contention (one format at a time)
- Support `.pt` → ONNX → TensorRT chain; `.pt` → OpenVINO direct
- Log all steps and failure reasons; actionable error states

**Benchmark Execution**:
- Dockerized container per format-target combination
- Metrics: latency (p50, p95, p99), throughput (FPS), warmup time, memory/VRAM, error rate, stability score
- Use `Raw Data/` video replay to simulate camera-buffer streaming
- Background execution with status polling via WebSocket

**Deployment Matrix**:
- Score and rank format-target pairs per model family
- Selection criteria: latency, throughput, hardware compatibility, stability
- Persist recommendations with rationale

**Visualization & Explainability**:
- Generate comparison bar charts, trend lines, platform-format heatmaps
- Generate attribution heatmaps for model explainability
- Link artifacts to benchmark runs and model versions

**Triton Integration** (mandatory research):
- Research Triton Inference Server deployment patterns
- Model repository structure, config.pbtxt generation
- gRPC/HTTP inference endpoints
- Dynamic batching and model warm-up

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

> **Supreme Directive Gate**: ACKNOWLEDGED. Every modification will follow the 5-point checklist: (1) immediate git commit with Conventional Commits message, (2) create/update docs/ .md files, (3) include ALL applicable Mermaid diagram types, (4) write detailed diagram explanations, (5) cross-link all .md files. This gate is **PASS**.

> **Test-in-Loop Gate**: ACKNOWLEDGED. The Test-in-Loop methodology (constitution §II) will be enforced: ALL tests (unit, integration, system) will be written BEFORE implementation for every task. No implementation code will be written until corresponding tests exist and fail. This gate is **PASS**.

**Additional Constitution Gates**:

| Gate | Status | Notes |
|------|--------|-------|
| §I Code Quality (SOLID, DI, 30-line functions) | PASS | Pipeline uses Strategy pattern (BasePyramidLayer), Factory (layer registry), DI via constructor injection in PipelineService. All new classes follow SRP. |
| §II Test-in-Loop (unit+integ+system, 80% coverage) | PASS | Tests written first for each layer, detector, cropper, merger, and end-to-end frame processing. |
| §III UX Consistency (uniform patterns, ≤500ms updates) | PASS | Frontend already has PredictionsPanel, StudentCard, BoundingBoxCanvas — extend, don't rebuild. WebSocket prediction updates already wired. |
| §IV Security (no hardcoded secrets, PII redaction) | PASS | Model paths loaded from validated Pydantic config; no student PII in logs; tracking IDs are anonymized. |
| §V UI Design (3 themes, WCAG AA) | PASS | Existing theme system applies; new/modified components use themed CSS modules. |
| §VI Performance (≥15 FPS, GPU/CPU fallback, lazy loading) | PASS | Lazy model loading via singleton; concurrent layer execution via ThreadPoolExecutor; GPU with CPU fallback. |
| §VII Delivery Sign-Off (7-step protocol) | PASS | Full lint/type/test/security/spec-compliance scan before sign-off. |
| §VIII Think Before Coding (explicit assumptions) | PASS | All design decisions documented with rationale; no silent tradeoff selection. |
| §IX Simplicity First (minimum code) | PASS | No speculative abstractions; each component has single clear purpose. |
| §X Surgical Changes (minimal touch) | PASS | Changes scoped to pipeline/ only; no tangential refactoring. |
| §XI Goal-Driven Execution (verify criteria) | PASS | Success criteria defined per US (SC-001 to SC-013); loop until verified. |

## Project Structure

### Documentation (this feature)

```text
specs/003-student-behavior-pyramid/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
│   ├── rest-api.md
│   └── websocket-api.md
└── tasks.md             # Phase 2 output (via /speckit-tasks)
```

### Source Code (repository root)

```text
backend/
├── apps/
│   ├── pipeline/                  # Primary feature module (EXISTS — extend)
│   │   ├── __init__.py
│   │   ├── apps.py
│   │   ├── base.py                # BasePyramidLayer ABC (exists — extend with typing)
│   │   ├── models.py              # Django models (exists — currently empty)
│   │   ├── services.py            # PipelineService orchestrator (exists — rewrite)
│   │   ├── rule_engine.py         # RuleEngine anomaly evaluator (exists — extend)
│   │   ├── tracker.py             # TrackerService (exists — extend)
│   │   ├── config.py              # Pydantic v2 config for model paths/thresholds
│   │   ├── exceptions.py          # Pipeline-specific exception hierarchy
│   │   ├── detector.py            # PersonDetector — YOLOv8 person detection + classification
│   │   ├── cropper.py             # FrameCropper — per-student image extraction
│   │   ├── merger.py              # PredictionMerger — consolidates 4-model results
│   │   ├── model_lifecycle/       # Model inventory, export, benchmark, deployment modules (US5)
│   │   │   ├── __init__.py
│   │   │   ├── inventory.py        # ModelArtifact scanner, format classifier, checksum
│   │   │   ├── capabilities.py    # Host capability detection (OS/CPU/GPU/CUDA/TensorRT/OpenVINO)
│   │   │   ├── export_orchestrator.py  # Export job queue, status, retry logic
│   │   │   ├── export_worker.py  # Celery: .pt → ONNX/TensorRT/OpenVINO
│   │   │   ├── benchmark_orchestrator.py  # Benchmark scheduling, container mgmt
│   │   │   ├── benchmark_runner.py  # Docker exec, metrics collection
│   │   │   ├── metrics.py        # BenchmarkMetrics dataclass, persistence
│   │   │   ├── deployment_matrix.py  # DeploymentMatrixEntry scoring, ranking
│   │   │   ├── visualizations.py  # Bars, trends, heatmaps (matplotlib/plotly)
│   │   │   ├── explainability.py  # Grad-CAM/SHAP attribution, storage
│   │   │   └── triton_adapter.py  # Triton config.pbtxt, model repo
│   │   └── layers/
│   │       ├── __init__.py        # Layer registry (exists — update exports)
│   │       ├── posture.py         # PostureLayer (exists — rewrite with real YOLO inference)
│   │       ├── horizontal_gaze.py # HorizontalGazeLayer (exists — rewrite)
│   │       ├── depth_gaze.py      # DepthGazeLayer (exists — rewrite)
│   │       └── vertical_gaze.py   # VerticalGazeLayer (exists — rewrite)
│   ├── detections/                # Existing: DetectionFrame, Detection, PyramidPrediction models
│   ├── cameras/                   # Existing: camera management
│   ├── anomalies/                 # Existing: anomaly handling
│   ├── sessions/                  # Existing: monitoring sessions
│   └── ...                        # Other existing apps unchanged
├── tests/
│   ├── unit/
│   │   └── pipeline/              # NEW: unit tests for every pipeline class
│   ├── integration/
│   │   └── test_pipeline_flow.py  # NEW: cross-layer integration tests
│   └── system/
│       └── test_frame_e2e.py      # NEW: end-to-end frame→prediction tests
└── config/
    └── settings/
        └── base.py                # Existing — add PIPELINE_* settings

frontend/
├── src/
│   ├── components/
│   │   └── detection/             # Existing: StudentCard, PredictionsPanel (extend if needed)
│   ├── types/
│   │   └── predictions.ts         # Existing: StudentPrediction interface (extend if needed)
│   └── ...
└── ...

docs/
├── backend/
│   └── apps/
│       └── pipeline/              # NEW/UPDATE: docs for all pipeline source files
│           ├── base.md
│           ├── config.md
│           ├── detector.md
│           ├── cropper.md
│           ├── merger.md
│           ├── model_lifecycle/
│           │   ├── __init__.md
│           │   ├── inventory.md       # ModelArtifact scanner, format classifier
│           │   ├── capabilities.md     # Host capability detection
│           │   ├── export_orchestrator.md  # Export job queue management
│           │   ├── export_worker.md  # Celery export tasks
│           │   ├── benchmark_orchestrator.md  # Benchmark scheduling
│           │   ├── benchmark_runner.md  # Docker execution, metrics
│           │   ├── metrics.md        # BenchmarkMetrics persistence
│           │   ├── deployment_matrix.md  # Deployment ranking
│           │   ├── visualizations.md  # Figure generation
│           │   ├── explainability.md  # Attribution generation
│           │   └── triton_adapter.md  # Triton integration
│           ├── services.md
│           ├── rule_engine.md
│           ├── tracker.md
│           ├── exceptions.md
│           └── layers/
│               ├── __init__.md
│               ├── posture.md
│               ├── horizontal_gaze.md
│               ├── depth_gaze.md
│               └── vertical_gaze.md
└── ...
```

**Structure Decision**: Web application layout — already established. The pipeline app is extended in-place. No new Django apps needed; all pyramid logic lives in `backend/apps/pipeline/`. The existing `detections` app provides the persistence layer (`DetectionFrame`, `Detection`, `PyramidPrediction` models). New files are added to `pipeline/` for detector, cropper, and merger. All four behavior layers are rewritten to call ultralytics YOLO models instead of returning stubs.

## Complexity Tracking

> No constitution violations detected — no justifications needed.

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| _None_ | _N/A_ | _N/A_ |
