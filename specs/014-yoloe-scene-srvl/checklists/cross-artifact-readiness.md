# Cross-Artifact Requirements Readiness Checklist: YOLOE Scene Segmentation And SRVL

**Purpose**: Validate that `spec.md`, `plan.md`, and `tasks.md` are mutually
consistent and ready for implementation planning review.  
**Created**: 2026-06-07  
**Feature**: [spec.md](../spec.md)  
**Audience**: Reviewer / PR readiness  
**Depth**: Standard requirements-quality gate  

## Requirement Completeness

- [x] CHK001 - Are all functional requirements represented by at least one task with explicit traceability? [Completeness, Spec §FR-001..FR-033, Tasks §Requirements Coverage Index]
- [x] CHK002 - Are all measurable success criteria represented by at least one task or evidence task? [Completeness, Spec §SC-001..SC-022, Tasks §Requirements Coverage Index]
- [x] CHK003 - Are the four prioritized user stories reflected as independently testable task phases? [Coverage, Spec §User Stories, Tasks §Phases 3-6]
- [x] CHK004 - Are mandatory runtime scenarios for offline, live-disabled, frontend/backend wiring, inference, identity, deployment, replay, benchmark, reconciliation, and lineage represented in tasks? [Coverage, Spec §Mandatory Runtime Scenarios, Tasks §Phases 2-7]
- [x] CHK005 - Are data entities in the specification and data model covered by schema, migration, persistence, and artifact tasks? [Completeness, Spec §Key Entities, Data Model §Entity Overview, Tasks T011-T015]

## Requirement Clarity

- [x] CHK006 - Is prompt-locked YOLOE export defined with checkpoint, ordered classes, digest, manifest, and stale-artifact rejection criteria? [Clarity, Spec §FR-002..FR-005, Plan §YOLOE Output And Configuration Authority, Tasks T020-T022/T041-T054]
- [x] CHK007 - Are YOLOE output fields and unavailable-output rules specific enough to prevent dropping boxes, class IDs, confidences, masks, shape metadata, or raw refs? [Clarity, Spec §FR-006/FR-031, Contracts §API Scene Artifacts, Tasks T035-T045]
- [x] CHK008 - Are vague overlap/recovery/renderer terms resolved into configured thresholds, TTL/lookahead settings, and configured-resolution artifacts? [Clarity, Spec §FR-009/FR-017/FR-021, Plan §Budgets/SLOs, Tasks T046-T047/T066/T082]
- [x] CHK009 - Are live stream expectations explicitly excluded from V1 and represented by disabled/unavailable behavior rather than omitted? [Clarity, Spec §Live Stream Scenario, Plan §Runtime Scenarios, Tasks T018-T019]
- [x] CHK010 - Are operational values required to originate from `.env`/settings instead of hardcoded feature constants? [Clarity, Spec §FR-032/FR-033, Plan §YOLOE Output And Configuration Authority, Tasks T008-T010/T041/T064/T079]

## Requirement Consistency

- [x] CHK011 - Do `spec.md`, `plan.md`, and `tasks.md` consistently describe V1 as disabled-by-default and offline-only? [Consistency, Spec §FR-001, Plan §Summary/Runtime Scenarios, Tasks T018-T019/T037/T094/T120]
- [x] CHK012 - Do API, WebSocket, artifact, frontend, and helper contracts align with the task file targets? [Consistency, Contracts directory, Plan §Project Structure, Tasks T023-T027/T089-T108]
- [x] CHK013 - Do the task target paths align with the plan's declared task path authority? [Consistency, Plan §Task path authority, Tasks §Phases 1-7]
- [x] CHK014 - Are PostgreSQL-only persistence and source-control visibility requirements consistent across requirements, plan, and tasks? [Consistency, Spec §FR-028, Plan §Storage/Constitution Check, Tasks T012-T013/T116-T117]
- [x] CHK015 - Are identity-association rules consistent for mismatch recovery, including deterministic one-to-one assignment and unresolved ambiguity? [Consistency, Spec §FR-012/FR-013, Plan §Temporal/Identity Authority, Tasks T057/T063]

## Acceptance Criteria Quality

