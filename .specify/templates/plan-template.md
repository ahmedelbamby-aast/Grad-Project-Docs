# Implementation Plan: [FEATURE]

**Branch**: `[###-feature-name]` | **Date**: [DATE] | **Spec**: [link]
**Input**: Feature specification from `/specs/[###-feature-name]/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/plan-template.md` for the execution workflow.

## Summary

[Extract from feature spec: primary requirement + technical approach from research]

## Technical Context

<!--
  ACTION REQUIRED: Replace the content in this section with the technical details
  for the project. The structure here is presented in advisory capacity to guide
  the iteration process.
-->

**Language/Version**: [e.g., Python 3.11, Swift 5.9, Rust 1.75 or NEEDS CLARIFICATION]  
**Primary Dependencies**: [e.g., FastAPI, UIKit, LLVM or NEEDS CLARIFICATION]  
**Storage**: [if applicable, e.g., PostgreSQL, CoreData, files or N/A]  
**Testing**: [e.g., pytest, XCTest, cargo test or NEEDS CLARIFICATION]  
**Target Platform**: [e.g., Linux server, iOS 15+, WASM or NEEDS CLARIFICATION]
**Project Type**: [e.g., library/cli/web-service/mobile-app/compiler/desktop-app or NEEDS CLARIFICATION]  
**Performance Goals**: [domain-specific, e.g., 1000 req/s, 10k lines/sec, 60 fps or NEEDS CLARIFICATION]  
**Constraints**: [domain-specific, e.g., <200ms p95, <100MB memory, offline-capable or NEEDS CLARIFICATION]  
**Scale/Scope**: [domain-specific, e.g., 10k users, 1M LOC, 50 screens or NEEDS CLARIFICATION]
**Runtime Scenarios**: [Live stream scenario and offline video processing scenario; state N/A rationale for either if not applicable]
**Inference/Tracking Reference**: [Triton/RTMPose/model contract and tracking/ReID decisions, or N/A]
**Runtime Authority**: [Production Triton-only route; active `live` or `offline` endpoint profile; inactive-profile rejection evidence]
**Temporal/Identity Authority**: [Timestamp envelope, canonical identity scope, lifecycle/ReID and invalidation rules]
**Evidence/Schema Authority**: [Telemetry, benchmark, API/WS/artifact/sequence schema versions and evidence outputs]
**Deployment Topology**: [Dev/test services and production native Linux services; production MUST NOT assume Docker or sudo]
**Runtime Reconciliation**: [Task/DB/artifact/queue/frontend convergence checks and failure handling]
**Lineage/Fingerprints**: [Evidence, model, runtime, deployment, benchmark, dataset, telemetry and artifact digest lineage]
**Budgets/SLOs**: [Latency, queue, retry, timeout, degradation and rollback thresholds]

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

[Gates determined based on constitution file]

> **Production Runtime Authority Gate**: Every production inference
> plan MUST enforce native Linux, no-Docker/no-sudo assumptions,
> Triton-only GPU inference, and exactly one active endpoint profile
> (`live` or `offline`). It MUST define startup rejection and
> inactive-profile validation. Local/mock inference cannot close a
> production gate.

> **Heterogeneous Runtime Maturity Gate**: Plans MUST separate local
> Windows/Docker contract validation from native Linux RTX 5090 production
> authority. Production claims MUST name the committed Git SHA,
> `backend/.env` fingerprint, PostgreSQL authority, Redis/Celery route,
> Triton profile, evidence root and accepted production job lineage.

> **Temporal and Identity Truth Gate**: Any plan touching frames,
> tracks, pose, behavior, events, artifacts, or replay MUST define
> source/ingest/queue/inference/persistence timestamps, gap and drift
> handling, session/camera/canonical/local identity scope, ReID
> provenance, lifecycle rules, and invalidation behavior.

