# Requirement And Acceptance Coverage Matrix

**Date**: 2026-06-08
**Purpose**: Prove that each explicit requirement and measurable outcome has an
implementation cycle and executable task coverage. This is a planning
traceability artifact, not implementation evidence.

## Functional Requirement Coverage

| Requirement | Owning cycle(s) | Primary task coverage |
|---|---|---|
| FR-001 versioned evidence envelope | 015.1 | T037-T045 |
| FR-002 immutable route snapshot | 015.0 | T019-T021, T026-T029 |
| FR-003 explainer for every active route | 015.3 | T071-T087 |
| FR-004 fast/deep path distinction | 015.3, 015.8 | T071, T079, T161-T177 |
| FR-005 deterministic fast explanation | 015.3 | T072-T082 |
| FR-006 deep attribution/perturbation/evaluation | 015.8 | T161-T177 |
| FR-007 model-specific strategies | 015.3, 015.8 | T072-T078, T163-T167 |
| FR-008 governed raw/compact score summaries | 015.1, 015.2, 015.3 | T037-T045, T054-T060, T072-T078 |
| FR-009 governed signal catalog | 015.1 | T038-T046 |
| FR-010 temporal anomaly evidence | 015.4 | T091-T104 |
| FR-011 non-accusatory review-priority score | 015.5 | T108-T123 |
| FR-012 reconstructable score contributions | 015.5 | T109-T118 |
| FR-013 contextual baselines and drift protection | 015.2, 015.5, 015.6 | T054-T067, T108-T123, T127-T139 |
| FR-014 calibrated confidence | 015.2 | T054-T067 |
| FR-015 uncertainty/conformal outputs | 015.6 | T127-T139 |
| FR-016 source-preserving cross-model fusion | 015.7 | T143-T157 |
| FR-017 identity-gated peer context | 015.7 | T145, T150-T157 |
| FR-018 governed review feedback | 015.9 | T181-T192 |
| FR-019 artifact access/audit/retention | 015.8, 015.9, 015.11 | T161-T173, T183-T188, T222-T233 |
| FR-020 bounded/idempotent/reconciled deep tasks | 015.8 | T168-T173 |
| FR-021 bounded live fast-XAI state | 015.4, 015.11 | T091-T104, T224-T233 |
| FR-022 all accepted analytical rendering uses WebGL2 | 015.10 | T196-T215 |
| FR-023 persistent contexts/typed arrays/incremental LOD | 015.10 | T196-T211 |
| FR-024 explicit WebGL unavailable state | 015.10 | T196-T211 |
| FR-025 figure download with lineage | 015.10 | T202, T207, T210-T215 |
| FR-026 modular interfaces and no parallel service | Setup, 015.1, 015.3, 015.5 | T011, T037-T045, T071-T079, T108-T115 |
| FR-027 configuration-governed operational values | Setup, all cycles | T002, T011, each investigation/preflight task |
| FR-028 full evidence packet per atomic cycle | All cycles | T016-T250 cycle investigation/figure/benchmark/result/ledger tasks |
| FR-029 one causal change and independent rollback | All cycles | `atomic-cycles.md`, each cycle investigation/rollback task |
| FR-030 block placeholders/drift/empty layers/unsafe language | Setup, 015.0, 015.11, 015.12 | T004-T006, T024-T033, T220-T250 |

## Success Criterion Coverage

| Success criterion | Owning cycle(s) | Primary task coverage |
|---|---|---|
| SC-001 all active routes have explainers | 015.3 | T071-T087 |
| SC-002 exact score reconstruction | 015.5 | T109-T123 |
| SC-003 invalid evidence withholds score | 015.5, 015.6 | T110-T118, T127-T139 |
| SC-004 calibration evaluation | 015.2 | T054-T067 |
| SC-005 fidelity/stability/sanity evaluation | 015.8 | T163-T177 |
| SC-006 bounded fast-path overhead | 015.3-015.7, 015.11 | cycle benchmarks and T229-T233 |
| SC-007 deep XAI critical-path isolation | 015.8 | T168-T177 |
| SC-008 bounded live soak/reconnect | 015.4, 015.11 | T095-T104, T224-T233 |
| SC-009 stride-1 production evidence | All cycles | each `prod_run_xai_cycle015_*` and ledger task |
| SC-010 WebGL performance/context budgets | 015.10 | T196-T215 |
| SC-011 all accepted figures use WebGL2 | 015.10 | T203-T215 |
| SC-012 versioned/idempotent/digest-backed records | 015.0-015.9 | persistence/contracts/integration tasks |
| SC-013 review feedback cannot self-modify | 015.9 | T183-T192 |
| SC-014 no accusation semantics | Setup, 015.0, 015.5, 015.11 | T004, T024-T026, T108-T123, T224-T233 |
| SC-015 runtime reconciliation | 015.0, 015.11, 015.12 | T025-T033, T220-T233, T238-T250 |
| SC-016 Figure Planner/Implementer per cycle | All cycles | paired figure-plan/implementation tasks |
| SC-017 every run in benchmark ledger | All cycles | T033, T050, T067, T087, T104, T123, T139, T157, T177, T192, T215, T233, T250 |
| SC-018 final canary fingerprint/evidence | 015.12 | T234-T250 |

## User Story Coverage

| User story | Primary cycles | Independent acceptance evidence |
|---|---|---|
| US1 Explain a model output | 015.0-015.3 | route truth, calibration, complete fast-adapter coverage |
| US2 Diagnose temporal review priority | 015.4-015.6 | bounded temporal evidence, reconstructable score, uncertainty/coverage |
| US3 Analyze cross-model/context evidence | 015.7 | identity-gated fusion and explanation graph |
| US4 Request deep XAI and review evidence | 015.8-015.9 | isolated evaluated artifacts and governed feedback |
| US5 Responsive WebGL workbench | 015.10 | WebGL2 renderer and representative/stress benchmark |
| US6 Production/scientific integrity | Every cycle, 015.11-015.12 | benchmark/figure/ledger/rollback packets and final canary |

## Coverage Result

- Functional requirements covered: `30/30`
- Success criteria covered: `18/18`
- User stories covered: `6/6`
- Atomic cycles with explicit benchmark, figures, decision, and rollback:
  `13/13`
