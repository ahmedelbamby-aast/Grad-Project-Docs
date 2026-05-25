# Feature Specification: Modular Design and Low Coupling Hardening

**Feature Branch**: `006-modular-low-coupling`  
**Created**: 2026-05-07  
**Status**: Draft  
**Input**: User description: "We need to apply the new principles of Modular design and Low Coupling in the current system without breaking the Functionality."

## Clarifications

### Session 2026-05-08

- Q: What regression baseline scope must modularization preserve? → A: Full delivered system baseline: live monitoring, offline video, auth, exams, rooms/cameras, sessions, anomalies, exports, health, settings, and dashboard navigation.
- Q: What refactor depth is required in this feature? → A: Full modular restructuring: refactor all major modules to the new boundary model in this feature.
- Q: Which boundary model governs the restructuring? → A: Hybrid boundaries: domain capabilities inside each runtime path, with shared contracts across live/offline flows.
- Q: What temporary coupling policy applies? → A: Time-boxed, risk-ranked exceptions: high-risk coupling must be removed now; medium/low-risk coupling may remain only with owner, expiry, removal plan, and regression coverage.
- Q: What evidence is required to prove no broken functionality? → A: Full evidence pack: before/after workflow baseline, automated regression tests, real-data live/offline results, coverage report, and reviewer sign-off.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Preserve Existing User Workflows During Refactor (Priority: P1)

As an instructor or administrator, I need the current monitoring, video review, inference, and dashboard workflows to keep working while the system is modularized so that architecture improvements do not interrupt exam operations or reduce trust in the system.

**Why this priority**: The feature is only successful if it reduces internal coupling without changing expected user behavior. Preserving existing functionality is the primary safety boundary for all later improvements.

**Independent Test**: Can be tested by running the current live monitoring workflow and the current offline video processing workflow before and after the modularization effort, then confirming the same user-visible outcomes, statuses, alerts, overlays, and reports remain available.

**Acceptance Scenarios**:

1. **Given** an instructor starts a live monitoring session that worked before the change, **When** the modularized system processes the same classroom feed, **Then** the instructor sees the same detection overlays, tracking identifiers, anomaly status updates, and session controls as before.
2. **Given** an administrator uploads or selects an offline video that worked before the change, **When** the modularized system processes the same video, **Then** the administrator can review the same playback overlays, inference results, and stored analysis outcomes as before.
3. **Given** a user performs a non-video dashboard workflow such as login, exam setup, room/camera management, export review, or health review, **When** the modularized system is used, **Then** the workflow remains available with no user-visible regression.

---

### User Story 2 - Define Clear Module Ownership and Boundaries (Priority: P1)

As a developer, I need each major system capability to have clear ownership, responsibilities, inputs, outputs, and dependency boundaries so that future changes can be made in one area without causing unexpected behavior in another.

**Why this priority**: Low coupling starts with explicit boundaries. Without agreed module ownership, refactoring can create new hidden dependencies while trying to remove old ones.

**Independent Test**: Can be tested by reviewing the module boundary map and verifying each existing capability is assigned to one primary owning module, with documented dependencies and consumer-facing contracts.

**Acceptance Scenarios**:

1. **Given** the current system modules are reviewed, **When** a capability such as camera management, inference, tracking, anomaly triage, exports, authentication, or dashboard state is inspected, **Then** it has one documented owner and clear consumers.
2. **Given** a module depends on another module, **When** the dependency is reviewed, **Then** the reason for the dependency, the data exchanged, and the expected failure behavior are documented.
3. **Given** a module exposes behavior used elsewhere, **When** another module consumes it, **Then** the consumer relies on the documented contract rather than undocumented internal details.

---

### User Story 3 - Change One Module Without Cascading Regressions (Priority: P2)

As a developer, I need to update or replace a module through a stable contract so that feature work, model-serving changes, and deployment hardening can progress without cascading edits across unrelated areas.

**Why this priority**: The project is actively evolving toward hardened inference and deployment. A low-coupling design must make expected changes local, testable, and reversible.

**Independent Test**: Can be tested by selecting representative changes in the current system, such as swapping an inference provider, changing a tracking rule, or updating a dashboard data source, and proving that unrelated workflows remain unchanged.

**Acceptance Scenarios**:

