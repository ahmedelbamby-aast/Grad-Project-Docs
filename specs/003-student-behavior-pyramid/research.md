# Research: Student Behavior Detection Pyramid

**Branch**: `003-student-behavior-pyramid` | **Date**: 2026-04-24 | **Spec**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md)

## Research Tasks

### R-001: Person Detection Model Selection (YOLO variant)

**Decision**: Use YOLOv8 via the `ultralytics` library (already in requirements.txt as `ultralytics==8.3.61`).

**Rationale**: YOLOv8 provides out-of-the-box person detection with the COCO-pretrained `yolov8n.pt` (nano) or `yolov8s.pt` (small) models. Person class is COCO class 0. The `ultralytics` package is already a project dependency, eliminating any new integration overhead. YOLOv8 supports both GPU (CUDA) and CPU inference natively, satisfying constitution §VI cross-platform requirement.

**Inference API**:
```python
from ultralytics import YOLO
model = YOLO("path/to/model.pt")

results = model.predict(
    source=frame,        # numpy array (HWC, BGR, uint8 0-255)
    conf=0.25,           # confidence threshold
    iou=0.7,             # NMS IoU threshold
    device="cuda:0",     # or "cpu"
    imgsz=640,           # inference resolution
    verbose=False,       # suppress console output
)
```

**Detection results structure**:
- `results[0].boxes.xyxy` — tensor [N, 4]: x1, y1, x2, y2 pixel coordinates
- `results[0].boxes.conf` — tensor [N]: confidence scores
- `results[0].boxes.cls` — tensor [N]: class indices (map via `model.names[int(cls)]`)

**Classification results structure**:
- `results[0].probs.top1` — int: top-1 class index
- `results[0].probs.top1conf` — float: confidence of top-1 prediction
- `results[0].names[top1]` — str: class label

**Alternatives considered**:
- **YOLOv5**: Older, also supported by `ultralytics`, but v8 has better accuracy/speed trade-offs and a cleaner API.
- **Detectron2**: More powerful for instance segmentation but heavier, slower setup, and overkill for bounding-box-only detection.
- **MediaPipe**: Lightweight but focused on pose estimation rather than person detection with classification.

---

### R-002: Student vs. Teacher Classification Strategy

**Decision**: Use a custom-trained YOLOv8 detection model with two classes: `student` (class 0) and `teacher` (class 1). This model performs detection + classification in a single inference pass.

**Rationale**: The spec assumes "the person detection model can distinguish students from teachers based on visual appearance or contextual cues." The spec's assumptions state: "The five pre-trained models (1 person detection/classification + 4 behavior models) are already trained and available for integration." This implies a single detection+classification model. Single-pass detection+classification is faster than a two-model pipeline, helping meet SC-005 (≤3s for 30 students).

**Alternatives considered**:
- **Two-stage (detect + classify)**: Use COCO person detector then a separate student/teacher classifier. Adds ~50-100ms per person. Rejected for latency.
- **Rule-based heuristic** (teacher = largest bbox, or person at front): Fragile, fails when teachers sit or students stand at board. Rejected.
- **Pose-based classification**: Use skeleton keypoints. Adds complexity without clear accuracy benefit. Rejected.

---

### R-003: Image Cropping Strategy

**Decision**: Use NumPy array slicing on the original frame using bounding box coordinates from the detector. Apply boundary clamping for edge-of-frame detections.

**Rationale**: NumPy slicing creates a view (zero-copy) — the fastest possible approach. Only call `.copy()` when the crop needs to outlive the original frame buffer. No image processing library is needed beyond NumPy. OpenCV (installed as an ultralytics dependency) handles any needed resizing of crops to match behavior model input dimensions.

**Pattern**:
```python
def crop_student(frame: np.ndarray, bbox: tuple[int, int, int, int]) -> np.ndarray:
    h, w = frame.shape[:2]
    x1, y1, x2, y2 = bbox
    x1, y1 = max(0, x1), max(0, y1)
    x2, y2 = min(w, x2), min(h, y2)
    return frame[y1:y2, x1:x2].copy()
```

**Alternatives considered**:
- **PIL/Pillow `crop()`**: Requires numpy→PIL→numpy roundtrip. Rejected for overhead.
- **`cv2.getRectSubPix`**: Requires format conversion. Rejected.
- **torchvision `crop()`**: Requires Tensor conversion. Only beneficial if behavior models expect PyTorch tensors directly. Rejected.

---

### R-004: Behavior Classification Model Architecture

