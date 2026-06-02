# Threading & Multi-Processing Analysis

**Last updated:** 2026-05-30

## Exam Monitoring Dashboard Project

**Date**: 2026-05-29  
**Scope**: Windows Development → Linux Production Deployment  
**Status**: Comprehensive Analysis

---

## Implementation Status (updated 2026-05-30)

This analysis predates the migration to **Triton-only** production inference. The
async **Triton offline batch queue** (`pending_frames` + `_flush_pending` +
`max_concurrency`/`max_inflight` + dynamic batching in
`backend/apps/video_analysis/tasks.py::_run_triton_frame_level_inference`,
driven by `TRITON_OFFLINE_BATCH_QUEUE_*`) already implements the concurrent /
batched **CPU-, I/O-, and GPU-bound** dispatch this doc proposed via process
pools. The proposed scaffolds were reconciled against that reality:

| Proposed scaffold | Outcome | Reason |
|---|---|---|
| Frame reading pipeline (`frame_pipeline.py`) | **WIRED** into the Triton decode loop, gated by `TRITON_OFFLINE_THREADED_DECODE` (default on) with inline fallback | Decouples disk read + cv2 decode from preprocess + dispatch — the one remaining serial I/O cost |
| Parallel multi-model inference (`multi_model_parallel.py`) | **REMOVED** | Wrapped the local ultralytics path, which is skipped by policy under `triton_only`; inert in production and Windows dev (sequential fallback) |
| DB write buffer (`db_buffer.py`) | **REMOVED** | Only batched `Frame`+`Detection`; the real persistence path has `Detection → BoundingBox → StudentTrack` FK/get-or-create dependencies it could not express safely |

Net result: concurrency is provided by the Triton batch queue (already active) +
threaded decode (newly wired); the obsolete scaffolds were deleted to keep the
codebase free of dead, architecture-mismatched code.

---

## Project Overview

The **Exam Monitoring Dashboard** is a real-time surveillance system with:
- **React 19 SPA** frontend with WebSocket live updates
- **Django 5.1.5 ASGI** backend (Daphne + Channels)
- **Celery 5.4.0** distributed task queue
- **PostgreSQL 16** + **Redis 7** for persistence & messaging
- **Triton Inference Server** + Local inference adapters (OpenVINO, TensorRT, ONNX)
- **Multi-Model YOLO** pipeline (person detection, posture, gaze, hand-raising)
- **ByteTrack/BoT-SORT** object tracking with ReID embeddings
- **FFmpeg-based** video encoding/decoding for offline & live streams

---

## Current Concurrency Model

### Frontend
- Vite dev server (single-threaded)
- React hooks for WebSocket subscriptions
- Browser-side JavaScript (single-threaded event loop)

### Backend
- **Django ASGI** (Daphne): async coroutines, **not true threads**
  - `async_to_sync()` adapters for blocking Django ORM
  - Channels consumer groups for WebSocket broadcasting
- **Celery Workers**: Process-based concurrency
  - **Windows**: `solo` pool (no multiprocessing, single serial queue)
  - **Linux**: `prefork` pool (multi-process, default: CPU count concurrency)
  - Concurrency configurable via `CELERY_WORKER_CONCURRENCY` (capped by `CELERY_GPU_CONCURRENCY_CAP`)

### Inference Runtime
- **OpenVINO**: Serialized via `_openvino_process_lock()` (file + thread lock)
  - Lines 35–76 in `backend/apps/pipeline/multi_model.py`
  - Prevents concurrent OpenVINO compile/infer across workers
- **Triton**: HTTP/gRPC requests, no built-in client serialization
- **TensorRT**: Used within Triton, not directly in local adapters (production only)

### Video Processing
- **FFmpeg subprocesses**: Spawned via `subprocess.run()` / `subprocess.Popen()`
- **OpenCV (cv2)**: Single-threaded frame I/O in most cases
- **Ultralytics YOLO**: Uses OpenVINO/TensorRT backends; framework doesn't handle parallelism

