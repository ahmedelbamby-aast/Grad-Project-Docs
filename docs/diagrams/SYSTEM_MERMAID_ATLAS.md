# System Mermaid Atlas

This atlas mirrors the implementation as it exists in the repository. It focuses on project-owned source, tests, scripts, infrastructure, and documentation flows. Generated caches, virtual environments, node modules, compiled Python bytecode, and built assets are intentionally excluded.

## Source Coverage Map

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
mindmap
  root((grad_project))
    backend
      config["ASGI, URLs, Celery, settings packages"]
      core["shared pagination, logging, exceptions, middleware"]
      apps
        accounts["auth, roles, permissions"]
        exams["rooms, exams, rosters"]
        cameras["camera sources, go2rtc registration, camera WS"]
        sessions["monitoring sessions, session WS, comments"]
        detections["live detection frames and predictions"]
        anomalies["events, triage, alert WS"]
        recordings["recording lifecycle and playback metadata"]
        exports["session ZIP export tasks"]
        audit["immutable audit middleware and services"]
        health["DB, Redis, storage, model-serving health"]
        pipeline["YOLO/OpenVINO/ONNX/Triton inference"]
        tracking["track IDs, colors, ReID, rendering, video export"]
        video_analysis["offline upload jobs, frame overlays, job WS"]
      tests["unit, contract, integration, system"]
    frontend
      src
        api["Axios modules per backend boundary"]
        pages["route-level user workflows"]
        components["layout, camera, anomaly, recording, video, UI"]
        hooks["WebSocket, WHEP, auth, health, upload live state"]
        stores["Zustand domain stores"]
        types["REST and WS contracts"]
        styles["tokens and themes"]
      tests["unit, integration, e2e"]
    infra
      systemd["native Triton production service"]
    scripts
      ci["documentation, boundary, release verifiers"]
      triton["PowerShell model repository helpers"]
    specs
      "006-modular-low-coupling"
```

## Runtime Container Flow

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    Browser["Browser: React SPA"] --> Nginx["Nginx reverse proxy"]
    Browser --> Vite["Vite dev server"]
    Nginx --> DjangoHTTP["Django HTTP: DRF views"]
    Nginx --> DjangoWS["Django ASGI: Channels consumers"]
    Nginx --> Go2RTC["go2rtc WHEP/WebRTC"]
    Nginx --> WHEP["/whep/{camera_id} auth-gated relay route"]
    DjangoHTTP --> Postgres["PostgreSQL"]
    DjangoHTTP --> Redis["Redis cache/broker/channel layer"]
    DjangoWS --> Redis
    DjangoHTTP --> Celery["Celery worker"]
    Celery --> Redis
    Celery --> Postgres
    Celery --> Pipeline["apps.pipeline services"]
    Celery --> Tracking["apps.tracking services"]
    Celery --> RuntimeIngest["runtime_ingestion.py"]
    RuntimeIngest --> RuntimeTables["FrameLifecycleEvent + ModelExecutionMetric + ServiceHealthSnapshot"]
    DjangoHTTP --> RuntimeAPI["views_runtime_analytics.py"]
    RuntimeAPI --> RuntimeTables
    Pipeline --> Triton["Triton Inference Server"]
    Pipeline --> LocalRuntime["Local YOLO/OpenVINO/ONNX runtime"]
    DjangoHTTP --> Gateway["camera provider abstraction (go2rtc / gst_mediamtx)"]
    Gateway --> Go2RTC
    Gateway --> GstGateway["gst_mediamtx gateway"]
    Go2RTC --> Camera["RTSP camera source"]
    GstGateway --> Camera
```

## Backend Module Boundaries

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart LR
    accounts["accounts"] --> audit["audit"]
    exams["exams"] --> accounts
    cameras["cameras"] --> exams
    sessions["sessions"] --> accounts
    sessions --> exams
    sessions --> cameras
    detections["detections"] --> sessions
    detections --> cameras
    anomalies["anomalies"] --> detections
    anomalies --> cameras
    recordings["recordings"] --> sessions
    recordings --> cameras
    exports["exports"] --> sessions
    health["health"] --> pipeline["pipeline"]
    video_analysis["video_analysis"] --> pipeline
    video_analysis --> tracking["tracking"]
    video_analysis --> health
    tracking --> video_analysis
    pipeline --> tracking