> **Pose and Behavior Semantics Gate**: Any pose or behavioral plan
> MUST distinguish `raw_keypoints`, `smoothed_keypoints`, and
> `display_keypoints`, declare visibility/interpolation rules, and
> define versioned feature/ontology and missing-data/ambiguity
> semantics before making behavioral claims.

> **Queue and Failure Gate**: Any orchestrated or streaming feature
> MUST define deterministic queue ownership, worker isolation,
> bounded retry/DLQ/backpressure behavior, RTSP reconnect state
> transitions where applicable, and fail-stop/degraded evidence
> handling.

> **Contract and Storage Gate**: Every external or persisted payload
> MUST have a governed versioned schema, explicit serializer exposure,
> idempotency and migration/retention rules. `fields = '__all__'`,
> unversioned evolution, and silent contract drift are forbidden.

> **Observability and Scientific Evidence Gate**: Any readiness,
> performance, quality, anomaly, or maturity claim MUST define
> probe-backed telemetry and immutable evidence. Benchmarks MUST use
> independent real baseline/candidate runs, variance and confidence
> reporting; research claims require statistical method and effect
> size disclosure.

> **Live/Offline Validation Gate**: Video or inference features MUST
> specify both live and offline validation when their behavior spans
> both modes. Each run occurs in a separate valid active-profile
> window; N/A requires an explicit rationale.

> **Anti-Regression Runtime Truth Gate**: Runtime claims MUST reconcile
> process, port, endpoint, queue, GPU, model, telemetry, task, database,
> artifact and frontend state. Acceptance scripts MUST inspect runtime truth
> and artifact content, not just booleans or file existence.

> **Evidence Integrity and Lineage Gate**: Evidence MUST be durable,
> immutable after publication, digest-addressed and reproducible. It MUST
> distinguish mock/real, CPU/GPU, synthetic/production, dev/prod and
> fallback/canonical execution. Placeholder evidence, temporary-path-only
> manifests and SQLite-backed evidence are forbidden.

> **Replay and Closure Gate**: Lifecycle or benchmark plans MUST define
> explicit replay policy (`reuse-success`, `fail-on-existing`, or
> `new-attempt`), reject failed replay lineage for acceptance, and define
> final closure evidence under `ci_evidence/production/runtime_maturity/final/`
> when the active runtime maturity plan applies.

> **XFail, Drift and Debt Gate**: Hidden xfails, untracked runtime overrides,
> environment drift tolerance and unbounded technical debt MUST block closure.
> Formal deferrals require owner, scope, expiry condition and affected
> constitutional rule.

## Project Structure

### Documentation (this feature)

```text
specs/[###-feature]/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)
<!--
  ACTION REQUIRED: Replace the placeholder tree below with the concrete layout
  for this feature. Delete unused options and expand the chosen structure with
  real paths (e.g., apps/admin, packages/something). The delivered plan must
  not include Option labels.
-->

```text
# [REMOVE IF UNUSED] Option 1: Single project (DEFAULT)
src/
├── models/
├── services/
├── cli/
└── lib/

tests/
├── contract/
├── integration/
└── unit/

docs/
└── src/           # Mirrors src/ with .md files per source file
    ├── models/
    ├── services/
    ├── cli/
    └── lib/

# [REMOVE IF UNUSED] Option 2: Web application (when "frontend" + "backend" detected)
backend/
├── src/
│   ├── models/
│   ├── services/
│   └── api/
└── tests/

frontend/
├── src/
│   ├── components/
│   ├── pages/
│   └── services/
└── tests/

docs/                # Mirrors backend/ and frontend/ with .md files
├── backend/         # per source file (see constitution §docs/)
│   └── src/
└── frontend/
    └── src/

# [REMOVE IF UNUSED] Option 3: Mobile + API (when "iOS/Android" detected)
api/
└── [same as backend above]

ios/ or android/
└── [platform-specific structure: feature modules, UI flows, platform tests]
```

**Structure Decision**: [Document the selected structure and reference the real
directories captured above]

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |
