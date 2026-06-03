# Cycle 15.B2 Four-Shard Design-Proof Investigation

**Last updated:** 2026-06-03

**Status:** PHASE A STARTED. No four-shard runtime implementation, production
benchmark, acceptance, rejection, skip, or closure exists yet.

**Streaming compatibility:** `offline-only`. Four-shard video partitioning
depends on whole-file access and parent/shard lifecycle coordination, so it
MUST NOT run on RTSP, RTSPS, or any live stream profile.

## Problem Statement

Cycle 15.B2 evaluates the higher-parallelism four-shard design proof. It is not
assumed better than two-shard: it has more boundary surfaces, more context-only
frames, and more parent/shard lifecycle risk. It must be measured and compared
against 15.B1 before any implementation decision.

This investigation authorizes only deterministic dry-run planning.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Parent investigation | `docs/cycle_15b_video_sharding_design_proof_investigation.md` | Defines the sharding safety gate before implementation. |
| Cycle 15 result | `docs/cycle_15_cuda_shared_memory_vs_sharding_results.md` | Promotes sharding to design proof and rejects immediate shared-memory work. |
| Shard planner | `tools/prod/prod_plan_video_shards.py` | Computes deterministic shard ownership and context-only frames. |
| Probe wrapper | `tools/prod/prod_run_cycle15b_shard_design_probe.sh` | Reproducible production dry-run wrapper. |
| Unit test | `backend/tests/unit/pipeline/test_prod_plan_video_shards.py` | Verifies four-shard ranges and coverage math. |
| Offline task code | `backend/apps/video_analysis/tasks.py` | Future implementation would have to split this lifecycle safely. |
| Constitution | `.specify/memory/constitution.md` | Section 8.6 forbids sharding on live profiles. |

## Hypothesis

Four-shard offline inference can be considered only if its added boundary risk
is justified after two-shard proof. The dry-run must show exact authoritative
coverage and bounded context overhead before any runtime implementation.

| Requirement | Four-shard expected proof |
|---|---|
| Authoritative coverage | Every frame index `0..4540` has exactly one authoritative shard. |
| Pre-roll context | Shards 1, 2, and 3 each decode 32 context frames before their starts. |
| Context-only rows | Context frames are non-authoritative and cannot persist DB rows. |
| Live safety | Scenario is explicitly `offline-only` and disabled for live streams. |

## Risk Assessment

| Risk | Severity | Control |
|---|---|---|
| More identity-boundary joins | High | Four-shard implementation blocked until two-shard stitching proof exists. |
| More duplicate context | Medium | Planner must quantify context-only frames and overhead. |
| Higher DB contention | High | Future benchmark must capture PostgreSQL and Redis wall separately. |
| Live stream collapse | High | Keep scenario `offline-only`; live profile must not inherit it. |

## Rollback Strategy

The design-proof helper is read-only. Runtime rollback is not needed unless a
future implementation lands. Future runtime rollback must restore accepted
single-job Cycle 14.B2 batch `16` behavior.

## Acceptance Criteria

Four-shard design proof can advance only if:

| Gate | Requirement |
|---|---|
| Dry-run proof | Production dry-run emits deterministic ranges and exact coverage. |
| Unit proof | Local unit tests pass for `4541` frames and `32` pre-roll frames. |
| Direct comparison | 15.B2 is compared to 15.B1 for context overhead and boundary count. |
| Implementation plan | Track stitching, DB idempotency, and parent/shard lifecycle are specified before code. |

No performance acceptance exists until a future full production benchmark runs.