```

## Frontend Route And File Flow

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TD
    main["src/main.tsx"] --> app["src/App.tsx"]
    app --> guards["components/auth/RouteGuards.tsx"]
    guards --> layout["components/layout/AppLayout.tsx"]
    layout --> pages["src/pages/*.tsx"]
    pages --> api["src/api/*.ts"]
    pages --> hooks["src/hooks/*.ts"]
    pages --> stores["src/stores/*.ts"]
    hooks --> ws["useWebSocket.ts"]
    hooks --> whep["useWhepClient.ts"]
    api --> client["api/client.ts"]
    client --> rest["/api/v1/*"]
    ws --> wsroutes["/ws/*"]
    whep --> go2rtc["/whep/*"]
    stores --> components["src/components/**/*.tsx"]
    types["src/types/*.ts"] --> api
    types --> hooks
    styles["src/styles/*.css"] --> components
```

## User Journey

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
journey
    title Instructor Monitoring Journey
    section Login
      Open login page: 3: Instructor
      Submit credentials through authApi: 4: Instructor
      ProtectedRoute loads dashboard: 4: System
    section Prepare
      Review dashboard: 3: Instructor
      Add or connect camera: 3: Instructor
      Start monitoring session: 4: Instructor
    section Monitor
      Watch WHEP live feed: 4: Instructor
      Receive detection overlays: 4: System
      Receive anomaly alerts: 3: System
      Acknowledge or dismiss alert: 4: Instructor
    section Review
      Open recordings or session detail: 3: Instructor
      Export evidence bundle: 4: Instructor
```

## Offline Video Upload Sequence

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
sequenceDiagram
    actor User
    participant Page as VideoAnalysisPage.tsx
    participant API as api/videoAnalysis.ts
    participant View as video_analysis/views.py
    participant Task as video_analysis/tasks.py
    participant Pipeline as apps.pipeline
    participant Triton as TritonClient
    participant Tracking as apps.tracking
    participant DB as PostgreSQL
    participant WS as VideoAnalysisConsumer

    User->>Page: select video and options
    Page->>API: upload(file, pipelineMode, showTrackingIds)
    API->>View: POST /api/v1/video-analysis/jobs/
    View->>DB: create VideoAnalysisJob
    View->>Task: enqueue process_video_upload
    Task->>Pipeline: resolve runtime policy and enabled models
    alt Triton enabled and healthy
        Task->>Triton: infer frame/model routes
    else local fallback
        Task->>Pipeline: run local multi-model inference
    end
    Task->>Tracking: assign IDs, colors, embeddings, export video
    Task->>DB: persist Frame, Detection, StudentTrack, BoundingBox
    Task->>WS: broadcast progress and overlay.frame
    WS->>Page: live job frames and progress
    Page->>User: stable overlays, tracking IDs, rendered video
```

## Live Streaming Sequence

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
sequenceDiagram
    actor User
    participant Page as CameraFeedPage.tsx
    participant SessionAPI as api/sessions.ts
    participant CameraAPI as api/cameras.ts
    participant WHEP as useWhepClient.ts
    participant DetectWS as useDetectionSocket.ts
    participant SessionView as sessions/views.py
    participant SessionTask as video_analysis/tasks.py
    participant Pipeline as apps.pipeline
    participant Tracking as apps.tracking
    participant Go2RTC as go2rtc
    participant Gateway as cameras/services.py provider
    participant DB as PostgreSQL

    User->>Page: connect cameras and start session
    Page->>CameraAPI: list/connect cameras
    Page->>SessionAPI: create session with camera_ids and overlay options
    SessionAPI->>SessionView: POST /api/v1/sessions/
    SessionView->>DB: create MonitoringSession
    SessionView->>SessionTask: enqueue run_live_stream_inference
    SessionTask->>Gateway: resolve gateway provider and stream endpoint
    Page->>WHEP: open /whep/{stream}
    WHEP->>Go2RTC: negotiate WebRTC media
    Page->>DetectWS: subscribe /ws/detections/{session}/
    SessionTask->>Pipeline: infer per frame
    Pipeline->>Tracking: associate tracks and render boxes
    SessionTask->>DB: persist DetectionFrame and Prediction
    SessionTask-->>DetectWS: detection.frame and prediction.update
    DetectWS-->>Page: update stores and overlays
