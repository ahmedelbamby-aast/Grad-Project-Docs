# Feature Specification: Student Behavior Detection Pyramid

**Feature Branch**: `003-student-behavior-pyramid`  
**Created**: 2026-04-24  
**Status**: Draft  
**Input**: User description: "Build a multi-layered behavior detection pyramid that processes classroom video frames to identify, isolate, and classify student behaviors using a series of specialized detection models."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Detect and Isolate Students in a Video Frame (Priority: P1)

An exam proctor or monitoring system feeds a classroom video frame into the behavior detection pyramid. The system automatically identifies all individuals in the frame, distinguishes students from teachers, extracts the locations of students, and discards teacher locations so that only student regions proceed to downstream analysis.

**Why this priority**: Person detection and student isolation is the foundational layer — no downstream behavior classification can occur without it. This is the entry point of the entire pyramid.

**Independent Test**: Can be fully tested by providing a classroom image with both students and a teacher, verifying the system returns bounding regions for students only and excludes the teacher.

**Acceptance Scenarios**:

1. **Given** a video frame containing 20 students and 1 teacher, **When** the frame is processed by the detection layer, **Then** the system returns location coordinates for each of the 20 students and excludes the teacher.
2. **Given** a video frame containing only a teacher and no students, **When** the frame is processed, **Then** the system returns an empty set of student locations and no data is sent downstream.
3. **Given** a video frame where a teacher is standing among seated students, **When** the frame is processed, **Then** the system correctly distinguishes and separates student locations from the teacher location.

---

### User Story 2 - Crop Individual Students from Frame (Priority: P1)

After student locations are identified, the system crops each individual student from the original full-resolution frame using the detected coordinates. Each cropped image represents a single student and is forwarded independently to all behavior classification layers.

**Why this priority**: Cropping is a prerequisite for all behavior models — each model needs an isolated image of a single student to produce accurate predictions.

**Independent Test**: Can be fully tested by providing a frame and a set of known student coordinates, verifying the system produces one cropped image per student at the correct region.

**Acceptance Scenarios**:

1. **Given** a frame with 5 detected student locations, **When** the copier layer processes them, **Then** 5 individual cropped images are produced, each containing exactly one student.
2. **Given** a student location near the edge of the frame, **When** cropping is performed, **Then** the cropped image is adjusted to stay within frame boundaries without distortion.
3. **Given** a frame where two students are sitting very close together, **When** cropping is performed, **Then** each crop isolates one student as precisely as possible based on the detected coordinates.

---

### User Story 3 - Classify Student Behaviors Across Multiple Models (Priority: P1)

Each cropped student image is sent simultaneously to four behavior classification models. Each model evaluates a specific binary behavior dimension:

- **Standing or Sitting**: Is the student standing or sitting?
- **Looking Left or Right**: Is the student looking to the left or to the right?
- **Looking Forward or Backward**: Is the student looking forward or backward?
- **Looking Up or Down**: Is the student looking up or down?

Each model returns its binary classification result for that student.

**Why this priority**: These four models are the core analytical capability of the system — they produce the behavioral signals that the monitoring system relies on.

**Independent Test**: Can be tested by providing a single cropped student image to each model independently and verifying the model returns one of its two class labels with a confidence score.

**Acceptance Scenarios**:

1. **Given** a cropped image of a student who is seated and looking forward, **When** the image is sent to all four models, **Then** each model returns its respective classification (e.g., sitting, looking-right or looking-left, looking-forward, looking-down or looking-up).
2. **Given** a cropped image of a student who is standing and looking left, **When** processed, **Then** the standing-or-sitting model returns "standing" and the left-or-right model returns "looking-left".
3. **Given** a low-quality or partially occluded cropped image, **When** processed by any model, **Then** the model still returns a prediction along with a confidence score indicating certainty level.

---

### User Story 4 - Merge All Predictions for Each Student (Priority: P2)

The merger layer collects all predictions from all four behavior models for each individual student within the same frame. It combines these into a single unified behavior profile per student and holds or passes this merged result to the next stage in the processing pipeline.