**Decision**: Use four independent YOLOv8 classification models (`yolov8n-cls`), each trained for a binary classification task:
1. `posture_model.pt` → standing / sitting
2. `horizontal_gaze_model.pt` → looking left / looking right
3. `depth_gaze_model.pt` → looking forward / looking backward
4. `vertical_gaze_model.pt` → looking up / looking down

**Rationale**: The spec explicitly defines four independent binary classifiers. YOLOv8-cls provides a lightweight classification architecture that shares the same `ultralytics` API as the detection model, ensuring consistent model loading and inference patterns. Each model outputs two class probabilities via `probs.top1` and `probs.top1conf`.

**Alternatives considered**:
- **Single multi-label model**: One model predicting all 4 dimensions simultaneously. Rejected because: (a) the spec defines them as independent models, (b) a single model couples all dimensions making individual updates impossible, (c) violates the Strategy pattern where each layer is independently swappable (constitution §I Open/Closed).
- **Custom CNN (e.g., ResNet-18)**: Viable but adds a second model framework alongside ultralytics. Rejected for consistency.
- **MediaPipe Pose + heuristic**: Could work for posture but unreliable for fine-grained gaze direction from classroom distance. Rejected.

---

### R-005: Thread Safety and Parallel Inference Strategy

**Decision**: Use `concurrent.futures.ThreadPoolExecutor` with `max_workers=4` for parallel behavior classification. Each of the 4 behavior models is a separate `YOLO` instance — they do NOT share model objects across threads.

**Rationale**:
- **YOLO model objects are NOT thread-safe.** Concurrent `predict()` calls on a shared instance cause race conditions due to mutable internal state. However, since each behavior model is a *different* model loaded from a *different* `.pt` file, they each have their own `YOLO` instance and don't contend with each other.
- YOLO inference calls into C++/CUDA extensions that release the Python GIL, so threads achieve true parallelism for the compute-heavy portion — even on standard Python 3.13 (with GIL enabled).
- `multiprocessing` would require serializing images across process boundaries and loading each model into separate GPU memory — wasteful for 4 lightweight classifiers.
- `asyncio` adds complexity with no benefit since `predict()` is synchronous; you'd still need `run_in_executor`.

**Pattern**:
```python
from concurrent.futures import ThreadPoolExecutor, as_completed

with ThreadPoolExecutor(max_workers=4) as executor:
    futures = {
        executor.submit(layer.predict, crop): layer.name
        for layer in self.layers
    }
    for future in as_completed(futures):
        name = futures[future]
        try:
            result = future.result(timeout=5.0)
            merged[name] = result
        except Exception as exc:
            errors[name] = str(exc)
```

**Alternatives considered**:
- **Sequential inference** (existing PipelineService pattern): 4× slower. At ~50ms/call on GPU, 30 students × 4 models sequential = ~6s. Parallel = ~1.5s. Rejected per SC-005.
- **ProcessPoolExecutor for CPU**: Adds inter-process communication overhead for image serialization. ThreadPoolExecutor works because native extensions release GIL. Rejected.
- **Celery tasks per model**: Too much overhead for per-student per-model granularity. Celery is better suited for frame-level dispatch. Rejected.

---

### R-006: Model Loading and Lifecycle Management

**Decision**: Implement lazy loading at the class level in each layer using double-checked locking. Each layer class holds its own `_model` and `_lock` as class-level attributes, initialized on first `predict()` call. Model paths come from Pydantic v2 `BaseSettings` configuration.

**Rationale**: Constitution §VI mandates "lazy model loading: pyramid layers loaded on-demand per camera session, not all at startup." Double-checked locking is thread-safe and avoids repeated lock acquisition after initialization.

**Lazy loading pattern**:
```python
import threading
from ultralytics import YOLO

class PostureLayer(BasePyramidLayer):
    name = 'posture'
    _model: YOLO | None = None
    _lock = threading.Lock()

    @classmethod
    def _get_model(cls, path: str) -> YOLO:
        if cls._model is None:
            with cls._lock:
                if cls._model is None:
                    cls._model = YOLO(path)
        return cls._model
```

**Pydantic configuration pattern**:
```python
from pydantic import Field
from pydantic_settings import BaseSettings

class PipelineConfig(BaseSettings):
    model_config = {"env_prefix": "PYRAMID_"}

    person_model_path: str = Field(default="models/person_detector.pt")
    posture_model_path: str = Field(default="models/posture.pt")
    horizontal_gaze_model_path: str = Field(default="models/horizontal_gaze.pt")
    depth_gaze_model_path: str = Field(default="models/depth_gaze.pt")
    vertical_gaze_model_path: str = Field(default="models/vertical_gaze.pt")
    confidence_threshold: float = Field(default=0.25, ge=0.0, le=1.0)
    device: str = Field(default="cpu")
    max_inference_timeout: float = Field(default=5.0, ge=0.1)
```