```

## Database Entity Relationships

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
erDiagram
    USER ||--o{ ROLE : has
    USER ||--o{ ROOM : creates
    USER ||--o{ EXAM : creates
    ROOM ||--o{ EXAM : hosts
    EXAM ||--o{ EXAM_STUDENT : enrolls
    EXAM ||--o{ MONITORING_SESSION : observed_by
    CAMERA_SOURCE ||--|| STREAM_REGISTRATION : registers
    CAMERA_SOURCE ||--o{ CONNECTION_EVENT : emits
    CAMERA_SOURCE ||--o{ DETECTION_FRAME : produces
    MONITORING_SESSION ||--o{ DETECTION_FRAME : groups
    DETECTION_FRAME ||--o{ LIVE_DETECTION : contains
    LIVE_DETECTION ||--|| PYRAMID_PREDICTION : classifies
    LIVE_DETECTION ||--o{ ANOMALY_EVENT : may_trigger
    ANOMALY_EVENT ||--o{ ANOMALY_NOTE : has
    MONITORING_SESSION ||--o{ RECORDING : references
    MONITORING_SESSION ||--o{ SESSION_EXPORT : exports
    VIDEO_ANALYSIS_JOB ||--o{ FRAME : contains
    FRAME ||--o{ OFFLINE_DETECTION : contains
    VIDEO_ANALYSIS_JOB ||--o{ STUDENT_TRACK : owns
    STUDENT_TRACK ||--o{ BOUNDING_BOX : colors
    OFFLINE_DETECTION ||--o{ BOUNDING_BOX : renders
    OFFLINE_DETECTION ||--o{ FRAME_EMBEDDING : embeds
```

## Offline Job State Machine

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
stateDiagram-v2
    [*] --> queued
    queued --> processing: Celery worker starts
    processing --> embedding: ReID/frame embedding stage
    embedding --> completed: annotated video and results ready
    processing --> completed: no embedding stage required
    queued --> failed: validation error
    processing --> failed: inference/persistence/export error
    embedding --> failed: embedding error
    completed --> [*]
    failed --> [*]
```

## Video Processing Data Flow

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart LR
    Upload["Multipart upload"] --> Job["VideoAnalysisJob"]
    Job --> Runtime["runtime_policy.py"]
    Runtime --> Route["model_route_service.py"]
    Route --> TritonClient["triton_client.py"]
    Route --> Local["multi_model.py local fallback"]
    TritonClient --> Orchestrator["video_analysis/services/inference_orchestrator.py"]
    Local --> FrameDetections["FrameDetections"]
    Orchestrator --> FrameDetections
    FrameDetections --> TrackIDs["tracking/tracker.py and tracking/pipeline.py"]
    TrackIDs --> Colors["tracking/colors.py"]
    TrackIDs --> ReID["tracking/reid.py + embeddings.py"]
    Colors --> StudentTrack["StudentTrack rows"]
    FrameDetections --> Boxes["BoundingBox rows"]
    ReID --> Embeddings["FrameEmbedding rows"]
    Boxes --> Exporter["tracking/video_exporter.py"]
    Exporter --> Media["annotated video file"]
    Boxes --> WebSocket["overlay.frame broadcasts"]
```

## Frontend Offline Overlay Flow

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TD
    VideoPage["VideoAnalysisPage.tsx"] --> UploadStore["stores/uploadStore.ts"]
    VideoPage --> VisibilityStore["stores/visibilityStore.ts"]
    VideoPage --> LiveHook["hooks/useVideoAnalysisLive.ts"]
    VideoPage --> UploadTab["components/VideoUploadTab/VideoUploadTab.tsx"]
    UploadTab --> VideoAPI["api/videoAnalysis.ts"]
    LiveHook --> WatchJob["videoAnalysisApi.watchJob"]
    WatchJob --> JobWS["/ws/video-analysis/jobs/{jobId}/"]
    LiveHook --> CurrentFrame["currentFrame with throttled overlay.frame updates"]
    CurrentFrame --> Player["components/VideoPlayer/VideoPlayer.tsx"]
    VisibilityStore --> Toggles["ModelVisibilityToggles.tsx"]
    Toggles --> VideoAPI
    VideoAPI --> Backend["video_analysis/views.py"]