---

## Problem Areas & Bottlenecks

### 🔴 HIGH PRIORITY

#### 1. **Video I/O Serialization** (I/O-Bound)
**Location**: `backend/apps/video_analysis/tasks.py`, `backend/apps/tracking/video_exporter.py`

**Problem**:
- Offline video processing reads frames sequentially
- FFmpeg encoding is a blocking subprocess
- No pipelining between frame reading, inference, and writing
- Frame preprocessing blocks inference scheduling

**Current Flow**:
```
Read Frame 1 → Preprocess → Infer → Write → Read Frame 2 → ...
```

**Impact**: 
- Video read/write I/O waits for inference completion
- For 30 FPS, 10-min video = 18K frames, each blocking
- On slow storage or network streams, causes backpressure

**Windows Limitation**:
- No `fork()`, so multiprocessing shares GIL (Global Interpreter Lock)
- Celery `solo` pool prevents worker-level parallelism
- Background encode jobs block the main task

---

#### 2. **Multi-Model Inference Serialization** (CPU-Bound)
**Location**: `backend/apps/pipeline/multi_model.py` lines 431–555

**Problem**:
- Models run **sequentially**: person → posture → gaze × 3 → hand
- ~8 forward passes per frame, each takes 50–500ms
- OpenVINO lock serializes even on multi-core systems
- No inter-model parallelism

**Current Flow**:
```
Model 1 (person) → Model 2 (posture) → Model 3 (gaze_h) → Model 4 (gaze_d) → Model 5 (gaze_v) → Model 6 (hand)
```

**Optimization Opportunity**:
- Models 2–6 are **independent** once bounding boxes exist
- Parallelizable with subprocess pools (one process per model)
- Avoid GIL on Windows by using separate processes

---

#### 3. **ReID Embedding Computation** (CPU-Bound)
**Location**: `backend/apps/tracking/embeddings.py`

**Problem**:
- Each detected person crop → embedding vector (cosine similarity)
- Happens **per-frame** across all tracked IDs
- No batching or multi-threaded crop processing
- Redis I/O for embedding cache is blocking

**Impact**:
- 30 people/frame × 30 FPS = 900 embeddings/sec
- Sequential crop extraction + inference = latency spike

---

#### 4. **Live Stream Frame Capture** (I/O-Bound)
**Location**: `backend/apps/video_analysis/tasks.py::run_live_stream_inference`

**Problem**:
- RTSP/go2rtc stream polling is blocking
- Frame decompression in main task thread
- No decoupling of capture → preprocess → inference

**Current Flow**:
```
Capture Frame (blocking) → Decompress → Preprocess → Send to Inference → Wait for Result
```

**Windows Limitation**:
- Celery `solo` pool = single serial queue for live inference
- Multiple camera streams queue up serially
- Can drop frames if inference is slow

---

#### 5. **Database Write Batching** (I/O-Bound)
**Location**: `backend/apps/video_analysis/tasks.py` (Detection, Frame, StudentTrack writes)

**Problem**:
- Each frame → 1–2 DB writes (Detection, StudentTrack)
- **18K frames** = 18K–36K separate ORM inserts
- Django ORM is synchronous, blocking
- No bulk_create with transaction batching in all paths

**Impact**:
- Database round-trip latency compounds
- PostgreSQL connection pool exhaustion risk under load

---

#### 6. **WebSocket Broadcasting Blocking** (I/O-Bound)
**Location**: `backend/config/settings/*.py`, Channels consumer groups

**Problem**:
- Broadcasting detection events to connected clients uses `sync_to_async()`
- If many clients connected, blocking the inference task
- No dedicated broadcast thread/queue

---

### 🟡 MEDIUM PRIORITY

#### 7. **Triton Request Queuing** (I/O-Bound)
**Location**: `backend/apps/pipeline/inference_runtime.py`