GPU/CPU auto-detection at config load:
```python
import torch
device = "cuda:0" if torch.cuda.is_available() else "cpu"
```

**Alternatives considered**:
- **`AppConfig.ready()`**: Runs at import time in every process (including management commands, migrations). Forces GPU allocation unnecessarily. Rejected.
- **Singleton ModelRegistry class**: Viable but adds indirection. Per-class lazy loading is simpler and follows the Strategy pattern more closely. Rejected.
- **Load models per request**: Extreme overhead (~2-5s per model load). Rejected.

---

### R-007: Result Merging and Partial Failure Handling

**Decision**: The `PredictionMerger` collects results from all four behavior models for each student. If a model fails or times out, the merger records available predictions and flags missing ones with `available=False`. This maps directly to FR-014.

**Merger output pattern**:
```python
from dataclasses import dataclass, field

@dataclass
class LayerResult:
    label: str = ""
    confidence: float = 0.0
    available: bool = True
    error: str | None = None

@dataclass
class MergedPrediction:
    tracking_id: str
    frame_ref: str
    posture: LayerResult = field(default_factory=LayerResult)
    horizontal_gaze: LayerResult = field(default_factory=LayerResult)
    depth_gaze: LayerResult = field(default_factory=LayerResult)
    vertical_gaze: LayerResult = field(default_factory=LayerResult)

    @property
    def is_complete(self) -> bool:
        return all(
            getattr(self, f).available
            for f in ['posture', 'horizontal_gaze', 'depth_gaze', 'vertical_gaze']
        )
```

**Rationale**: FR-014 requires "handle partial results when one or more models fail, flagging missing predictions rather than discarding the entire record." Downstream consumers (rule engine, WebSocket, frontend) can check the availability flag. The existing `PyramidPrediction` model already has fields for all four dimensions; empty strings represent unavailable predictions.

**Alternatives considered**:
- **Reject partial predictions**: Contradicts FR-014. Rejected.
- **Retry failed models**: Could delay frame processing beyond SC-005's 3s target. Better to flag and let downstream consumers handle. Rejected.
- **Optional fields (None for missing)**: Less explicit than a dedicated flag. Rejected.

---

### R-008: Frame Input Source Integration

**Decision**: The pipeline accepts frames as NumPy arrays (`np.ndarray` of shape `(H, W, 3)` in BGR format, standard OpenCV convention). Frame acquisition from video feeds (RTSP, file, webcam) is handled by existing camera/session infrastructure. The pipeline entry point is `PipelineService.process_frame(frame, frame_meta)`.

**Rationale**: The spec states "each frame is processed independently" (Assumption 5). Decoupling frame acquisition from pipeline processing follows Single Responsibility (constitution §I). The existing `DetectionFrame` model stores frame metadata; the pipeline only needs the pixel data and metadata reference.

**Alternatives considered**:
- **Accept file paths**: Adds I/O latency for reading from disk. For live feeds, frames are already in memory. Rejected.
- **Accept base64 strings**: Encoding/decoding overhead. Only useful for HTTP API input. Rejected.

---

### R-009: Model Artifact Inventory and Format Detection

**Decision**: Treat root `models/` as the canonical inventory source for deployment artifacts and classify files by extension into format families: `.pt`, `.onnx`, TensorRT (`.engine`/`.trt`), and OpenVINO IR (`.xml` + `.bin`).

**Rationale**: User Story 5 and FR-017/FR-018 require the system to be format-aware and to detect missing artifacts before scheduling export/benchmark workflows. Extension-first detection is deterministic and fast; pairing rules handle multi-file OpenVINO artifacts.

**Rules**:
- `.pt` -> source/original PyTorch artifact
- `.onnx` -> CPU target baseline
- `.engine` or `.trt` -> TensorRT runtime target
- `.xml` + matching `.bin` -> OpenVINO IR complete artifact

Missing format detection is computed per model family by normalized stem.

---

### R-010: Runtime Capability Detection Policy

**Decision**: Capture runtime capability snapshot at startup and before benchmark scheduling, including OS, CPU details, GPU presence, CUDA availability, TensorRT availability, and OpenVINO runtime readiness.