```

## WebSocket Contract Map

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart LR
    useWebSocket["hooks/useWebSocket.ts"] --> CameraStatus["useCameraStatus.ts"]
    useWebSocket --> DetectionSocket["useDetectionSocket.ts"]
    useWebSocket --> AnomalySocket["useAnomalySocket.ts"]
    useWebSocket --> HealthSocket["useHealthSocket.ts"]
    videoAnalysisAPI["api/videoAnalysis.ts watchJob"] --> VideoLive["useVideoAnalysisLive.ts"]
    CameraStatus --> CameraConsumer["cameras/consumers.py"]
    DetectionSocket --> DetectionConsumer["detections/consumers.py"]
    AnomalySocket --> AnomalyConsumer["anomalies/consumers.py"]
    HealthSocket --> HealthConsumer["health/consumers.py"]
    VideoLive --> VideoConsumer["video_analysis/consumers.py"]
    CameraConsumer --> Redis["Redis channel layer"]
    DetectionConsumer --> Redis
    AnomalyConsumer --> Redis
    HealthConsumer --> Redis
    VideoConsumer --> Redis
```

## Test Architecture

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    BackendTests["backend/tests"] --> UnitPy["unit: pure functions, services, model helpers"]
    BackendTests --> ContractPy["contract: API, WS, runtime, docs boundaries"]
    BackendTests --> IntegrationPy["integration: auth, sessions, cameras, upload, pipeline"]
    BackendTests --> SystemPy["system: real video, Triton, load, live/offline equivalence"]
    FrontendTests["frontend/tests"] --> UnitTs["unit: components, hooks, stores, API contracts"]
    FrontendTests --> IntegrationTs["integration: theme, modular performance"]
    FrontendTests --> E2E["e2e: login, camera feed, anomaly triage, recording playback"]
    Scripts["scripts/ci"] --> DocsCheck["verify_docs_diagrams.py"]
    Scripts --> BoundaryCheck["verify_module_boundaries.py"]
    Scripts --> ReleaseGate["verify_release_gate.py"]
    ContractPy --> DocsCheck
    E2E --> BrowserFlows["real user workflows"]
```

## Script And Infrastructure Flow

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart LR
    Env[".env / .env.example"] --> DockerCompose["docker-compose.dev.yml"]
    DockerCompose --> Postgres["Postgres dev"]
    DockerCompose --> Redis["Redis dev"]
    DockerCompose --> Go2RTC["go2rtc dev"]
    Env --> BackendSettings["backend config settings"]
    Env --> FrontendEnv["Vite env variables"]
    TritonPS["scripts/*triton*.ps1"] --> ModelRepo["Triton model repository"]
    TensorRTPS["scripts/export-tensorrt-models.ps1"] --> TensorRT["TensorRT artifacts"]
    Systemd["infra/systemd/triton-server.service"] --> NativeTriton["native Linux Triton"]
    CI["scripts/ci/*.py"] --> Docs["docs/**/*.md"]
    CI --> Source["backend/apps + frontend/src"]
```

## Model Serving Decision Timeline

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
timeline
    title Inference Runtime Resolution
    Request enters pipeline : runtime_policy evaluates settings and feature flags
    Route selection : model_route_service maps task to model version and runtime
    Health check : Triton readiness and degradation state are checked
    Triton path : inference_orchestrator dispatches to TritonClient
    Local path : multi_model uses local YOLO/OpenVINO/ONNX fallback
    Persistence : tracking and video_analysis write stable frame and track data
```

## Dependency Risk Quadrants

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
quadrantChart
    title Coupling Risk View
    x-axis Low Runtime Blast Radius --> High Runtime Blast Radius
    y-axis Low Change Frequency --> High Change Frequency
    quadrant-1 Watch
    quadrant-2 Harden First
    quadrant-3 Keep Simple
    quadrant-4 Document Contract
    "frontend api/client": [0.7, 0.8]
    "video_analysis/tasks": [0.9, 0.9]
    "pipeline/services": [0.8, 0.7]
    "tracking/colors": [0.5, 0.5]
    "docs contracts": [0.3, 0.7]
    "infra/systemd": [0.8, 0.2]
```