1. **Given** an inference provider or model-serving path is changed, **When** live and offline processing are validated, **Then** camera management, user management, exports, and non-inference dashboard workflows do not require behavior changes.
2. **Given** a tracking or anomaly rule changes, **When** the related contract tests and system scenarios run, **Then** unrelated modules keep their previous behavior and data expectations.
3. **Given** a module is temporarily unavailable, **When** a dependent workflow encounters the failure, **Then** the user sees a documented fallback, degraded state, or actionable error without unrelated workflows failing.

---

### User Story 4 - Explain the Modular System Clearly (Priority: P2)

As a maintainer or reviewer, I need accurate diagrams and plain-language module explanations so that I can verify the design, onboard faster, and identify coupling risks before implementation begins.

**Why this priority**: Refactoring without explainability makes the system harder to govern. Diagrams and module explanations are required to prove that the design is understandable and aligned with the constitution.

**Independent Test**: Can be tested by asking a reviewer to trace live-stream processing, offline-video processing, frontend-backend interactions, backend-inference interactions, and deployment boundaries from documentation alone.

**Acceptance Scenarios**:

1. **Given** a reviewer opens the feature documentation, **When** they inspect the Mermaid diagrams and explanations, **Then** they can trace major user workflows and module dependencies without reading source code.
2. **Given** a module boundary changes, **When** documentation is reviewed, **Then** affected diagrams and related document links reflect the new boundary.
3. **Given** a coupling risk is found during review, **When** the risk is recorded, **Then** it has an owner, severity, mitigation path, and verification method.

### Edge Cases

- A module is shared by multiple workflows and cannot be assigned a single owner without splitting responsibilities.
- Existing behavior depends on undocumented cross-module access that must be preserved temporarily.
- A coupling reduction changes timing or ordering of user-visible events while producing the same final result.
- Live-stream processing succeeds while offline video processing fails, or the reverse.
- Real model weights or real raw test data are unavailable in the current environment.
- A module boundary improvement conflicts with an active feature branch or existing implementation task.
- A deployment boundary differs between Docker-based development/test and native Linux production.

### Mandatory Runtime Scenarios *(video/inference features)*

- **Live Stream Scenario**: A live classroom feed is processed through the current monitoring flow before and after modularization. The user-visible feed state, overlays, tracking continuity, anomaly updates, and recovery behavior must remain equivalent. Verification uses real model weights and real raw/live test data in development or test.
- **Offline Video Processing Scenario**: An uploaded or raw offline video is processed before and after modularization. The user-visible job status, stored results, playback overlays, and review experience must remain equivalent. Verification uses real model weights and real raw video data in development or test.
- **Frontend-Backend Wiring**: Existing user-facing actions and state updates must remain compatible through documented request/response and real-time message contracts. The user must not need to change how they operate the dashboard.
- **Backend-Inference Wiring**: Inference and tracking interactions must remain observable, health-checkable, and failure-aware through documented contracts. Real model artifacts and real test media are required for validation.
- **Deployment Boundary**: Development and test may rely on Docker-based services. Production Linux must remain supportable without Docker and must use native service assumptions in deployment documentation.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST preserve the full delivered system baseline while modular design and low-coupling improvements are applied, including live monitoring, offline video, auth, exams, rooms/cameras, sessions, anomalies, exports, health, settings, and dashboard navigation.
- **FR-002**: The system MUST define a module ownership map covering all current top-level capabilities, including frontend surfaces, backend domains, inference/tracking, storage-facing workflows, background processing, and deployment/runtime responsibilities.
- **FR-003**: Each module in the ownership map MUST have documented purpose, primary responsibilities, public inputs, public outputs, consumers, dependencies, and failure behavior.
- **FR-004**: The system MUST use hybrid boundaries as the approved module boundary model: domain capabilities own behavior, while live-stream and offline-video runtime paths define shared contracts across those capabilities.
- **FR-005**: The system MUST identify, rank, and document existing coupling risks before refactoring begins, including hidden dependencies, shared mutable state, duplicated responsibilities, direct cross-module access, and unclear ownership.
- **FR-006**: The system MUST prioritize behavior-preserving changes over behavior-changing redesign. Any intentional user-visible behavior change MUST be separately approved as its own feature or explicit requirement.
- **FR-007**: The system MUST refactor all major modules in the ownership map to the approved module boundary model within this feature, while preserving the full delivered system baseline.
- **FR-008**: The system MUST require regression coverage for every workflow affected by a module boundary change.
- **FR-009**: Live stream processing MUST remain functionally equivalent after modularization for supported camera/session workflows.
- **FR-010**: Offline video processing MUST remain functionally equivalent after modularization for supported upload/raw-video workflows.
- **FR-011**: Inference, tracking, video-processing, and overlay validation MUST use real model weights and real raw test data in development/test environments.
- **FR-012**: The system MUST maintain 100% line and branch coverage for affected modules and workflows, or record a time-bounded exception before implementation completion proceeds.
- **FR-013**: The system MUST document frontend-backend contracts for affected workflows so that user-facing state transitions remain explicit and testable.
- **FR-014**: The system MUST document backend-inference contracts for affected workflows so that prediction, tracking, health, timeout, and failure states remain explicit and testable.
- **FR-015**: The system MUST document development/test Docker assumptions separately from production native Linux assumptions.
- **FR-016**: The system MUST ensure no production requirement introduced by this feature depends on Docker availability.
- **FR-017**: The system MUST update Mermaid explainability diagrams for affected modules, workflows, and deployment boundaries.
- **FR-018**: Each updated diagram MUST include a detailed explanation that maps nodes, relationships, and flows to the current system behavior.
- **FR-019**: The system MUST provide a review checklist for confirming that modularity changes reduce coupling without removing required functionality.
- **FR-020**: The system MUST remove high-risk coupling in this feature.
- **FR-021**: The system MAY retain medium-risk or low-risk temporary coupling only when it is documented with owner, expiry, removal plan, and regression coverage.
- **FR-022**: The system MUST produce a full evidence pack before completion, including before/after workflow baseline, automated regression tests, real-data live/offline results, coverage report, and reviewer sign-off.