**Problem**:
- HTTP requests to Triton are blocking (despite async-capable client)
- No connection pooling explicit in current code
- Timeout handling uses synchronous sleep

---

#### 8. **Video Encoding Quality Settings** (CPU-Bound)
**Location**: `backend/apps/tracking/video_exporter.py`

**Problem**:
- H.264 encoding (`PYRAMID_OUTPUT_VIDEO_PRESET`) runs in single FFmpeg process
- Presets like `medium`/`slow` use multiple threads within FFmpeg, but allocation is opaque
- No inter-frame parallelism (most H.264 implementations are sequential)

---

## Recommended Multi-Threading Solutions

### ✅ For I/O-Bound Operations (Threads)

Use when:
- **I/O wait dominates** (disk, network, DB)
- **Avoiding multi-process complexity** (shared memory, IPC overhead)
- **Windows-compatible** (no `fork()` requirement)

#### 1. **Frame Reading Pipeline** (High Priority)
```python
# backend/apps/video_analysis/frame_pipeline.py (NEW)
from concurrent.futures import ThreadPoolExecutor
from queue import Queue

class FrameReadPipeline:
    def __init__(self, video_path: str, max_queue_size: int = 30):
        self.executor = ThreadPoolExecutor(max_workers=2)  # 1 reader, 1 preprocessor
        self.frame_queue = Queue(maxsize=max_queue_size)
    
    def start_reader_thread(self):
        """Thread 1: Read frames from disk/stream (I/O-bound)"""
        self.executor.submit(self._reader_loop)
    
    def start_preprocessor_thread(self):
        """Thread 2: Resize, normalize, augment (CPU, but async to inference)"""
        self.executor.submit(self._preprocessor_loop)
    
    def get_next_frame(self):
        """Main inference thread polls preprocessed frames"""
        return self.frame_queue.get(timeout=5)
```

**Benefits**:
- Decouple I/O from inference
- Inference starts on Frame N while reading Frame N+K
- Works on Windows (solo pool), Linux (prefork pool)

**Implementation**:
- Wrap `run_multi_model_inference_streaming()` to use pipeline
- Update `process_video_upload` task to instantiate pipeline

---

#### 2. **Database Write Buffer** (High Priority)
```python
# backend/apps/video_analysis/db_buffer.py (NEW)
from threading import Thread
from queue import Queue
from django.db import transaction

class DBWriteBuffer:
    def __init__(self, batch_size: int = 100, flush_interval_ms: int = 500):
        self.batch_queue = Queue()
        self.batch_size = batch_size
        self.flush_interval = flush_interval_ms / 1000
        self.daemon_thread = Thread(target=self._flush_loop, daemon=True)
    
    def add_frame(self, frame: Frame):
        self.batch_queue.put(('frame', frame))
    
    def add_detection(self, det: Detection):
        self.batch_queue.put(('detection', det))
    
    def _flush_loop(self):
        """Background thread flushes batched writes to DB"""
        buffer = {'frames': [], 'detections': []}
        while True:
            # Collect up to batch_size items
            while len(buffer['frames']) + len(buffer['detections']) < self.batch_size:
                try:
                    kind, obj = self.batch_queue.get(timeout=self.flush_interval)
                    buffer[f"{kind}s"].append(obj)
                except:
                    break
            
            # Write batch to DB
            if buffer['frames'] or buffer['detections']:
                with transaction.atomic():
                    Frame.objects.bulk_create(buffer['frames'], batch_size=50)
                    Detection.objects.bulk_create(buffer['detections'], batch_size=50)
                    buffer = {'frames': [], 'detections': []}
```

**Benefits**:
- Reduces DB round-trips from 18K to ~180 per video
- ~100× throughput improvement
- Atomic batch transactions

**Implementation**:
- Instantiate in `process_video_upload()`
- Replace direct `frame.save()` with `buffer.add_frame(frame)`

---

