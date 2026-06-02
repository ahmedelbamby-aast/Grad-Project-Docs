# System Architecture Overview

**Last updated:** 2026-05-25

This page is the architecture spine for the docs set. It reflects the implementation in `backend/`, `frontend/`, `infra/`, `docker-compose.dev.yml`, and `nginx.conf`, then links into deeper module-level docs.

## High-Level Runtime Topology

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    subgraph Clients["Clients"]
        SPA["React SPA (frontend/src)"]
        APIClient["External API clients"]
    end

    subgraph Edge["Edge & Gateway"]
        Nginx["nginx.conf"]
        Go2rtc["go2rtc relay"]
        GstGateway["gst_mediamtx gateway (camera service abstraction)"]
        Cameras["RTSP/ONVIF cameras"]
    end

    subgraph App["Backend Runtime"]
        DRF["Django DRF APIs"]
        Channels["Django Channels WebSockets"]
        Celery["Celery workers"]
        Beat["Celery Beat"]
        RuntimePolicy["pipeline/services/runtime_policy.py"]
        RuntimeIngestion["pipeline/services/runtime_ingestion.py"]
        RuntimeAnalytics["pipeline/views_runtime_analytics.py"]
    end

    subgraph AI["Inference & Tracking"]
        Route["model_route_service.py"]
        Triton["Triton Inference Server"]
        LocalInference["Local adapters (non-production only)"]
        Tracking["tracking pipeline + ReID + video exporter"]
    end

    subgraph State["State & Storage"]
        Postgres[("PostgreSQL")]
        Redis[("Redis (broker/results/channel layer/cache)")]
        Files["Media files + artifacts"]
    end

    SPA -->|/api/v1| Nginx
    SPA -->|/ws| Nginx
    SPA -->|/whep/camera_id| Nginx
    APIClient -->|HTTPS| Nginx

    Nginx --> DRF
    Nginx --> Channels
    Nginx --> Go2rtc

    Go2rtc --> Cameras
    GstGateway --> Cameras
    DRF --> GstGateway
    DRF --> Go2rtc

    DRF --> Postgres
    DRF --> Redis
    DRF --> Celery
    Channels --> Redis
    Beat --> Redis
    Celery --> Redis
    Celery --> Postgres
    Celery --> Files

    Celery --> RuntimePolicy
    RuntimePolicy --> Route
    RuntimePolicy --> RuntimeIngestion
    Route --> Triton
    Route --> LocalInference
    Triton --> Tracking
    LocalInference --> Tracking
    Tracking --> Postgres
    Tracking --> Redis
    Tracking --> Files

    RuntimeIngestion --> Postgres
    RuntimeAnalytics --> Postgres
    RuntimeAnalytics --> DRF
```

## End-to-End Execution Modes

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart LR
    Upload["Offline upload POST /api/v1/video-analysis/jobs/"] --> Job["VideoAnalysisJob"]
    Job --> Queue["process_video_upload (Celery)"]
    Queue --> Policy["runtime policy resolution"]
    Policy --> Infer["Active Triton profile in production"]
    Infer --> Track["tracking + ReID + overlays"]
    Track --> Persist["Frame/Detection/Track rows + output media + WS updates"]

    Live["Live session start API"] --> Session["MonitoringSession + stream registration"]
    Session --> LiveTask["run_live_stream_inference (Celery)"]
    LiveTask --> LivePolicy["runtime policy + provider health"]
    LivePolicy --> LiveInfer["frame inference"]
    LiveInfer --> LiveTrack["tracking + anomaly signals"]
    LiveTrack --> LiveWs["detection/anomaly websocket events"]
```

Upload processing supports two runtime paths:
- `legacy_crop`: historical crop-first pipeline and primary tracking flow.
- `full_frame`: full-frame multi-model inference with shared detection packet schema and optional annotated export output.

These pipeline paths describe preprocessing and artifact behavior, not
production inference-provider choice. Production inference authority is native
Linux Triton-only, with exactly one active endpoint profile at a time. Local
adapter paths in the implementation are limited to development or explicitly
non-production validation and cannot satisfy production evidence gates.

## Deployment Boundary

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart LR
    DevCompose["docker-compose.dev.yml"] --> DevRedis["redis container"]
    DevCompose --> DevPG["postgres container"]
    DevCompose --> DevGo2rtc["go2rtc container"]
    DevCompose --> DevTriton["tritonserver container (optional profile)"]

    ProdHost["Production Linux host"] --> Systemd["infra/systemd/triton-server.service"]
    Systemd --> ProdTriton["one active native Triton profile"]
    ProdBackend["backend services"] --> ProdTriton
```

Production runs do not depend on Docker or sudo. The live endpoint profile
uses `39000/39001/39002`; the offline endpoint profile uses
`39100/39101/39102`. Readiness acceptance requires the selected profile to be
healthy and the inactive profile to be unreachable.

## Domain Boundaries

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart LR
    Accounts["accounts"] --> Exams["exams"]
    Exams --> Cameras["cameras"]
    Cameras --> Sessions["sessions"]
    Sessions --> Detections["detections"]
    Detections --> Anomalies["anomalies"]
    Sessions --> Recordings["recordings"]
    Sessions --> Exports["exports"]
    VideoAnalysis["video_analysis"] --> Pipeline["pipeline"]
    Pipeline --> Tracking["tracking"]
    Health["health"] --> Pipeline
    Audit["audit"] --> Accounts
```

## Related Deep Dives

- [Data Flow](backend/architecture/data-flow.md)
- [Deployment Topology](backend/architecture/deployment-topology.md)
- [Observability Runbook](backend/architecture/observability-runbook.md)
- [Triton Operations](backend/architecture/triton-operations.md)
- [System Mermaid Atlas](diagrams/SYSTEM_MERMAID_ATLAS.md)
