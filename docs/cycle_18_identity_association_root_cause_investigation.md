# Cycle 18 Identity Association Root-Cause Investigation

**Last updated:** 2026-06-04

**Status:** `ROOT_CAUSE_CONFIRMED` / `PROBE_ONLY` /
`NO_RUNTIME_CANDIDATE_SELECTED` /
`NO_DECISION_PRODUCTION_BENCHMARK_REQUIRED`.

**Streaming compatibility:** `offline-only`. This investigation concerns
finite-file shard boundaries. It does not authorize tracklet buffering,
cross-frame windows, Redis boundary state, or parent coordination on the live
profile.

## Problem Statement

Cycle 18 was blocked because the failed Cycle 15.B1 sharded candidates showed
near-perfect detection geometry but very low track-sensitive baseline
agreement. The earlier investigation treated that result as one identity
failure. This deeper audit proves that it contains two separate blockers:

1. the evaluator compares `tracking_id_text` by raw string equality across
   independent tracker runs, even though local track labels are source-scoped
   opaque namespaces; and
2. after removing that namespace error with a one-to-one global assignment,
   the sharded candidate still has real fragmentation and merge ambiguity.

Correcting the evaluation contract does not make the current sharding runtime
acceptable. It only measures the remaining identity problem more accurately.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Constitution | `.specify/memory/constitution.md` §4.1 / §4.3 / §4.6 | Defines local versus canonical identity and the independent-run association doctrine. |
| Constitution | `.specify/memory/constitution.md` §12.5 / §12.6 | Reserves optimization decisions for completed end-to-end production benchmarks. |
| Cycle doc | `docs/cycle_18_redis_boundary_state_cache_investigation.md` | Records prior Cycle 18 contract evidence and the absent child identity state. |
| Ledger | `docs/agent_19_cycle_18_turn.md` | Records Agent 19 scope, tasks, production lock state, and validation results. |
| Runtime | `backend/apps/video_analysis/services/offline_sharding.py` | Defines shard requests, boundary summaries, and the current geometry-vote parent track map. |
| Runtime | `backend/apps/video_analysis/tasks.py` | Defines independent shard tracker execution, pseudo-embedding use during tracking, boundary-summary write, and post-processing embedding flow. |
| Runtime | `backend/apps/tracking/embeddings.py` | Defines real crop-embedding extraction and deterministic fallback behavior. |
| Existing evaluator | `tools/prod/prod_compare_benchmark_accuracy.py` | Loads and compares persisted `tracking_id_text` values. |
| Existing stitching probe | `tools/prod/prod_analyze_cycle15b1_stitching.py` | Separates geometry agreement from raw track-label agreement. |
| New evaluator | `tools/prod/prod_probe_cycle18_label_invariant_tracking.py` | Performs read-only maximum-weight one-to-one assignment across local label namespaces. |
| New evaluator test | `backend/tests/unit/pipeline/test_prod_probe_cycle18_label_invariant_tracking.py` | Proves namespace recovery, residual fragmentation reporting, and no runtime decision authority. |
| Production summary | `docs/architecture/cycle18_label_invariant_tracking_probe.production_summary.json` | Preserves production job IDs, measured metrics, remote evidence paths, hashes, and the no-selection result. |
| Existing candidate summary | `docs/architecture/cycle18_identity_candidate_probe.production_summary.json` | Proves child shards retained no embedding, alias, lifecycle, canonical-ID, ReID-score, or ReID-decision rows. |
| Benchmark authority | `docs/production_inference_benchmark.md` | Records the completed Cycle 15 baseline and sharded candidate performance results. |
| Coordination | `docs/four_agent_cycle_coordination_board.md` | Keeps Agent 19 in the Cycle 18 contract/evidence lane and forbids runtime mutation or sharding enablement. |

## Primary-Source Best-Practice Review

