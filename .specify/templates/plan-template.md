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
**Inference/Tracking Reference**: [Ultralytics official docs decisions for YOLO prediction/tracking, or N/A]
**Deployment Topology**: [Dev/test Docker services and production native Linux services; production MUST NOT assume Docker]

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

[Gates determined based on constitution file]

> **Supreme Directive Gate**: Every plan MUST confirm that the
> commit-after-every-modification, ALL-diagram-types, detailed
> diagram explanations, and .md cross-linking requirements from
> constitution §Supreme Directive are acknowledged and will be
> enforced throughout implementation. This gate is BLOCKING.

> **Test-in-Loop Gate**: Every plan MUST confirm that the
> Test-in-Loop methodology (constitution §II) will be enforced:
> ALL tests (unit, integration, system) MUST be written BEFORE
> implementation for every task. No implementation code may be
> written until corresponding tests exist and fail. This gate
> is BLOCKING.

> **100% Real-Data Test Gate**: Every plan MUST require 100% line
> and branch coverage for all affected modules and features. Tests
> for inference, prediction, tracking, video processing, and overlays
> MUST use real model weights and real raw test data. Mocks MUST NOT
> replace model weights, raw media, trackers, prediction outputs, or
> frontend-backend/backend-Triton wiring.

> **Live/Offline Scenario Gate**: Every video or inference plan MUST
> define both a live stream scenario and an offline video processing
> scenario, including performance targets, failure handling, and
> system tests for each. If a scenario is not applicable, the plan
> MUST explain why.

> **System Hardening Gate**: Every plan MUST document frontend-backend
> contracts, backend-Triton wiring, Docker development/test topology,
> and production native Linux topology. Production plans MUST NOT
> depend on Docker availability.

> **Ultralytics Authority Gate**: Prediction, tracking, tracker
> configuration, and YOLO inference decisions MUST use the official
> Ultralytics docs website as the primary reference. Deviations MUST
> be documented in research.md with rationale.

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