## Requirement Coverage

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
requirementDiagram
    requirement LiveOfflineParity {
        id: R1
        text: "Live streaming and offline upload must share inference and tracking contracts."
        risk: high
        verifymethod: test
    }
    requirement StableTrackDisplay {
        id: R2
        text: "Tracking IDs and colors must remain stable under frame stride and load variation."
        risk: high
        verifymethod: test
    }
    requirement ModularBoundaries {
        id: R3
        text: "Domain modules expose behavior through documented boundaries."
        risk: medium
        verifymethod: inspection
    }
    requirement DiagramCoverage {
        id: R4
        text: "Documentation must include source, user, data, runtime, and cross-file flows."
        risk: medium
        verifymethod: analysis
    }
    functionalRequirement VideoUpload {
        id: F1
        text: "Upload video, process frames, persist detections, stream progress."
    }
    functionalRequirement LiveSession {
        id: F2
        text: "Start camera session, stream media, infer frames, broadcast detections."
    }
    LiveOfflineParity - satisfies -> VideoUpload
    LiveOfflineParity - satisfies -> LiveSession
    StableTrackDisplay - traces -> VideoUpload
    StableTrackDisplay - traces -> LiveSession
    DiagramCoverage - contains -> ModularBoundaries
```

## Git And Release Evidence Flow

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
gitGraph
    commit id: "baseline"
    branch "006-modular-low-coupling"
    checkout "006-modular-low-coupling"
    commit id: "contracts"
    commit id: "runtime hardening"
    commit id: "docs diagrams"
    branch "verification"
    checkout "verification"
    commit id: "unit-contract-system checks"
    checkout "006-modular-low-coupling"
    merge "verification"
```

## File Responsibility Matrix

| Area | Representative files | Responsibility | Primary flows |
|------|----------------------|----------------|---------------|
| Backend config | `backend/config/*`, `backend/manage.py` | ASGI/WSGI, URL routing, Celery bootstrap, settings composition | HTTP, WS, task queue |
| Backend core | `backend/core/*` | Shared exceptions, logging, pagination, middleware | all DRF requests |
| Accounts | `backend/apps/accounts/*` | users, roles, auth serializers, permissions | login, protected API |
| Exams | `backend/apps/exams/*` | rooms, exams, rosters | session planning |
| Cameras | `backend/apps/cameras/*` | camera CRUD, stream registration, camera status | live monitoring |
| Sessions | `backend/apps/sessions/*` | monitoring session lifecycle and comments | live monitoring |
| Detections | `backend/apps/detections/*` | live detection frames and predictions | live overlays |
| Anomalies | `backend/apps/anomalies/*` | event creation and triage | alerting workflow |
| Recordings | `backend/apps/recordings/*` | recording metadata and playback | review workflow |
| Exports | `backend/apps/exports/*` | async export model, services, Celery tasks | evidence bundle |
| Audit | `backend/apps/audit/*` | append-only action history | compliance trail |
| Health | `backend/apps/health/*` | subsystem health and model serving health | dashboard health |
| Pipeline | `backend/apps/pipeline/**/*` | inference layers, runtime clients, model lifecycle | live and offline inference |
| Tracking | `backend/apps/tracking/**/*` | track association, colors, ReID, rendering, export | stable identity overlays |
| Video analysis | `backend/apps/video_analysis/**/*` | upload jobs, job API, job WS, Celery processing | offline upload |
| Frontend API | `frontend/src/api/*.ts` | REST/WS client modules | frontend-backend contract |
| Frontend pages | `frontend/src/pages/*.tsx` | user workflow screens | route-level behavior |
| Frontend components | `frontend/src/components/**/*` | rendered UI and overlays | user-visible state |
| Frontend hooks | `frontend/src/hooks/*.ts` | WebSocket/WHEP/live state hooks | real-time behavior |
| Frontend stores | `frontend/src/stores/*.ts` | Zustand state per domain | UI state propagation |
| Frontend types | `frontend/src/types/*.ts` | shared TS contracts | compile-time validation |
| Tests | `backend/tests/**/*`, `frontend/tests/**/*` | unit, contract, integration, system, e2e checks | regression evidence |
| Scripts | `scripts/**/*` | CI gates and Triton artifact helpers | release and model operations |
| Infra | `infra/**/*`, `docker-compose.dev.yml`, `nginx.conf`, `go2rtc.yaml` | dev services, reverse proxy, native service config | deployment/runtime |
| Specs/docs | `specs/**/*`, `docs/**/*` | feature contracts and diagram evidence | planning and verification |
