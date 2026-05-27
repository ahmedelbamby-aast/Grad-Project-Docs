---

description: "Task list template for feature implementation"
---

# Tasks: [FEATURE NAME]

**Input**: Design documents from `/specs/[###-feature-name]/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests and Evidence**: Tests and evidence tasks are mandatory for every
applicable constitutional authority. Runtime/ML production claims MUST be
validated through the native Linux, GPU-backed, Triton-only route with one
active endpoint profile and real media. Temporal, identity, pose, behavioral,
queue, telemetry, schema, storage, security, benchmark, and failure semantics
MUST have focused validation and evidence paths. Mocks MAY support unit
isolation, but cannot close production or scientific maturity gates.
Acceptance tasks MUST include runtime reconciliation, immutable evidence
lineage, artifact authenticity, xfail/drift/debt checks and rollback criteria
for every production or maturity claim.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Single project**: `src/`, `tests/` at repository root
- **Web app**: `backend/src/`, `frontend/src/`
- **Mobile**: `api/src/`, `ios/src/` or `android/src/`
- Paths shown below assume single project - adjust based on plan.md structure

<!-- 
  ============================================================================
  IMPORTANT: The tasks below are SAMPLE TASKS for illustration purposes only.
  
  The /speckit.tasks command MUST replace these with actual tasks based on:
  - User stories from spec.md (with their priorities P1, P2, P3...)
  - Feature requirements from plan.md
  - Entities from data-model.md
  - Endpoints from contracts/
  
  Tasks MUST be organized by user story so each story can be:
  - Implemented independently
  - Tested independently
  - Delivered as an MVP increment
  
  DO NOT keep these sample tasks in the generated tasks.md file.
  ============================================================================
-->

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 Create project structure per implementation plan
- [ ] T002 Initialize [language] project with [framework] dependencies
- [ ] T003 [P] Configure linting and formatting tools

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

Examples of foundational tasks (adjust based on your project):

- [ ] T004 Setup database schema and migrations framework
- [ ] T005 [P] Implement authentication/authorization framework
- [ ] T006 [P] Setup API routing and middleware structure
- [ ] T007 Create base models/entities that all stories depend on
- [ ] T008 Configure error handling and logging infrastructure
- [ ] T009 Setup environment configuration management
- [ ] T010 [P] Define native Linux Triton-only runtime mode, endpoint profile,
      GPU/model readiness, and inactive-profile rejection evidence
- [ ] T011 [P] Define timestamp envelope, continuity/drift, canonical identity,
      lifecycle/ReID, and pose stream authority contracts
- [ ] T012 [P] Define explicit API/WS/artifact/sequence schema versions,
      serializer fields, telemetry truth, idempotency and retention rules
- [ ] T013 [P] Define queue isolation, bounded retry/DLQ, backpressure,
      reconnect and failure/degradation evidence contracts
- [ ] T014 [P] Define runtime reconciliation checks for task, database,
      queue, artifact, telemetry and frontend state convergence
- [ ] T015 [P] Define immutable evidence lineage, artifact digest,
      environment/runtime/dependency/GPU fingerprint and dataset/telemetry
      provenance requirements
- [ ] T016 [P] Define xfail registry, runtime-drift detection, technical debt
      budget, rollback criteria and acceptance veto conditions

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - [Title] (Priority: P1) 🎯 MVP

**Goal**: [Brief description of what this story delivers]

**Independent Test**: [How to verify this story works on its own]

### Validation and Evidence for User Story 1 (MANDATORY)

> **MANDATORY**: Define applicable runtime, integrity, contract, failure, and
> scientific evidence gates before claiming this story complete

- [ ] T017 [P] [US1] Unit test for [module/function] in tests/unit/test_[name].py
- [ ] T018 [P] [US1] Contract/schema test for [endpoint/event/artifact] in tests/contract/test_[name].py
- [ ] T019 [P] [US1] Integrity test for [timestamp/identity/pose/feature/failure semantics] in tests/integration/test_[name].py
- [ ] T020 [P] [US1] Runtime reconciliation test for task/database/queue/artifact/telemetry/frontend convergence
- [ ] T021 [P] [US1] Evidence authenticity test for digest, lineage, fingerprint and non-placeholder artifacts
- [ ] T022 [P] [US1] Native Triton live-profile validation with real media and evidence manifest in tests/system/test_[name]_live.py
- [ ] T023 [P] [US1] Native Triton offline-profile validation with real media and evidence manifest in tests/system/test_[name]_offline.py

### Implementation for User Story 1

- [ ] T024 [P] [US1] Create [Entity1] model in src/models/[entity1].py
- [ ] T025 [P] [US1] Create [Entity2] model in src/models/[entity2].py
- [ ] T026 [US1] Implement [Service] in src/services/[service].py (depends on T024, T025)
- [ ] T027 [US1] Implement [endpoint/feature] in src/[location]/[file].py
- [ ] T028 [US1] Add explicit validation, degradation and rejection paths
- [ ] T029 [US1] Add probe-backed telemetry and evidence lineage
- [ ] T030 [US1] Add rollback/fail-closed behavior and acceptance veto handling

**Checkpoint**: At this point, User Story 1 should be fully functional and testable independently

---

## Phase 4: User Story 2 - [Title] (Priority: P2)

**Goal**: [Brief description of what this story delivers]

**Independent Test**: [How to verify this story works on its own]

### Validation and Evidence for User Story 2 (MANDATORY)

- [ ] T025 [P] [US2] Unit test for [module/function] in tests/unit/test_[name].py
- [ ] T026 [P] [US2] Contract/schema test for [endpoint/event/artifact] in tests/contract/test_[name].py
- [ ] T027 [P] [US2] Integrity test for [timestamp/identity/pose/feature/failure semantics] in tests/integration/test_[name].py
- [ ] T028 [P] [US2] Native Triton live-profile validation with real media and evidence manifest in tests/system/test_[name]_live.py
- [ ] T029 [P] [US2] Native Triton offline-profile validation with real media and evidence manifest in tests/system/test_[name]_offline.py

### Implementation for User Story 2

- [ ] T030 [P] [US2] Create [Entity] model in src/models/[entity].py
- [ ] T031 [US2] Implement [Service] in src/services/[service].py
- [ ] T032 [US2] Implement [endpoint/feature] in src/[location]/[file].py
- [ ] T033 [US2] Integrate with User Story 1 components (if needed)

**Checkpoint**: At this point, User Stories 1 AND 2 should both work independently

---

## Phase 5: User Story 3 - [Title] (Priority: P3)

**Goal**: [Brief description of what this story delivers]

**Independent Test**: [How to verify this story works on its own]

### Validation and Evidence for User Story 3 (MANDATORY)

- [ ] T034 [P] [US3] Unit test for [module/function] in tests/unit/test_[name].py
- [ ] T035 [P] [US3] Contract/schema test for [endpoint/event/artifact] in tests/contract/test_[name].py
- [ ] T036 [P] [US3] Integrity test for [timestamp/identity/pose/feature/failure semantics] in tests/integration/test_[name].py
- [ ] T037 [P] [US3] Native Triton live-profile validation with real media and evidence manifest in tests/system/test_[name]_live.py
- [ ] T038 [P] [US3] Native Triton offline-profile validation with real media and evidence manifest in tests/system/test_[name]_offline.py

### Implementation for User Story 3

- [ ] T039 [P] [US3] Create [Entity] model in src/models/[entity].py
- [ ] T040 [US3] Implement [Service] in src/services/[service].py
- [ ] T041 [US3] Implement [endpoint/feature] in src/[location]/[file].py

**Checkpoint**: All user stories should now be independently functional

---

[Add more user story phases as needed, following the same pattern]

---

## Phase N: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

- [ ] TXXX [P] Create/update `docs/` Markdown files for all source files touched (constitution §docs/)
- [ ] TXXX [P] Create/update module README.md for every module worked on
- [ ] TXXX [P] Documentation updates in docs/ — verify Mermaid diagrams render correctly
- [ ] TXXX [P] Verify ALL diagram types used in every docs/ .md file (constitution §Mandatory Diagram Types)
- [ ] TXXX [P] Verify every diagram has introduction + detailed explanation (constitution §Diagram Explanation Requirements)
- [ ] TXXX [P] Verify every .md file has Related Documents cross-links (constitution §.md File Interconnection)
- [ ] TXXX [P] Validate all cross-reference links resolve to existing files — no broken links
- [ ] TXXX Code cleanup and refactoring
- [ ] TXXX Performance optimization across all stories
- [ ] TXXX [P] Verify production Triton-only routing, exactly one active
      endpoint profile, model/GPU health, and inactive-profile rejection
- [ ] TXXX [P] Verify timestamp, identity, lifecycle/ReID and pose stream
      integrity, including invalid/suppressed behavioral windows
- [ ] TXXX [P] Verify queue retry/DLQ/backpressure/RTSP failure behavior is
      bounded and observable
- [ ] TXXX [P] Verify API/WS/artifact/sequence contracts are versioned,
      serializers explicit, and migrations/idempotency covered
- [ ] TXXX [P] Run real live and offline validation in separate active-mode
      windows and store evidence manifests
- [ ] TXXX [P] Produce benchmark/statistical reports with real independent
      baseline/candidate runs, variance, confidence and failure accounting
- [ ] TXXX [P] Verify all maturity evidence is immutable, digest-addressed,
      lineage-complete, non-placeholder and free of temporary-path-only
      manifests
- [ ] TXXX [P] Verify no hidden xfails, unbounded technical debt, untracked
      runtime overrides, SQLite evidence paths or unsupported fallback
      execution paths remain
- [ ] TXXX [P] Verify rollback criteria, production freeze criteria and
      emergency remediation records for the accepted capability
- [ ] TXXX Security hardening
- [ ] TXXX Run quickstart.md validation

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3+)**: All depend on Foundational phase completion
  - User stories can then proceed in parallel (if staffed)
  - Or sequentially in priority order (P1 → P2 → P3)
- **Polish (Final Phase)**: Depends on all desired user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational (Phase 2) - No dependencies on other stories
- **User Story 2 (P2)**: Can start after Foundational (Phase 2) - May integrate with US1 but should be independently testable
- **User Story 3 (P3)**: Can start after Foundational (Phase 2) - May integrate with US1/US2 but should be independently testable

### Within Each User Story

- Validation and evidence gates MUST be defined before a story is claimed complete
- Models before services
- Services before endpoints
- Core implementation before integration
- Story complete before moving to next priority

### Parallel Opportunities

- All Setup tasks marked [P] can run in parallel
- All Foundational tasks marked [P] can run in parallel (within Phase 2)
- Once Foundational phase completes, all user stories can start in parallel (if team capacity allows)
- All tests for a user story marked [P] can run in parallel
- Models within a story marked [P] can run in parallel
- Different user stories can be worked on in parallel by different team members

---

## Parallel Example: User Story 1

```bash
# Launch all tests for User Story 1 together (if tests requested):
Task: "Contract test for [endpoint] in tests/contract/test_[name].py"
Task: "Integration test for [user journey] in tests/integration/test_[name].py"

# Launch all models for User Story 1 together:
Task: "Create [Entity1] model in src/models/[entity1].py"
Task: "Create [Entity2] model in src/models/[entity2].py"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL - blocks all stories)
3. Complete Phase 3: User Story 1
4. **STOP and VALIDATE**: Test User Story 1 independently
5. Deploy/demo if ready

### Incremental Delivery

1. Complete Setup + Foundational → Foundation ready
2. Add User Story 1 → Test independently → Deploy/Demo (MVP!)
3. Add User Story 2 → Test independently → Deploy/Demo
4. Add User Story 3 → Test independently → Deploy/Demo
5. Each story adds value without breaking previous stories

### Parallel Team Strategy

With multiple developers:

1. Team completes Setup + Foundational together
2. Once Foundational is done:
   - Developer A: User Story 1
   - Developer B: User Story 2
   - Developer C: User Story 3
3. Stories complete and integrate independently

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Validate nominal, failure, integrity, and degraded cases before acceptance
- Every source file change MUST include its corresponding `docs/` .md file
- Module README.md MUST be created/updated after finishing work on a module
- Stop at any checkpoint to validate story independently
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence
