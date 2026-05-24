# Implementation Plan: Parallel Pose Inference

**Branch**: `009-parallel-pose-inference` | **Date**: 2026-05-23 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `specs/009-parallel-pose-inference/spec.md`

## Summary

Refactor the pipeline so object detection and RTMPose run concurrently for live and offline processing while preserving shared frame identity, complete artifacts, and explicit degraded/failure semantics. The plan enforces hybrid runtime validation (Intel GPU/OpenVINO + NVIDIA GPU/Triton in one real-video run), benchmarked optimization profiles, grayscale evidence gating, and a governed telemetry boundary where dashboards use authenticated REST/WebSocket APIs and a mandatory read-only MCP server consumes normalized persisted evidence.

## Technical Context

**Language/Version**: Python 3.11 (backend/inference workers), TypeScript (frontend dashboards), PowerShell/Bash (ops scripts)  
**Primary Dependencies**: Celery, Redis/RQ transport, OpenVINO runtime, Triton Inference Server, RTMPose + detector stack, FastAPI/Django API layer, WebSocket transport, Nsight Systems CLI, Intel PresentMon  
**Storage**: Relational metadata store + artifact/object storage + telemetry evidence persistence  
**Testing**: `pytest` (+ `pytest-xdist`), Playwright, Vitest, benchmark harnesses for live/offline throughput-latency comparison  
**Target Platform**: Windows development host, Linux inference services (production native Linux; Docker only for dev/test and instrumented validation profiles)  
**Project Type**: Web application with backend inference services + frontend monitoring UI  
**Performance Goals**: At least 2.0x offline throughput vs sequential baseline; at least 20% p95 processing latency reduction for accepted hybrid optimized profile; live delayed-timeline p95 jitter < 1.5s after buffer establishment  
**Constraints**: Preserve every ingestible live frame via delayed-live buffer (default 5 minutes); use bounded tiered buffering (RAM queue plus disk spill queue) under pressure; fail offline run if >2% decoded frames miss required pose status or any continuous pose gap >10s; no misleading combined artifacts during degraded states  
**Scale/Scope**: Dual-pipeline live sessions and offline jobs with per-model dedicated workers, model-runtime isolation, bounded backpressure, tiered overload buffering, and complete evidence packaging  
**Runtime Scenarios**: Mandatory live stream scenario and mandatory offline video processing scenario are both in scope  
**Inference/Tracking Reference**: Ultralytics docs are authoritative for YOLO prediction/tracking decisions; RTMPose remains top-down using detector regions first with validated degraded fallback regions only  
**Deployment Topology**: Dev/test uses containerized services where applicable; production assumes native Linux services and must not require Docker availability

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- Supreme Directive Gate: PASS for planning scope. Plan acknowledges commit/documentation and cross-linking obligations and keeps artifacts under `specs/009-parallel-pose-inference/` synchronized.
- Test-in-Loop Gate: PASS. Tasks must define tests before implementation and enforce RED->GREEN->REFACTOR sequencing across unit/integration/system levels.
- 100% Real-Data Test Gate: PASS with explicit requirement to use real model weights and representative raw live/offline media for inference paths.
- Live/Offline Scenario Gate: PASS. Both scenarios are mandatory and independently validated.
- System Hardening Gate: PASS. Plan preserves frontend-backend contracts, backend-inference runtime wiring, telemetry normalization boundary, Docker dev/test vs native Linux production split.
- Ultralytics Authority Gate: PASS. YOLO prediction/tracking configuration decisions remain tied to official Ultralytics guidance and recorded in research artifacts.

## Project Structure

### Documentation (this feature)

```text
specs/009-parallel-pose-inference/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
└── tasks.md
```

### Source Code (repository root)

```text
backend/
├── apps/
├── services/
├── inference/
├── telemetry/
└── tests/
    ├── unit/
    ├── integration/
    ├── contract/
    ├── system/
    ├── performance/
    └── resilience/

frontend/
├── src/
│   ├── components/
│   ├── pages/
│   ├── stores/
│   └── services/
└── tests/

docs/
├── backend/
└── frontend/
```

**Structure Decision**: Web application with split frontend/backend and dedicated inference + telemetry service boundaries. Planning artifacts remain spec-local; implementation tasks must map to backend worker orchestration, artifact contracts, telemetry ingestion/projection, and frontend state/reporting integration.

## Phase 0: Research Plan (Completed)

Research outputs are captured in [research.md](./research.md) and resolve key unknowns:
- Hybrid telemetry tooling and compatibility rules (Nsight CLI + Triton metrics + PresentMon + OpenVINO proof).
- Mandatory run-correlation model for baseline vs optimized comparability.
- MCP scope restrictions (read-only, bounded resources/tools, audited denials, no mutation commands).
- Grayscale acceptance/rejection gates tied to quality parity and throughput targets.

## Phase 1: Design & Contracts Plan (Completed)

Design outputs are available and aligned with the spec:
- Data model: [data-model.md](./data-model.md)
- Operational/implementation guidance: [quickstart.md](./quickstart.md)
- External interfaces/contracts: [`contracts/`](./contracts/)

Design emphasis:
- Shared frame timeline identity and reconciled object/pose/association artifacts.
- Per-model dedicated Celery worker isolation independent of runtime vendor.
- Hybrid validation evidence package as a first-class acceptance object.
- Dashboard API and MCP are both mandatory telemetry access surfaces over the same normalized persistence, with dashboard as the primary frontend data plane and MCP as the mandatory AI/operator access plane.

## Phase 2: Task Planning Approach

`/speckit.tasks` should produce dependency-ordered tasks grouped by:
1. Pipeline concurrency orchestration and frame identity propagation.
2. Artifact persistence semantics + degraded/failure state modeling.
3. Worker model isolation and runtime routing by model identity/workload mode.
4. Hybrid telemetry capture ingestion/normalization + dashboard contracts.
5. MCP read-only telemetry resources/tools + RBAC/audit enforcement.
6. Benchmark and validation harnesses for baseline vs optimized hybrid runs.
7. Final completion gate evidence, including source-fidelity export checks.

## Post-Design Constitution Re-Check

- All original gates remain PASS.
- No justified constitution violations are required at planning stage.
- Any future exception must be added to Complexity Tracking with owner, expiry, and removal plan.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| None | N/A | N/A |