**Rationale**: Host constraints vary (Windows CPU-only vs NVIDIA GPU-enabled). FR-021 requires scheduling decisions to be capability-driven instead of static assumptions.

**Capability precedence**:
1. TensorRT path enabled only when NVIDIA GPU + CUDA + TensorRT runtime are all present.
2. OpenVINO path enabled only when OpenVINO runtime is present.
3. ONNX CPU path always available as baseline fallback.

---

### R-011: Export Strategy for Missing Artifacts

**Decision**: When required formats are missing and `.pt` exists, queue sequential background export jobs (`.pt` -> ONNX/TensorRT/OpenVINO) with per-step structured logs and explicit failure states.

**Rationale**: FR-019/FR-020 require autonomous conversion when possible and non-blocking continuation when source artifacts are missing. Sequential execution avoids resource spikes on constrained GPUs.

**Failure policy**:
- Missing `.pt` source -> mark incomplete, persist actionable error, continue processing other model families.
- Export failure -> preserve partial outputs, log root cause, continue with available formats.

---

### R-012: Benchmark Methodology and Metrics

**Decision**: Benchmark each model-format-target tuple in Dockerized runtimes using replay streams from root `Raw Data/` to emulate camera-buffer ingestion.

**Rationale**: FR-023/FR-024/FR-029 require reproducible, environment-isolated benchmarks with production-like workload patterns.

**Required metrics per run**:
- latency (ms)
- throughput (FPS)
- warmup time (ms)
- memory usage (MB)
- VRAM usage (MB, when GPU path)
- error rate/stability

---

### R-013: Deployment Matrix Scoring

**Decision**: Build a weighted scoring model per target platform and format, then persist one selected recommendation per model family/platform pair.

**Rationale**: FR-025 requires deterministic recommendation evidence. Scoring must be reproducible from stored benchmark metrics.

**Example scoring factors**:
- lower latency (higher score)
- higher throughput (higher score)
- lower error rate (higher score)
- resource fit (memory/VRAM within constraints)

---

### R-014: Explainability and Figure Generation

**Decision**: Generate explainability artifacts (for example attribution heatmaps) on benchmark samples and generate benchmark figures (comparison bars, trend lines, matrix heatmap) as persisted artifacts linked to benchmark batches.

**Rationale**: FR-027/FR-028 require planning for interpretability and reporting outputs for later frontend display.

---

### R-015: Triton Production-Path Validation

**Decision**: Include Triton Inference Server as a mandatory validation target in benchmark/deployment research, with dedicated health checks and failure diagnostics.

**Rationale**: FR-030 makes Triton integration/research mandatory for production-grade performance. Validation ensures deployment plans include serving-layer behavior, not just raw model inference.

## Summary of Resolved Unknowns

| Unknown | Resolution |
|---------|------------|
| Person detection model | YOLOv8 via ultralytics (already in dependencies) |
| Student/teacher classification | Custom-trained YOLOv8 detection model with 2 classes (single-pass) |
| Cropping approach | NumPy array slicing with boundary clamping |
| Behavior model architecture | 4× YOLOv8-cls binary classifiers |
| Thread safety | YOLO models are NOT thread-safe — use separate instances per model |
| Parallel inference | ThreadPoolExecutor (works for both GPU and CPU since native extensions release GIL) |
| Model lifecycle | Per-class lazy loading with double-checked locking |
| Configuration | Pydantic v2 BaseSettings with `PYRAMID_` env prefix |
| Partial failure handling | PredictionMerger with per-layer availability flag |
| Frame input format | NumPy ndarray (H,W,3) BGR |
| Model inventory | Root `models/` extension-based inventory with OpenVINO pair validation |
| Runtime capability policy | Startup capability snapshot drives scheduling decisions |
| Missing format exports | Sequential background export from `.pt` with non-blocking failure policy |
| Benchmark methodology | Dockerized runs using `Raw Data/` replay with required metric set |
| Deployment recommendations | Weighted deployment matrix scoring persisted per model/platform |
| Explainability/reporting | Persist explainability and figure artifacts linked to benchmark runs |
| Triton strategy | Mandatory production-path validation with health diagnostics |

## Related Documents

- [spec.md](spec.md) — Feature specification
- [plan.md](plan.md) — Implementation plan
- [data-model.md](data-model.md) — Entity definitions and relationships
- [contracts/rest-api.md](contracts/rest-api.md) — REST API contracts
- [contracts/websocket-api.md](contracts/websocket-api.md) — WebSocket message contracts
- [../../.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Project governance
