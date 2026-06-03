# Cycle 15.B1 Two-Shard Design-Proof Investigation

**Last updated:** 2026-06-03

**Status:** DESIGN PROOF PASSED / RUNTIME IMPLEMENTATION STILL BLOCKED. No
two-shard runtime implementation, production benchmark, acceptance, rejection,
skip, or closure exists yet. Results are recorded in
`docs/cycle_15b_shard_design_probe_results.md`.

**Streaming compatibility:** `offline-only`. Video sharding relies on whole-file
boundaries and parent/shard coordination, so it MUST NOT run on RTSP, RTSPS, or
any live stream profile.

## Problem Statement

Cycle 15.B1 tests whether a two-shard offline plan can preserve exact
authoritative frame ownership while giving each non-first shard enough pre-roll
context to warm tracking state. It is the lowest-risk sharding scenario because
Cycle 15 Phase A estimated only `0.704691 %` context overhead for two shards.

This investigation does not authorize multi-process inference. It authorizes a
design-proof helper only.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Parent investigation | `docs/cycle_15b_video_sharding_design_proof_investigation.md` | Defines the sharding safety gate before implementation. |
| Cycle 15 result | `docs/cycle_15_cuda_shared_memory_vs_sharding_results.md` | Promotes sharding to design proof and rejects immediate shared-memory work. |
| Shard planner | `tools/prod/prod_plan_video_shards.py` | Computes deterministic shard ownership and context-only frames. |
| Probe wrapper | `tools/prod/prod_run_cycle15b_shard_design_probe.sh` | Reproducible production dry-run wrapper. |
| Unit test | `backend/tests/unit/pipeline/test_prod_plan_video_shards.py` | Verifies two-shard ranges and coverage math. |
| Offline task code | `backend/apps/video_analysis/tasks.py` | Future implementation would have to split this lifecycle safely. |
| Constitution | `.specify/memory/constitution.md` | Section 8.6 forbids sharding on live profiles. |

## Hypothesis

Two-shard offline inference can become the first implementable sharding
candidate if the dry-run proves:

| Requirement | Two-shard expected proof |
|---|---|
| Authoritative coverage | Every frame index `0..4540` has exactly one authoritative shard. |
| Pre-roll context | Shard 1 decodes 32 context frames before its authoritative start. |
| Context-only rows | Context frames are non-authoritative and cannot persist DB rows. |
| Live safety | Scenario is explicitly `offline-only` and disabled for live streams. |

## Risk Assessment

| Risk | Severity | Control |
|---|---|---|
| Duplicate persisted frames | High | Persist only authoritative ranges; context frames are read-only. |
| Track split at boundary | High | Use pre-roll context and require future stitch proof before implementation. |
| Parent job completes early | High | Future implementation needs parent/shard terminal-state coordination. |
| Live stream collapse | High | Keep scenario `offline-only`; live profile must not inherit it. |

## Rollback Strategy

The design-proof helper is read-only. Runtime rollback is not needed unless a
future implementation lands. Future runtime rollback must restore accepted
single-job Cycle 14.B2 batch `16` behavior.

## Acceptance Criteria

Two-shard design proof can advance to implementation planning only if:

| Gate | Requirement |
|---|---|
| Dry-run proof | Production dry-run emits deterministic ranges and exact coverage. |
| Unit proof | Local unit tests pass for `4541` frames and `32` pre-roll frames. |
| Streaming proof | Docs and env plan mark the candidate `offline-only`. |
| Implementation plan | DB idempotency and track stitching are specified before code. |

No performance acceptance exists until a future full production benchmark runs.
