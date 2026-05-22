# Mermaid Drift Audit (Implementation vs Diagram)

Date: 2026-05-22

This audit lists Mermaid diagrams that currently do not reflect the real implemented code paths, models, or runtime endpoints.

## Findings

1. Runtime analytics file path is outdated in multiple architecture diagrams.
- Diagram references:
  - `docs/ARCHITECTURE.md` (High-Level Runtime Topology): `pipeline/views_runtime_analytics.py`
  - `docs/diagrams/SYSTEM_MERMAID_ATLAS.md` (Runtime Container Flow): `views_runtime_analytics.py`
  - `docs/backend/architecture/data-flow.md` (Runtime Telemetry Flow): `views_runtime_analytics.py`
- Implemented code:
  - Runtime analytics views are implemented inside `backend/apps/video_analysis/views.py` (classes `Runtime*View`), not in a separate `views_runtime_analytics.py`.
- Evidence:
  - [docs/ARCHITECTURE.md](E:\grad_project\docs\ARCHITECTURE.md)
  - [SYSTEM_MERMAID_ATLAS.md](E:\grad_project\docs\diagrams\SYSTEM_MERMAID_ATLAS.md)
  - [data-flow.md](E:\grad_project\docs\backend\architecture\data-flow.md)
  - [views.py](E:\grad_project\backend\apps\video_analysis\views.py)

2. Runtime ingestion module path is incorrect in diagrams.
- Diagram references:
  - `docs/ARCHITECTURE.md`: `pipeline/services/runtime_ingestion.py`
- Implemented code:
  - Runtime ingestion module is `backend/apps/pipeline/runtime_ingestion.py` (not under `services/`).
- Evidence:
  - [ARCHITECTURE.md](E:\grad_project\docs\ARCHITECTURE.md)
  - [runtime_ingestion.py](E:\grad_project\backend\apps\pipeline\runtime_ingestion.py)

3. Runtime telemetry model names in diagrams do not match current database models.
- Diagram references:
  - `docs/diagrams/SYSTEM_MERMAID_ATLAS.md`: `FrameLifecycleEvent + ModelExecutionMetric + ServiceHealthSnapshot`
  - `docs/backend/architecture/data-flow.md`: `ModelExecutionMetric`, `ServiceHealthSnapshot`
  - `docs/backend/architecture/observability-runbook.md`: same set
- Implemented code:
  - Existing runtime models are `FrameLifecycleEvent`, `ModelCallEvent`, `RuntimeRawTruthEvent`, `RuntimeSessionRollup`, `RunSessionSummary`.
  - `ModelExecutionMetric` and `ServiceHealthSnapshot` do not exist.
- Evidence:
  - [SYSTEM_MERMAID_ATLAS.md](E:\grad_project\docs\diagrams\SYSTEM_MERMAID_ATLAS.md)
  - [data-flow.md](E:\grad_project\docs\backend\architecture\data-flow.md)
  - [observability-runbook.md](E:\grad_project\docs\backend\architecture\observability-runbook.md)
  - [pipeline/models.py](E:\grad_project\backend\apps\pipeline\models.py)

4. Runtime API prefix in diagrams is incorrect.
- Diagram references:
  - `docs/backend/architecture/data-flow.md`: `/api/v1/runtime/*`
  - `docs/backend/architecture/observability-runbook.md`: `/api/v1/runtime/*`
- Implemented code:
  - Runtime endpoints are under `/api/v1/video-analysis/runtime/*` via `backend/apps/video_analysis/urls.py`, mounted by `backend/config/urls.py`.
- Evidence:
  - [data-flow.md](E:\grad_project\docs\backend\architecture\data-flow.md)
  - [observability-runbook.md](E:\grad_project\docs\backend\architecture\observability-runbook.md)
  - [video_analysis/urls.py](E:\grad_project\backend\apps\video_analysis\urls.py)
  - [config/urls.py](E:\grad_project\backend\config\urls.py)

5. Live-stream WHEP path in sequence diagram is too generic and mismatched with actual client behavior.
- Diagram reference:
  - `docs/diagrams/SYSTEM_MERMAID_ATLAS.md` (Live Streaming Sequence): `open /whep/{stream}`
- Implemented code:
  - Frontend opens `/whep/{camera_id}/` and derives `camera_id` from stream name in `useWhepClient.ts`.
- Evidence:
  - [SYSTEM_MERMAID_ATLAS.md](E:\grad_project\docs\diagrams\SYSTEM_MERMAID_ATLAS.md)
  - [useWhepClient.ts](E:\grad_project\frontend\src\hooks\useWhepClient.ts)

6. ER cardinality between `USER` and `ROLE` is reversed in the atlas ER diagram.
- Diagram reference:
  - `docs/diagrams/SYSTEM_MERMAID_ATLAS.md`: `USER ||--o{ ROLE : has`
- Implemented code:
  - `User` has a FK to `Role` (`role = models.ForeignKey(Role, ...)`), so one `Role` has many `User` rows.
  - Correct direction should be `ROLE ||--o{ USER`.
- Evidence:
  - [SYSTEM_MERMAID_ATLAS.md](E:\grad_project\docs\diagrams\SYSTEM_MERMAID_ATLAS.md)
  - [accounts/models.py](E:\grad_project\backend\apps\accounts\models.py)

7. ER entity naming drift: `LIVE_DETECTION` and `OFFLINE_DETECTION` are not real model names.
- Diagram reference:
  - `docs/diagrams/SYSTEM_MERMAID_ATLAS.md` (Database Entity Relationships)
- Implemented code:
  - Live detection model is `apps.detections.models.Detection`.
  - Offline detection model is `apps.video_analysis.models.Detection`.
  - Diagram uses conceptual aliases that do not map to actual table/model names.
- Evidence:
  - [SYSTEM_MERMAID_ATLAS.md](E:\grad_project\docs\diagrams\SYSTEM_MERMAID_ATLAS.md)
  - [detections/models.py](E:\grad_project\backend\apps\detections\models.py)
  - [video_analysis/models.py](E:\grad_project\backend\apps\video_analysis\models.py)

## Notes

- Many Mermaid blocks in `docs/frontend/src/**` intentionally use simplified `.ts` labels while implementation files are `.tsx`; these are naming/label shortcuts, not always flow-level drift.
- Specs under `specs/**` may describe planned or contract-state behavior; this report focuses on docs that present current implementation architecture/flow.