**Why this priority**: While essential for the complete pipeline, merging depends on all upstream layers functioning correctly. It adds no analytical value on its own but is critical for consolidating results.

**Independent Test**: Can be tested by providing mock prediction outputs from the four models for a set of students and verifying the merger produces one consolidated record per student with all four behavior dimensions.

**Acceptance Scenarios**:

1. **Given** predictions from all four models for student A in frame N, **When** the merger layer processes them, **Then** a single merged record is produced containing: student identifier, frame reference, and all four behavior classifications.
2. **Given** predictions for 15 students in a single frame, **When** the merger processes all of them, **Then** 15 individual merged records are produced, each correctly associating the four predictions with the right student.
3. **Given** one model fails to return a prediction for a student, **When** the merger processes available results, **Then** the merged record includes the available predictions and flags the missing model's result as unavailable.

---

### User Story 5 - Model Inventory, Export, Benchmarking, and Deployment Selection (Priority: P1)

The engineering team stores test assets in two operational directories: `Raw Data/` and `models/`. `Raw Data/` contains decompressed classroom video folders used to simulate camera-buffer streaming for workload, performance, and stability testing. `models/` contains available model artifacts (currently `yolo12l.pt` for testing). The system scans `models/`, detects available formats, exports missing runtime formats from `.pt` in the background when possible, benchmarks each supported format on its required execution target, and stores benchmark results for deployment decisions.

**Why this priority**: Production readiness depends on selecting the right model format per hardware target. Without automated inventory, conversion, benchmarking, and deployment mapping, runtime behavior is unpredictable and hard to operate at scale.

**Independent Test**: Can be tested by providing a `models/` directory containing mixed artifacts (`.pt`, `.onnx`, TensorRT engine, OpenVINO IR), running a model-scan job, and verifying the system (a) exports missing artifacts from `.pt` when needed, (b) runs benchmark jobs per target, and (c) saves a deployment matrix with ranking evidence.

**Acceptance Scenarios**:

1. **Given** `models/` contains only `yolo12l.pt`, **When** the model inventory scan runs, **Then** the system marks ONNX/TensorRT/OpenVINO as missing and schedules background export jobs from the `.pt` source.
2. **Given** TensorRT and ONNX artifacts both exist for a model, **When** benchmark orchestration runs, **Then** TensorRT is benchmarked on NVIDIA GPU target and ONNX is benchmarked on CPU target, each in its designated container runtime.
3. **Given** no compatible GPU runtime is detected on the host, **When** deployment selection executes, **Then** GPU-only variants are skipped with explicit logs and the best CPU-compatible variant is selected.
4. **Given** benchmark jobs complete for a model family, **When** results are persisted, **Then** latency, throughput, memory usage, error rate, and selected deployment recommendation are stored in the database.
5. **Given** historical benchmark data exists, **When** an operator requests result visualization, **Then** the system can generate figures (comparison bars, trend lines, and platform-format matrix heatmap) and expose them for later frontend rendering.
6. **Given** explainability is enabled for an evaluation run, **When** a benchmark sample is processed, **Then** explainability artifacts (for example heatmaps/attribution overlays) are generated and linked to the benchmark record.

---

### Edge Cases