### Key Entities *(include if feature involves data)*

- **Module Boundary**: A documented boundary around a system capability, including its owner, responsibilities, public inputs, public outputs, consumers, dependencies, and failure behavior.
- **Hybrid Boundary Model**: The approved modularization model where domain capabilities own behavior and live-stream/offline-video runtime paths define shared contracts across capabilities.
- **Coupling Risk**: A known dependency or design issue that can cause unrelated modules to change together or fail together.
- **Behavior Baseline**: The expected user-visible behavior captured before modularization and used to prove no regression after the change.
- **Compatibility Contract**: A documented expectation between modules or user-facing surfaces that must remain stable unless changed through an approved feature.
- **Runtime Scenario**: A validated operating path, especially live stream processing and offline video processing, used to prove modularization preserves real workflows.
- **Refactor Exception**: A documented, time-bounded reason for temporarily retaining medium-risk or low-risk coupling while preserving system functionality.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of full delivered system baseline workflows pass regression validation after modularization.
- **SC-002**: 100% of affected modules have documented ownership, purpose, inputs, outputs, dependencies, consumers, and failure behavior.
- **SC-003**: 100% of identified coupling risks have severity, owner, mitigation, verification status, and exception eligibility recorded before implementation is marked ready.
- **SC-004**: 100% of major modules in the ownership map are refactored to the hybrid boundary model in this feature.
- **SC-005**: A representative module-boundary change can be validated without requiring unrelated user workflows to change.
- **SC-006**: Live stream processing passes regression validation with real model weights and real raw/live test data.
- **SC-007**: Offline video processing passes regression validation with real model weights and real raw video data.
- **SC-008**: Affected modules maintain 100% line and branch coverage, or every exception is documented with owner, expiry, and removal plan.
- **SC-009**: Reviewers can trace frontend-backend, backend-inference, and deployment boundaries from diagrams and explanations without reading source code.
- **SC-010**: Production deployment documentation continues to support native Linux operation without Docker dependency.
- **SC-011**: The full evidence pack is complete and reviewed before the feature is marked ready for implementation completion.

## Assumptions

- This feature is a behavior-preserving hardening effort, not a request to add new user-facing product capabilities.
- Current live stream and offline video workflows are the baseline for regression validation.
- Existing model weights and raw test data are available in development/test environments.
- Temporary coupling may remain only for medium-risk or low-risk cases where removing it would risk breaking functionality during full modular restructuring; such cases must be documented as refactor exceptions with owner, expiry, removal plan, and regression coverage.
- Detailed implementation techniques, module names, and sequencing will be decided during planning.
