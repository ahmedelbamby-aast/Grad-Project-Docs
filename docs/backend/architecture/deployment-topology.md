# Deployment Topology — Containers, Native Services, and Network Boundaries

**Updated**: 2026-05-15

## 1. Development Topology (`docker-compose.dev.yml`)

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
    DevHost["Developer host"] --> Compose["docker-compose.dev.yml"]
    Compose --> Postgres["postgres container"]
    Compose --> Redis["redis container"]
    Compose --> Go2rtc["go2rtc container"]
    Compose --> Triton["tritonserver container (profile: triton)"]
    Compose --> Exporters["rtmpose-triton and model-converter helper services"]
    Backend["backend app/celery (host or containerized)"] --> Postgres
    Backend --> Redis
    Backend --> Go2rtc
    Backend --> Triton
```

### Dev notes

- Triton is profile-gated and can be omitted for local fallback-only runs.
- `backend\models\triton_repository` is mounted/used as the model repository source.
- Frontend dev server talks to backend and WHEP through nginx or direct local URLs depending on env.

---

## 2. Production Topology (Native Triton + Systemd)

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart LR
    Internet["Clients"] --> Nginx["Nginx edge"]
    Nginx --> Django["Django APIs + Channels"]
    Django --> Celery["Celery workers"]
    Django --> Redis[("Redis")]
    Django --> Postgres[("PostgreSQL")]
    Django --> Triton["Triton HTTP/gRPC/metrics"]
    Systemd["infra/systemd/triton-server.service"] --> Triton
    ModelRepo["/opt/triton/models (or configured path)"] --> Triton
    CameraInfra["RTSP cameras + relay tier"] --> Django
```

### Production notes

- Triton is expected to run as a native service via `infra\systemd\triton-server.service`.
- Triton ports (`8000/8001/8002`) are internal-only.
- Backend runtime policy can still route to local inference if Triton is unavailable.

---

## 3. Container-to-Service Dependencies

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
graph LR
    BackendSvc["backend service"] --> RedisDep["redis dependency"]
    BackendSvc --> PgDep["postgres dependency"]
    BackendSvc --> RelayDep["go2rtc/gateway dependency"]
    WorkerSvc["celery worker"] --> RedisDep
    WorkerSvc --> PgDep
    WorkerSvc --> TritonDep["triton optional dependency"]
    NginxSvc["nginx"] --> BackendSvc
    NginxSvc --> RelayDep
```

---

## 4. Runtime Dependency Order

1. Redis and PostgreSQL first.
2. Relay tier (`go2rtc` and/or gateway service) next.
3. Backend API + Channels.
4. Celery workers.
5. Optional Triton service (for Triton-enabled workloads).
6. Frontend and operator clients.

## Related Documents

- [ARCHITECTURE.md](../../ARCHITECTURE.md)
- [data-flow.md](data-flow.md)
- [triton-operations.md](triton-operations.md)