#### 3. **Live Stream Capture Decoupling** (Medium Priority)
```python
# backend/apps/video_analysis/live_frame_queue.py (NEW)
from threading import Thread
from queue import Queue
import logging

class LiveStreamFrameCapture:
    def __init__(self, stream_url: str, max_queue_size: int = 5):
        self.stream_url = stream_url
        self.frame_queue = Queue(maxsize=max_queue_size)
        self.capture_thread = Thread(target=self._capture_loop, daemon=True)
    
    def start(self):
        self.capture_thread.start()
    
    def _capture_loop(self):
        """Background thread: RTSP polling loop (I/O-bound)"""
        import cv2
        cap = cv2.VideoCapture(self.stream_url)
        while True:
            ret, frame = cap.read()
            if ret:
                try:
                    self.frame_queue.put_nowait(frame)
                except:
                    # Queue full; drop oldest (implements backpressure)
                    try:
                        self.frame_queue.get_nowait()
                        self.frame_queue.put_nowait(frame)
                    except:
                        pass
            else:
                # Stream reconnect logic
                cap.release()
                cap = cv2.VideoCapture(self.stream_url)
    
    def get_latest_frame(self, timeout_s: int = 1):
        return self.frame_queue.get(timeout=timeout_s)
```

**Benefits**:
- Decouples stream capture from inference scheduling
- Handles slow streams gracefully (backpressure)
- Allows multiple streams in parallel (if Celery workers support)

**Implementation**:
- Instantiate in `run_live_stream_inference()`
- Main loop calls `get_latest_frame()` instead of blocking capture

---

#### 4. **WebSocket Broadcast Queue** (Low Priority)
```python
# backend/apps/video_analysis/ws_broadcast_queue.py (NEW)
from threading import Thread
from queue import Queue
from asgiref.sync import async_to_sync
from django.core.cache import cache

class WSBroadcastQueue:
    def __init__(self):
        self.broadcast_queue = Queue()
        self.thread = Thread(target=self._broadcast_loop, daemon=True)
    
    def queue_detection_event(self, session_id: str, detection: dict):
        self.broadcast_queue.put(('detection', session_id, detection))
    
    def _broadcast_loop(self):
        """Background thread broadcasts WebSocket events"""
        from channels.layers import get_channel_layer
        channel_layer = get_channel_layer()
        while True:
            try:
                event_type, session_id, payload = self.broadcast_queue.get(timeout=1)
                async_to_sync(channel_layer.group_send)(
                    f"session_{session_id}",
                    {"type": event_type, "data": payload}
                )
            except:
                pass
```

**Benefits**:
- Non-blocking detection → broadcast path
- Reduces inference task latency

---

### ✅ For CPU-Bound Operations (Multi-Processing)

Use when:
- **CPU-intensive work dominates** (no GIL contention)
- **True parallelism needed** (not just concurrency)
- **Linux production** deployment (can use `fork()`)
- **Windows trade-off**: accept overhead of IPC for isolation

#### 1. **Parallel Multi-Model Inference** (High Priority)
**Location**: `backend/apps/pipeline/multi_model.py`

**Problem**: Models run sequentially; 80–90% of offline video time spent here.