- What happens when no persons are detected in the frame at all?
- What happens when the detection model cannot determine whether a person is a student or a teacher?
- What happens when a student is only partially visible in the frame (e.g., only head or only torso)?
- What happens when the video frame is too dark, blurry, or corrupted to process?
- What happens when two students overlap significantly in the frame?
- What happens when one or more of the four behavior models is unavailable or times out?
- What happens when a model format is missing and no `.pt` source model exists to export from?
- What happens when TensorRT export succeeds but runtime validation fails on the detected GPU/CUDA stack?
- What happens when the host is Windows with NVIDIA hardware present but GPU inference libraries are unavailable?
- What happens when Triton-backed deployment fails health checks during format benchmarking?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST accept a video frame as input and detect all persons present in the frame.
- **FR-002**: System MUST classify each detected person as either a "student" or a "teacher" and extract location coordinates for each.
- **FR-003**: System MUST discard teacher locations and pass only student locations to downstream layers.
- **FR-004**: System MUST crop each detected student from the original frame using the student's location coordinates, producing one cropped image per student.
- **FR-005**: System MUST send each cropped student image to all four behavior classification models in parallel.
- **FR-006**: The standing-or-sitting model MUST classify each student image as either "standing" or "sitting".
- **FR-007**: The looking-left-or-right model MUST classify each student image as either "looking left" or "looking right".
- **FR-008**: The looking-forward-or-backward model MUST classify each student image as either "looking forward" or "looking backward".
- **FR-009**: The looking-up-or-down model MUST classify each student image as either "looking up" or "looking down".
- **FR-010**: Each behavior model MUST return a confidence score alongside its classification.
- **FR-011**: The merger layer MUST collect predictions from all four behavior models for each student in the same frame.
- **FR-012**: The merger layer MUST produce one unified behavior record per student per frame, combining all four model outputs.
- **FR-013**: The merger layer MUST associate each merged record with a student identifier and frame reference.
- **FR-014**: The merger layer MUST handle partial results when one or more models fail, flagging missing predictions rather than discarding the entire record.
- **FR-015**: System MUST process frames from a live or recorded video feed, handling frames sequentially.
- **FR-016**: System MUST support a `Raw Data/` directory containing decompressed video subfolders used to replay videos as camera-buffer-like streams for performance and workload testing.
- **FR-017**: System MUST support a `models/` directory scan phase that inventories all model artifacts, including filename, extension, inferred format, and compatibility status.
- **FR-018**: System MUST recognize at minimum `.pt`, `.onnx`, TensorRT engine (`.engine`/`.trt`), and OpenVINO IR artifacts for deployment planning.
- **FR-019**: If required deployment formats are missing and a `.pt` source exists, system MUST trigger background export jobs sequentially per model and log every step and failure reason.
- **FR-020**: If required deployment formats are missing and no `.pt` source exists, system MUST flag the model as incomplete, persist an actionable error state, and continue processing other models.
- **FR-021**: System MUST validate runtime capabilities on startup (OS, CPU, GPU availability, CUDA/TensorRT/OpenVINO runtime readiness) before scheduling benchmark/deployment jobs.
- **FR-022**: System MUST map model formats to required execution targets: `.pt` -> GPU-preferred fallback policy, TensorRT -> NVIDIA GPU, `.onnx` -> CPU, OpenVINO -> Intel CPU/Intel discrete GPU.
- **FR-023**: System MUST execute per-model benchmark jobs in the background and store benchmark metrics by format, hardware target, and runtime container image.
- **FR-024**: Benchmark execution MUST include repeatable measurements for at least latency, throughput, warmup time, memory/VRAM usage, and run stability/error rate.
- **FR-025**: System MUST build and persist a deployment matrix that records, per model family and format, the recommended deployment destination and ranking rationale.
- **FR-026**: System MUST persist benchmark results and deployment recommendations in the database for future analysis and frontend visualization.
- **FR-027**: System MUST support figure-generation planning from benchmark data, including comparison bars, trend charts, and platform-format matrix visualizations.
- **FR-028**: System MUST support explainability analysis planning for behavior model outputs, including generation and storage of explainability artifacts linked to model/version and benchmark run.
- **FR-029**: All export, benchmark, and deployment-selection workflows MUST run in Dockerized environments with format-specific images/containers.
- **FR-030**: Triton Inference Server integration/research is a mandatory requirement for production-grade inference orchestration and MUST be included in the deployment strategy.
- **FR-031**: System MUST expose structured logs and status events for model scan, export, benchmark, and deployment decision phases.

### Key Entities