| Primary source | Finding used in Cycle 18 | Adopted rule |
|---|---|---|
| [HOTA paper](https://arxiv.org/abs/2009.07736) | Tracking evaluation must expose detection and association quality instead of allowing one to conceal the other. | Report geometry/detection agreement separately from association agreement and fragmentation. |
| [TrackEval official repository](https://github.com/JonathonLuiten/TrackEval) | The official evaluation toolkit implements HOTA, CLEAR, and Identity metric families, including AssA, IDF1, and fragmentation. | A future ground-truth-backed acceptance package must report HOTA/AssA, IDF1, ID switches, and fragmentation. |
| [SciPy `linear_sum_assignment`](https://docs.scipy.org/doc/scipy-1.13.0/reference/generated/scipy.optimize.linear_sum_assignment.html) | Rectangular maximum-weight bipartite assignment provides a deterministic one-to-one mapping between independent label namespaces. | Cross-run diagnostic comparison uses a maximum-weight one-to-one assignment, not raw local-ID equality. |
| [Deep SORT](https://arxiv.org/abs/1703.07402) | Appearance information improves association through occlusion and reduces identity switches. | Boundary state must carry governed appearance evidence; geometry alone cannot authorize a canonical merge. |
| [BoT-SORT](https://arxiv.org/abs/2206.14651) | Robust association combines motion and appearance information. | A future shard association candidate must combine temporal/motion and appearance evidence. |
| [ByteTrack](https://www.ecva.net/papers/eccv_2022/papers_ECCV/html/315_ECCV_2022_paper.php) | Discarding low-confidence observations can fragment trajectories. | Boundary state must retain bounded low-confidence continuity evidence when policy permits it. |
| [Tracking by Associating Clips](https://www.ecva.net/papers/eccv_2022/papers_ECCV/html/5114_ECCV_2022_paper.php) | Multi-frame clip/tracklet association is more robust to interruptions than only adjacent-frame matching. | Shard association must reason over bounded tracklets, not one boundary box or one vote. |

## Root-Cause Tree

### Root Cause 1: Invalid cross-run label comparison

The current baseline-agreement helper loads `tracking_id_text` and requires the
same raw label for a track-sensitive match. Independent tracker instances reset
their local label namespaces, so equal numbers are not identity proof and
different numbers are not automatically identity failure. This violates the
canonical/local distinction already required by constitution §4.1.

The production label-invariant probe confirms the defect. In shard 1, minimum
raw-label F1 was `2.917%`; maximum-weight one-to-one assignment raised the
minimum to `79.876%`. The evaluator therefore materially overstated identity
failure when it treated raw local labels as canonical.

### Root Cause 2: Missing boundary identity state

The current shard request sets `skip_embeddings=True`. The current boundary
summary carries track label, box geometry, and confidence, but no governed
appearance prototype, temporal motion state, lifecycle evidence, feature
version, ambiguity margin, or canonical identity decision provenance.

The earlier production identity-candidate probe found zero child embeddings,
aliases, lifecycle events, canonical IDs, ReID scores, and ReID decisions. A
Redis cache cannot recover fields the producer never generated.

### Root Cause 3: Geometry-vote association is under-specified

The current parent merge gives first-shard IDs pass-through authority and maps
later shard tracks through best-IoU or majority-vote geometry. This does not
provide enough evidence for crowded, crossing, occluded, entering, or exiting
people. It also allows a whole child track to inherit a decision from a small
boundary vote set.

### Root Cause 4: Real association loss remains after label normalization

The label-invariant production probe did not erase the failure:

| Scope | Minimum geometry F1 | Minimum raw-label F1 | Minimum global-assignment F1 | Remaining association gap |
|---|---:|---:|---:|---:|
| All frames | `99.802%` | `53.730%` | `69.830%` | `29.972 pp` |
| Shard 1 authoritative | `100.000%` | `2.917%` | `79.876%` | `20.124 pp` |

Per-model shard-1 results:

| Model | Geometry F1 | Raw-label F1 | Global-assignment F1 | Residual gap | Fragmented baseline tracks | Merged candidate tracks |
|---|---:|---:|---:|---:|---:|---:|
| `attention_tracking` | `100.000%` | `2.917%` | `90.565%` | `9.435 pp` | `7` | `7` |
| `hand_raising` | `100.000%` | `3.984%` | `88.103%` | `11.897 pp` | `6` | `6` |
| `person_detection` | `100.000%` | `20.380%` | `79.876%` | `20.124 pp` | `18` | `17` |
| `sitting_standing` | `100.000%` | `4.692%` | `91.290%` | `8.710 pp` | `7` | `7` |

The evidence confirms both namespace mismatch and real fragmentation/merge
ambiguity. Neither diagnosis may be hidden by the other.

## Adopted Cycle 18 Best-Practice Contract

1. Local track IDs are source-scoped opaque labels. They are never canonical
   identity merely because their string or integer values match.
2. Independent-run or cross-shard comparison first performs observation
   matching, then a deterministic one-to-one global association.
3. Geometry/detection quality and identity/association quality are reported
   separately.
4. A single-run baseline is a reproducibility proxy, not human-labeled
   identity ground truth.
5. A future boundary producer emits bounded tracklets containing temporal
   range, motion history/state, governed appearance prototypes, observation
   quality, lifecycle state, source scope, and feature/model provenance.
6. Candidate associations use hard scope/time gates and a globally one-to-one
   assignment over documented motion, geometry, appearance, and lifecycle
   costs.
7. Ambiguous or conflicting associations remain unresolved. Offset/new label
   allocation is namespace management, not proof of a new person.
8. A future production decision reports HOTA/AssA, IDF1, ID switches,
   fragmentation, eligible/matched/unresolved/collision counts, DB/model
   parity, throughput, GPU, memory, and evidence lineage. Ground-truth-free
   proxy metrics cannot alone authorize scientific identity acceptance.

## Applied Changes

| Change | Application |
|---|---|
| Label-invariant evaluator | `tools/prod/prod_probe_cycle18_label_invariant_tracking.py` |
| Focused evaluator tests | `backend/tests/unit/pipeline/test_prod_probe_cycle18_label_invariant_tracking.py` |
| Production evidence summary | `docs/architecture/cycle18_label_invariant_tracking_probe.production_summary.json` |
| Binding governance | `.specify/memory/constitution.md` §4.6 and §14.25 |
| Agent execution guidance | `AGENTS.md` identity-association binding section |
| Planning propagation | `.specify/templates/plan-template.md`, `.specify/templates/spec-template.md`, and `.specify/templates/tasks-template.md` |

## Production Probe Evidence

| Field | Value |
|---|---|
| Baseline replay/job | `cycle15b-pre-shard-baseline-20260603T193531Z` / `74561b05-105f-4ca8-aeaf-f510f4f802de` |
| Candidate replay/job | `cycle15b1c2-majority-vote-20260603T223932Z` / `78388c2c-d7f5-42b7-afa4-321216d23b11` |
| Evidence directory | `/home/bamby/grad_project/backend/logs/cycle18-label-invariant-probe-20260604T163806Z` |
| JSON SHA-256 | `689a32083b17351ccaebf7d65372c2cab9f0e44380fb933b33e217277cd7bc7f` |
| Markdown SHA-256 | `e31baf5a07d2556b3b367638eca3cdbccd1a7fb78542e15efb71a4f2c661e771` |
| Decision authority | `PROBE_ONLY` |
| Runtime writes | None |
| Runtime candidate selected | `false` |

This probe reuses already completed production jobs. It is not a new
end-to-end candidate benchmark and cannot accept, reject, close, skip, or
deprioritize Cycle 18 under constitution §12.5/§12.6.

## Required Future Runtime Candidate

A future runtime candidate is eligible for review only after it defines and
tests:

| Required element | Minimum contract |
|---|---|
| Namespaced tracklet key | Parent job, child job, shard index, camera/source, and local track ID |
| Temporal evidence | First/last source timestamps, frame range, bounded observation history, and gap state |
| Motion evidence | Versioned state/history with documented coordinate system and prediction semantics |
| Appearance evidence | Bounded prototype set, model/version/digest, quality, and distance semantics |
| Lifecycle evidence | Active/occluded/lost/ended/unresolved state with transition reason |
| Candidate edges | Cost components, gates, thresholds, ambiguity margin, and rejection reason |
| Global decision | One-to-one assignment, unresolved conflicts, provenance, and append-only decision record |
| Failure behavior | Missing or stale evidence fails unresolved; no forced merge and no silent fallback |
| Streaming guard | Offline profile only; live profile explicitly disables the candidate |

## Step Log

| Step | Date | Action | Result |
|---|---|---|---|
| 1 | 2026-06-04 | Re-read governance, board, Cycle 18 evidence, and current runtime/evaluator code. | Confirmed Agent 19 remains contract/evidence only and the benchmark lock is not held. |
| 2 | 2026-06-04 | Reviewed primary sources for MOT evaluation and association. | Adopted separated detection/association metrics, one-to-one assignment, motion-plus-appearance tracklets, and unresolved ambiguity. |
| 3 | 2026-06-04 | Implemented a read-only label-invariant evaluator. | Added deterministic global assignment without runtime or DB writes. |
| 4 | 2026-06-04 | Ran focused local tests and compile/whitespace checks. | `3` focused tests passed; compile and whitespace checks passed. |
| 5 | 2026-06-04 | Started the production probe from `/tmp`. | First attempt failed before data access because the repository stitching helper was not on `PYTHONPATH`. |
| 6 | 2026-06-04 | Re-ran with `tools/prod/prod_analyze_cycle15b1_stitching.py` resolvable on `PYTHONPATH`. | Probe completed and wrote hashed read-only production evidence. |
| 7 | 2026-06-04 | Compared raw-label, global-assignment, and geometry metrics. | Confirmed both the evaluation-contract defect and residual runtime association gap. |
| 8 | 2026-06-04 | Propagated adopted rules into governance and planning templates. | Constitution v2.9.0 and execution guidance now prohibit raw local-ID equality as cross-run identity proof. |
| 9 | 2026-06-04 | Integrated the investigation, evaluator, tests, and production summary into the Cycle 18 contract gate. | Gate passed with `28` artifacts, `6` visible JSON files, `157` resolvable references, and `25` resolvable line references. |
| 10 | 2026-06-04 | Ran the final local and production-evidence audit. | `39` focused tests passed, `260` reading-order docs resolved, `5/5` Mermaid blocks rendered, compile/JSON/whitespace checks passed, and production evidence hashes matched. |

## Final Decision

`NO_RUNTIME_CANDIDATE_SELECTED` /
`NO_DECISION_PRODUCTION_BENCHMARK_REQUIRED` /
`CLOSURE_BLOCKED_NO_CYCLE18_RUNTIME_BENCHMARK`.

The current raw-label comparison is invalid for cross-run or cross-shard
identity correctness and must not be used as the sole identity gate again.
However, the label-invariant production probe still measures substantial
association loss, fragmentation, and merge ambiguity. The current geometry
vote, the Redis boundary-cache idea, and the existing child evidence are not
eligible runtime candidates. Cycle 18 remains open and blocked until a reviewed
boundary tracklet-state producer exists and a later completed native-Linux RTX
5090 end-to-end production benchmark creates a constitution §12.6 decision.
