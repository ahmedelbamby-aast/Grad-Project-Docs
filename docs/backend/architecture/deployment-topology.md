# Deployment Topology — Containers, Native Services, and Network Boundaries

**Updated**: 2026-05-25

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
    Django --> Authority["Selected Triton profile only"]
    Authority --> Live["live: 39000/39001/39002"]
    Authority --> Offline["offline: 39100/39101/39102"]
    Systemd["user-space native service"] --> Authority
    ModelRepo["configured model repository"] --> Authority
    CameraInfra["RTSP cameras + relay tier"] --> Django
```

### Production notes

- Production inference requires a native Linux Triton process operated without
  Docker or sudo assumptions.
- Two endpoint profiles are configured, but exactly one is active: live
  `39000/39001/39002` or offline `39100/39101/39102`; inactive ports must be
  unreachable during readiness acceptance.
- Backend runtime policy must fail closed or emit explicitly
  non-authoritative degradation when selected Triton/model readiness fails.
  Production local inference fallback is forbidden.

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
    WorkerSvc --> TritonDep["active Triton authority"]
    NginxSvc["nginx"] --> BackendSvc
    NginxSvc --> RelayDep
```

---

## 4. Runtime Dependency Order

1. Redis and PostgreSQL first.
2. Relay tier (`go2rtc` and/or gateway service) next.
3. Backend API + Channels.
4. Celery workers.
5. Selected native Triton endpoint profile and required models, validated
   before accepting inference work.
6. Frontend and operator clients.

## Related Documents

- [ARCHITECTURE.md](../../ARCHITECTURE.md)
- [data-flow.md](data-flow.md)
- [triton-operations.md](triton-operations.md)