**Solution A: Process Pool (Recommended for Linux Production)**
```python
# backend/apps/pipeline/multi_model_parallel.py (NEW)
from multiprocessing import Pool, Manager
from functools import partial
import logging

def _run_single_model_worker(
    toggle_name: str,
    family: str,
    role: str,
    label_map: dict,
    video_path: str,
    enabled_models: list,
    predict_kwargs: dict,
    return_dict: dict,  # Shared dict for results
):
    """Worker process: runs one model independently"""
    try:
        from apps.pipeline.multi_model import (
            _load_yolo_model,
            _build_predict_kwargs,
            MODEL_COLORS,
            FrameDetections,
            DetectionBox,
        )
        model, active_runtime, active_device = _load_yolo_model(family, role)
        if model is None:
            return
        
        color = MODEL_COLORS.get(toggle_name, '#22D3EE')
        results = model.predict(**predict_kwargs)
        
        frame_detections = {}
        for frame_idx, result in enumerate(results, 1):
            # ... extraction logic from original code ...
            frame_detections[frame_idx] = detected_boxes
        
        return_dict[toggle_name] = frame_detections
    except Exception as e:
        logging.exception(f"Model {toggle_name} failed: {e}")
        return_dict[toggle_name] = {}

def run_multi_model_inference_parallel(
    video_path: str,
    enabled_models: list[str],
    project_dir: str,
    max_processes: int = 4,
) -> dict[int, 'FrameDetections']:
    """Run enabled models in parallel using process pool"""
    
    manager = Manager()
    return_dict = manager.dict()
    
    # Prepare jobs
    jobs = []
    for toggle_name, family, role, label_map in MODEL_REGISTRY:
        if toggle_name not in enabled_models:
            continue
        
        predict_kwargs = _build_predict_kwargs(video_path, ...)
        jobs.append((
            toggle_name, family, role, label_map,
            video_path, enabled_models, predict_kwargs, return_dict
        ))
    
    # Run models in parallel
    with Pool(processes=min(max_processes, len(jobs))) as pool:
        worker = partial(
            _run_single_model_worker,
            return_dict=return_dict
        )
        pool.starmap(worker, jobs)
    
    # Merge results
    all_frames = {}
    for toggle_name, frame_detections in return_dict.items():
        for frame_idx, boxes in frame_detections.items():
            if frame_idx not in all_frames:
                all_frames[frame_idx] = FrameDetections(frame_index=frame_idx, ...)
            all_frames[frame_idx].boxes.extend(boxes)
    
    return all_frames
```

**Benefits**:
- **4–6× speedup** (one process per model family)
- **Linux production**: Uses `fork()`, full parallelism
- **Windows dev**: Processes share memory via `Manager().dict()`, slower but functional
- Removes OpenVINO lock contention

**Trade-off**:
- IPC overhead (serializing frame data to workers)
- Higher memory (each process loads models independently)
- Mitigation: pre-load models once, fork, then inference

**Implementation**:
- Add `run_multi_model_inference_parallel()` function
- Update `process_video_upload()` to use parallel version on Linux, sequential on Windows
- Configurable via `PYRAMID_PARALLEL_MODELS=true` env var

---

#### 2. **ReID Embedding Batch Processing** (High Priority)
**Location**: `backend/apps/tracking/embeddings.py`

**Problem**: Per-frame embedding computation is sequential, blocking.

**Solution: Batch Processing with Threading**
```python
# backend/apps/tracking/embeddings_batch.py (NEW)
from concurrent.futures import ThreadPoolExecutor
import numpy as np
from typing import List, Tuple

class EmbeddingBatchProcessor:
    def __init__(self, max_batch_size: int = 32, num_workers: int = 2):
        self.max_batch_size = max_batch_size
        self.executor = ThreadPoolExecutor(max_workers=num_workers)
        self.pending_crops = []
    
    def add_crop(self, track_id: int, crop_image: np.ndarray):
        """Queue a crop for embedding"""
        self.pending_crops.append((track_id, crop_image))
        
        if len(self.pending_crops) >= self.max_batch_size:
            self.flush()
    
    def flush(self):
        """Process pending crops as a batch"""
        if not self.pending_crops:
            return
        
        crops = [crop for _, crop in self.pending_crops]
        track_ids = [tid for tid, _ in self.pending_crops]
        
        # Batch embedding computation (GPU-friendly)
        embeddings = self._compute_batch_embeddings(crops)
        
        # Cache results
        for track_id, embedding in zip(track_ids, embeddings):
            cache_job_track_embedding(track_id, embedding)
        
        self.pending_crops = []
    
    def _compute_batch_embeddings(self, crops: List[np.ndarray]) -> List[np.ndarray]:
        """Compute embeddings for a batch of crops (thread-safe)"""
        # Use model that supports batch inference
        embeddings = []
        for crop in crops:
            emb = generate_embedding_vector(crop)
            embeddings.append(normalize_vector(emb))
        return embeddings
```