- **Frame**: A single image captured from the video feed; the input to the detection layer. Key attributes: frame number/timestamp, resolution, source reference.
- **Person Detection**: The result of detecting a person in a frame. Key attributes: bounding box coordinates, person classification (student or teacher), confidence score.
- **Cropped Student Image**: An image region extracted from the original frame containing a single student. Key attributes: source frame reference, student identifier, pixel data, bounding coordinates.
- **Behavior Prediction**: The output of a single behavior model for one student. Key attributes: model name, classification label, confidence score, student reference.
- **Merged Behavior Record**: The consolidated result combining all behavior predictions for one student in one frame. Key attributes: student identifier, frame reference, list of all behavior predictions, completeness flag.
- **Model Artifact**: A discovered file in `models/` representing a deployable or source model. Key attributes: path, filename, extension, format type, version tag, checksum, creation time.
- **Model Capability Snapshot**: Host/runtime capability record captured before export/benchmark. Key attributes: OS, CPU type, GPU presence, CUDA availability, TensorRT availability, OpenVINO availability.
- **Model Export Job**: Background task converting source `.pt` to required deployment artifacts. Key attributes: source model, target format, status, start/end timestamps, logs, error details.
- **Benchmark Run**: Performance test execution for a model-format-target combination. Key attributes: runtime target, container image, batch size, latency, throughput, memory/VRAM, stability metrics.
- **Deployment Matrix Entry**: Recommendation mapping from model format to deployment destination with ranking. Key attributes: model id, format, platform, score, decision reason, selected flag.
- **Explainability Artifact**: Visualization/attribution output tied to inference samples. Key attributes: algorithm name, artifact path, sample reference, generation metadata.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: The system correctly identifies and separates students from teachers in at least 90% of test frames containing both roles.
- **SC-002**: Each cropped student image contains the intended student with no more than 10% of the crop area occupied by background or adjacent persons.
- **SC-003**: Each behavior model achieves at least 85% classification accuracy on its binary task when tested against labeled evaluation data.
- **SC-004**: The merger layer produces a complete behavior record (all four predictions present) for at least 95% of detected students per frame under normal operating conditions.
- **SC-005**: The full pyramid processes a single frame (detection through merger) within 3 seconds for a classroom of up to 30 students.
- **SC-006**: The system handles frames where no students are detected without errors, producing an empty result set.
- **SC-007**: When a behavior model is unavailable, the system continues processing with the remaining models and flags incomplete records rather than failing entirely.
- **SC-008**: Model inventory scan detects and classifies supported artifacts in `models/` with 100% filename/extension coverage for files in scope.
- **SC-009**: For models with `.pt` sources, missing required formats are exported automatically in background jobs with >=95% job completion success under valid runtime prerequisites.
- **SC-010**: Benchmark suite completes for all available format-target pairs and stores full metric records for >=99% of scheduled benchmark jobs.
- **SC-011**: Deployment matrix generation produces one ranked recommendation per model family and target platform with reproducible scoring inputs.
- **SC-012**: Figure generation pipeline can produce at least 3 benchmark visual outputs (format comparison, trend over runs, deployment matrix view) for a completed benchmark batch.
- **SC-013**: Explainability workflow generates and stores at least one attribution artifact per benchmarked model format in evaluation mode.

## Assumptions

- The five pre-trained models (1 person detection/classification + 4 behavior models) are already trained and available for integration. This feature focuses on building the pipeline that connects them, not on training the models.
- The person detection model can distinguish students from teachers based on visual appearance or contextual cues (e.g., position, attire, posture).
- The classroom video feed provides frames at a resolution sufficient for person detection and behavior classification.
- Each frame is processed independently — no cross-frame tracking or temporal analysis is required for this feature.
- The four behavior models expect a cropped image of a single person as input and return a binary classification with a confidence score.
- "Standing or sitting" covers the two primary posture states; intermediate states (e.g., leaning, crouching) are classified into the nearest category by the model.
- Gaze direction labels (left/right, forward/backward, up/down) refer to the student's head orientation relative to the camera's perspective.
- `Raw Data/` contains multiple decompressed subfolders of test videos and is used to simulate camera-buffer streams for system stress and workload testing.
- `models/` currently contains testing models (currently `yolo12l.pt`) and will later include multiple export formats.
- Runtime environment may include Windows hosts with CPU-only execution or NVIDIA GPU execution (for example RTX 5030 Ti 4GB VRAM); runtime capability detection determines which acceleration path is actually available.
- OpenVINO deployment is targeted for Intel CPU/Intel discrete GPU environments.
- Triton Inference Server is a mandatory production-track dependency and must be considered in deployment architecture decisions.
