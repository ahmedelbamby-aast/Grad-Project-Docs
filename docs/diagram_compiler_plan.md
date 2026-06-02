# Project Diagram Compiler Plan

**Last updated:** 2026-05-22

## Summary

Build a canonical, generated diagram pipeline under `scripts/diagrams/` that extracts implementation facts from the repo, emits normalized graph JSON, generates Mermaid diagrams, renders `PNG`, `SVG`, and `PDF` through `mmdc_wrapper.py`, and fails CI when generated diagrams drift from committed output.

Defaults locked:

- Generated diagrams are canonical for implementation-backed diagrams.
- Use a mix of low-dependency custom extractors and targeted external tools.
- Cover the entire repo, including `External Resources`, but tag non-owned/reference material separately so it is not presented as implemented behavior.

## Key Changes

### 1. Extract

- Backend extractor: Django models, DRF URLs, admin URLs, Channels routes, Celery tasks, app boundaries, import dependencies, service call edges.
- Frontend extractor: React routes, API clients, hooks, stores, components, test boundaries, TS/TSX dependency graph.
- Infra/scripts extractor: `docker-compose.dev.yml`, `nginx.conf`, `go2rtc.yaml`, `infra/**`, `scripts/**`, PowerShell script flow, service dependencies.
- Requirements extractor: parse `specs/**`, `docs/**`, and `docs/External Resources/**` for FR/NFR statements, risk/quality gates, and reference-only knowledge diagrams.
- Drift extractor: compare generated implementation facts against existing Mermaid blocks and report mismatches.

### 2. Normalize

- Write deterministic JSON artifacts to `docs/diagrams/generated/data/`.
- Use stable IDs, sorted nodes/edges, source path metadata, and `authority` tags: `implemented`, `specified`, `reference`, `manual`.
- Keep generated graph data small enough to pass to an LLM later without sending the whole codebase.

### 3. Generate Mermaid

- Emit canonical `.mmd` and index `.md` files under `docs/diagrams/generated/`.
- Generate these diagram families:
  - `erDiagram`: Django model relationships.
  - `flowchart`: high-level, mid-level, and low-level architecture.
  - `flowchart`: modules, services, scripts, infra, and dependency maps.
  - `sequenceDiagram`: request/task/WebSocket flows and "who calls who".
  - `stateDiagram-v2`: jobs, sessions, cameras, streams, exports, health states.
  - `classDiagram`: selected backend service classes and frontend component contracts.
  - `requirementDiagram`: FR/NFR coverage mapped to code/test evidence.
  - `journey`: implemented user workflows only when route/API/task evidence exists.
  - `timeline` or `gitGraph`: release, CI, model-export, and deployment flows.
- Existing hand-written diagrams move conceptually into `docs/diagrams/manual/` or are marked with `<!-- diagram-authority: manual -->`.

### 4. Render

- Reuse and extend `mmdc_wrapper.py` as the only renderer.
- All diagrams render with one shared Mermaid config, same theme, and transparent background.
- Output formats are always `.mmd`, `.md`, `.svg`, `.png`, and `.pdf`.
- Rendered assets go to `Images_Docs/generated/` with deterministic semantic names and no duplicates.

## Public Interfaces

Add these commands:

```powershell
.\.venv\Scripts\python.exe scripts\diagrams\build_all.py
.\.venv\Scripts\python.exe scripts\diagrams\build_all.py --check
.\.venv\Scripts\python.exe scripts\diagrams\build_all.py --render
.\.venv\Scripts\python.exe scripts\diagrams\audit_existing.py --output docs\mermaid_drift_audit.md
```

Add focused dependencies:

- Backend dev group: `django-extensions`, `pydeps` or `pylint` for `pyreverse`, `PyYAML`.
- Frontend dev dependencies: `dependency-cruiser` preferred over `madge` for stricter TS dependency rules.
- Keep custom Python AST/Django extractors as primary truth where external tools are too generic.

CI integration:

- Extend `scripts/ci/verify_docs_diagrams.py` or add `scripts/ci/verify_generated_diagrams.py`.
- CI fails when generated `.mmd`, `.md`, or graph JSON differs from committed canonical output.
- CI also fails when a generated implementation diagram references a missing file, route, model, task, or consumer.

## Test Plan

Use layered tests:

- Unit tests for each extractor using small fixture files.
- Snapshot tests for normalized graph JSON and generated Mermaid output.
- Integration test against this repo that verifies:
  - all generated Mermaid parses successfully;
  - all rendered `SVG`, `PNG`, and `PDF` assets are produced;
  - every implementation-backed diagram node maps to an existing file, model, route, task, script, or config key;
  - every current drift finding is either fixed or intentionally marked manual/reference.
- CI check mode must be deterministic: running it twice with no source changes produces no diff.

Acceptance criteria:

- `python scripts/diagrams/build_all.py --check` exits `0` on a clean generated state.
- `Images_Docs/generated/` contains transparent `PNG`, `SVG`, and `PDF` for every generated diagram.
- Existing architecture diagrams that claim implementation truth are either generated or explicitly marked manual/reference.
- The generated graph JSON is compact enough to use as LLM context for future doc updates.

## Assumptions

- "Entire repo" means every markdown/code/config file is scanned, but only project-owned code/config is treated as implemented truth.
- `docs/External Resources/**` is included as reference material and may produce reference diagrams, but it will not override actual repo implementation facts.
- Generated diagrams become the canonical source for implementation-backed architecture, dependency, ERD, API, task, WebSocket, and requirements-coverage diagrams.
- Manual diagrams remain allowed only when they are marked as manual/reference and excluded from strict implementation drift checks.