**Benefits**:
- Reduces overhead from 30 individual inferences to ~1 batch call per frame
- Thread-safe (GIL-safe in Python)
- Works on Windows + Linux

**Implementation**:
- Replace per-crop embedding calls in `process_frame()` with batch processor
- Flush at end of frame batch or timeout

---

#### 3. **Parallel Video Encoding** (Medium Priority)
**Location**: `backend/apps/tracking/video_exporter.py`

**Problem**: Single FFmpeg process encodes sequentially.

**Solution: Multi-Pass with Thread Pool**
```python
# Use existing `PYRAMID_OUTPUT_VIDEO_PRESET` but add threading
from concurrent.futures import ThreadPoolExecutor

def render_annotated_video_threaded(
    job: VideoAnalysisJob,
    visible_models: list[str] | None = None,
    num_threads: int = 2,  # FFmpeg handles internal threading
):
    """Render annotated video with optional frame-parallel preparation"""
    
    # Note: FFmpeg's -threads option already parallelizes encoding
    # Here we optimize pre-processing (frame annotation)
    
    with ThreadPoolExecutor(max_workers=num_threads) as executor:
        # Submit annotation tasks for future frames
        futures = []
        for frame_idx in range(total_frames):
            future = executor.submit(
                _annotate_single_frame,
                job, frame_idx, visible_models
            )
            futures.append(future)
        
        # Consume results in order and pipe to FFmpeg
        for future in futures:
            annotated_frame = future.result()
            yield annotated_frame  # Pipe to encoder
```

**Benefits**:
- Overlap annotation with encoding
- Leverages FFmpeg's internal threading

**Implementation**:
- Add `-threads N` to FFmpeg command if not present
- Wrap frame annotation loop with thread pool

---

## Windows → Linux Deployment Recommendations

### ✅ Development (Windows)

**Constraints**:
- Celery `solo` pool (no multiprocessing)
- `fork()` not available
- Multi-threaded code must be GIL-aware

**Strategy**:
1. **Use threading for I/O**:
   - Frame reading pipeline ✅
   - DB write buffer ✅
   - WebSocket broadcast queue ✅

2. **Avoid multi-process inference on Windows**:
   - Keep sequential multi-model (acceptable for dev)
   - Use local inference adapters only (OpenVINO/TensorRT on GPU)
   - Test Triton integration separately

3. **Configuration**:
   ```env
   CELERY_WORKER_POOL=solo
   CELERY_WORKER_CONCURRENCY=1
   PYRAMID_PARALLEL_MODELS=false  # Skip parallel models on Windows
   ```

### ✅ Production (Linux)

**Advantages**:
- Celery `prefork` pool (true multi-process)
- `fork()` available → efficient process creation
- RTX 5090 GPU for inference (Triton only)

**Strategy**:
1. **Use multi-processing for CPU-bound**:
   - Parallel multi-model inference ✅ (4–6 processes)
   - ReID embedding batch processing ✅
   - Encoding pre-processing ✅

2. **Use threading for I/O**:
   - Frame reading pipeline ✅
   - DB write buffer ✅
   - Live stream capture ✅
   - WebSocket broadcast ✅

3. **Configuration**:
   ```env
   CELERY_WORKER_POOL=prefork
   CELERY_WORKER_CONCURRENCY=6  # RTX 5090: tune to GPU memory
   CELERY_GPU_CONCURRENCY_CAP=2  # Cap GPU-bound tasks
   PYRAMID_PARALLEL_MODELS=true
   PYRAMID_WORKER_COUNT=4  # Parallel model processes
   ```