- [x] CHK016 - Are production benchmark acceptance criteria measurable with replay keys, job IDs, exact video, FPS, latency, GPU, DB/Redis, correctness, frontend, and unavailable-metric evidence? [Measurability, Spec §SC-012, Plan §Constitution Check, Tasks T091/T098-T107]
- [x] CHK017 - Are generated figure requirements tied to the same raw artifacts as the benchmark decision table and assigned planner/implementer ownership? [Measurability, Spec §SC-013, Plan §Constitution Check, Tasks T030-T031/T092/T101]
- [x] CHK018 - Are rollback criteria measurable for disabled scene/SRVL flags and existing offline pipeline behavior? [Measurability, Spec §SC-001/FR-029, Contracts §prod_rollback_yoloe_scene, Tasks T037/T094/T104-T105/T120]
- [x] CHK019 - Are frontend renderer criteria measurable with FPS, frame time, long tasks, context loss, memory growth, screenshots, and drop counters? [Measurability, Spec §SC-010/SC-011, Contracts §Frontend Metrics Contract, Tasks T075-T088/T093/T102-T103]
- [x] CHK020 - Are security/access criteria measurable for authenticated job-scoped scene artifacts? [Measurability, Spec §FR-030/SC-015, Contracts §API Scene Artifacts, Tasks T016-T017/T025/T096]

## Scenario And Edge Case Coverage

- [x] CHK021 - Are YOLOE zero/one/duplicate/low-confidence people and count mismatch scenarios covered by requirements and tasks? [Coverage, Spec §Edge Cases, Tasks T055-T064]
- [x] CHK022 - Are SRVL zero-object, one-object, invalid-coordinate, duplicate-ID, 128-object, above-128-object, top-k, thresholded, and heatmap-only cases covered? [Coverage, Spec §FR-019/SC-008, Tasks T069-T080]
- [x] CHK023 - Are missing Redis evidence, stale appearance evidence, absent vectors, and ambiguous matches covered as requirements-quality cases? [Coverage, Spec §Edge Cases, Tasks T056-T060/T062-T065]
- [x] CHK024 - Are visualization overload, latest-frame-wins, and source-frame-count preservation cases covered? [Coverage, Spec §FR-020/FR-021/SC-011, Tasks T072-T083]
- [x] CHK025 - Are configuration drift, stale prompt digest, wrong checkpoint, and prompt-after-export cases covered? [Coverage, Spec §Edge Cases/SC-002, Tasks T033-T034/T041-T054]

## Dependencies And Assumptions

- [x] CHK026 - Are the accepted V1 assumptions documented and reflected in setup/foundation tasks? [Assumption, Spec §Assumptions, Plan §Technical Context, Tasks T001-T032]
- [x] CHK027 - Are external/runtime dependencies documented with the correct authority split between local contract checks and production evidence? [Dependency, Plan §Technical Context/Target Platform, Quickstart §Local Contract Checks/Production Benchmark Flow, Tasks T089-T108]
- [x] CHK028 - Are future concurrency changes explicitly excluded from V1 acceptance unless a separate governed production benchmark exists? [Assumption, Spec §SC-020, Plan §Constitution Check, Tasks T119]
- [x] CHK029 - Are async/vector/orchestration governance conditions represented as conditional acceptance tasks instead of silent assumptions? [Assumption, Spec §SC-019, Plan §Constitution Check, Tasks T118]
- [x] CHK030 - Are no-sudo/no-Docker production helper constraints captured in requirements, plan, contracts, and tasks? [Dependency, Spec §Deployment Boundary, Plan §Target Platform, Contracts §Production Helper Commands, Tasks T032/T098-T105]

## Ambiguities And Conflicts

- [x] CHK031 - Are placeholder markers and clarification gaps absent from `spec.md`, `plan.md`, and `tasks.md`? [Ambiguity, Spec/Plan/Tasks]
- [x] CHK032 - Are ambiguous quality words avoided or tied to thresholds, counters, or configured values in requirement-bearing text? [Ambiguity, Spec §FR-009/FR-017/FR-020/FR-021, Plan §Budgets/SLOs]
- [x] CHK033 - Are task IDs sequential and formatted consistently with Spec Kit task checklist rules? [Traceability, Tasks §Format]
- [x] CHK034 - Are tasks without requirement or success-criterion traceability absent? [Traceability, Tasks §Requirements Coverage Index]
- [x] CHK035 - Are contradiction/recovery requirements consistently append-only and non-mutating across spec, data model, and tasks? [Conflict, Spec §FR-009/FR-016, Data Model §Persistence Rules, Tasks T039/T047/T060/T065]
