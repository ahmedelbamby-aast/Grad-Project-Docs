# Data Flow — API, Streaming, Inference, and Runtime Telemetry

**Updated**: 2026-05-15

## Scope

This document maps implementation-backed flows across:
1. HTTP + WebSocket request paths
2. Live streaming inference
3. Offline video analysis inference
4. Runtime telemetry ingestion and analytics APIs

---

## 1. Edge Routing and Protocol Split

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart LR
    Browser["Browser / frontend"] --> Nginx["nginx.conf"]
    Nginx -->|/api/v1/*| DRF["Django DRF views"]
    Nginx -->|/ws/*| Channels["Django Channels consumers"]
    Nginx -->|/whep/camera_id| Whep["go2rtc WHEP endpoint"]
    DRF --> Postgres[("PostgreSQL")]
    DRF --> Redis[("Redis")]
    Channels --> Redis
```

---

## 2. Offline Upload Inference Pipeline

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
sequenceDiagram
    actor User
    participant FE as frontend videoAnalysis API
    participant View as video_analysis/views.py
    participant DB as PostgreSQL
    participant Task as process_video_upload (Celery)
    participant Policy as runtime_policy.py
    participant Route as model_route_service.py
    participant Triton as triton_client.py
    participant Local as local multi-model runtime
    participant Track as tracking pipeline
    participant WS as VideoAnalysisConsumer

    User->>FE: upload video + options
    FE->>View: POST /api/v1/video-analysis/jobs/
    View->>DB: create VideoAnalysisJob
    View->>Task: enqueue background processing
    Task->>Policy: resolve runtime mode
    Policy->>Route: map task to model route
    alt Triton enabled and healthy
        Route->>Triton: infer(frame)
        Triton-->>Task: TritonInferenceResponse
    else local fallback or policy local
        Route->>Local: run local inference
        Local-->>Task: frame detections
    end
    Task->>Track: assign IDs/colors/ReID and render outputs
    Task->>DB: persist Frame/Detection/StudentTrack/BoundingBox
    Task-->>WS: progress + overlay.frame events
```

---

## 3. Live Stream Inference Pipeline

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    SessionStart["POST /api/v1/sessions/"] --> SessionRow["MonitoringSession row"]
    SessionRow --> LiveTask["run_live_stream_inference"]
    LiveTask --> CameraProvider["camera gateway provider selection"]
    CameraProvider --> Go2rtc["go2rtc"]
    CameraProvider --> Gst["gst_mediamtx path"]
    Go2rtc --> Frames["frame stream"]
    Gst --> Frames
    Frames --> Policy["runtime_policy"]
    Policy --> TritonPath["Triton path"]
    Policy --> LocalPath["Local path"]
    TritonPath --> Tracking["tracking + overlays + events"]
    LocalPath --> Tracking
    Tracking --> Persist["DetectionFrame/LiveDetection/Prediction rows"]
    Tracking --> DetWS["/ws/detections/*"]
    Tracking --> AnoWS["/ws/anomalies/*"]
```

---

## 4. Runtime Telemetry Flow

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart LR
    Workers["video_analysis and pipeline services"] --> Ingest["runtime_ingestion.py"]
    Ingest --> EventTbl["FrameLifecycleEvent"]
    Ingest --> MetricTbl["ModelExecutionMetric"]
    Ingest --> HealthTbl["ServiceHealthSnapshot"]
    EventTbl --> Analytics["views_runtime_analytics.py"]
    MetricTbl --> Analytics
    HealthTbl --> Analytics
    Analytics --> API["/api/v1/runtime/* endpoints"]
    API --> FE["runtime dashboards and operator tools"]
```

---

## 5. Flow Invariants

- Live and offline workloads share runtime policy + route services, but use distinct entrypoints and persistence models.
- Triton path is optional at runtime; local inference remains a first-class path.
- Redis is used for queueing/channel-layer/cache state; PostgreSQL is the durable source of truth.
- Runtime analytics APIs read telemetry snapshots/events and do not mutate inference state.

## Related Documents

- [ARCHITECTURE.md](../../ARCHITECTURE.md)
- [deployment-topology.md](deployment-topology.md)
- [observability-runbook.md](observability-runbook.md)
- [triton-operations.md](triton-operations.md)