4. **Triton Integration**:
   - **Production**: Triton-only (no local adapters)
   - **Inference**: gRPC to Triton (faster than HTTP, batching support)
   - **Batch size**: Tune via Triton model config

---

## Implementation Roadmap

### Phase 1: Low-Risk, High-Impact (Weeks 1–2)

- [ ] **Frame Reading Pipeline** (Threading)
  - `backend/apps/video_analysis/frame_pipeline.py`
  - Update `process_video_upload()` to use pipeline
  - Benchmark: measure speedup

- [ ] **DB Write Buffer** (Threading)
  - `backend/apps/video_analysis/db_buffer.py`
  - Batch Detection, Frame, StudentTrack writes
  - Benchmark: measure DB query reduction

### Phase 2: High-Impact, Medium-Complexity (Weeks 3–4)

- [ ] **Parallel Multi-Model Inference** (Multi-Processing)
  - `backend/apps/pipeline/multi_model_parallel.py`
  - Test on Linux development machine
  - Graceful fallback to sequential on Windows

- [ ] **ReID Embedding Batch** (Threading)
  - `backend/apps/tracking/embeddings_batch.py`
  - Integrate with `process_frame()`

### Phase 3: Polish & Optimization (Weeks 5–6)

- [ ] **Live Stream Capture Decoupling** (Threading)
  - Handle multiple cameras in parallel Celery workers

- [ ] **WebSocket Broadcast Queue** (Threading)
  - Non-blocking broadcast path

- [ ] **Production Tuning**:
  - Load test with realistic camera counts
  - Measure end-to-end latency
  - Adjust concurrency settings

---

## Testing Strategy

### Unit Tests
- Mock I/O operations (frame reads, DB writes)
- Verify buffer batch sizes and flush timing
- Validate embedding batch correctness

### Integration Tests
- Process sample videos (offline)
- Measure end-to-end time improvements
- Verify frame/detection consistency

### Load Tests
```bash
# 5x sample videos in parallel
k6 run load-tests/parallel_uploads.js --vus 5 --duration 10m

# Live stream simulation (10 concurrent streams)
python scripts/load_tests/live_camera_simulator.py --num_cameras 10
```

### Windows-Specific Tests
- Verify threading doesn't deadlock on Windows file locks
- Test solo pool with concurrent uploads

### Linux Production Tests
- Verify prefork pool with 6 workers
- Profile GPU memory usage (RTX 5090 constraint)
- Stress test Triton connection pool

---

## Risk Assessment

| Component | Risk | Mitigation |
|-----------|------|-----------|
| **Threading (I/O)** | GIL under CPU load | Batch operations; monitor latency |
| **Multi-Process (inference)** | Memory overhead | Pre-load models; tunable process count |
| **DB Batching** | Transaction atomicity | Use `bulk_create()` with transaction scope |
| **Windows Fallback** | Sequential bottleneck in dev | Warn developers; skip parallel models on Windows |
| **Triton Integration** | Connection pool exhaustion | Monitor connection count; configure timeouts |

---

## Success Criteria

✅ **Offline video processing**: 40–50% speedup (frame pipeline + DB batching + parallel models)  
✅ **Live streaming**: Handle 10+ concurrent streams without frame drops  
✅ **ReID latency**: <5ms per-frame embedding on RTX 5090  
✅ **Windows dev**: No regressions; graceful fallback to sequential  
✅ **Linux production**: Full parallelism; <5s overhead for setup  
✅ **Database**: <100ms for 18K-frame video writes (from ~10s)  

---

## References

- `agents.md`: Production deployment topology & constraints
- `backend/config/celery.py`: Current Celery configuration
- `backend/apps/pipeline/multi_model.py`: OpenVINO lock implementation
- `backend/apps/video_analysis/tasks.py`: Main offline processing task
- `backend/apps/tracking/video_exporter.py`: Video encoding pipeline
- `docs/heterogeneous_production_runtime_maturity_plan.md`: Production runtime policy (Triton-only)
