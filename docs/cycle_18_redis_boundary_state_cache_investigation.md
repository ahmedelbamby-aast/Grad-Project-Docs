# Cycle 18 Redis Boundary-State Cache Investigation

**Last updated:** 2026-06-05

**Status:** `CYCLE_18B_NOT_ACCEPTED` /
`CYCLE_18C_BENCHMARK_LOCK_HELD` /
`BENCHMARK_LOCK_HELD_FOR_CYCLE_18C`. The historical Agent 19 Phase A review remains
`NO_RUNTIME_CANDIDATE_SELECTED` for Redis boundary-state identity evidence. The
Agent 20 override production benchmark completed for the disabled-by-default
`one_to_one` boundary track-map candidate and rejected that candidate on
identity/model-agreement evidence. Cycle 18.B then ran the appearance-backed
packet association profile and rejected it on packet-validity, merge-readiness,
StudentTrack, model-agreement, and label-invariant identity evidence. Cycle
18.C is now staged locally only; Cycle 18 as a whole is not closed or complete.

**Streaming compatibility:** `offline-only` for any future sharding runtime.
Boundary state depends on finite shard boundaries and whole-file coordination.
It must not be enabled for RTSP, RTSPS, WHEP/WebRTC, or HLS live profiles.

## Problem Statement

Cycle 18 was originally staged as a Redis boundary-state cache for future video
sharding. Production evidence now changes its status: Cycle 15.B1, Cycle
15.B1.C1, and Cycle 15.B1.C2 all failed identity/model-agreement gates, and
Cycle 15.B2 is blocked. Redis must not be used as a shortcut around that
failure.

The safe Cycle 18 task is therefore contract design only: define what boundary
state would be required for a future sharding identity proof, without writing a
runtime Redis cache or changing the parent merge.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Doc | `docs/cycle_15b1_two_shard_runtime_investigation.md` | Records context `32`, context `256`, and majority-vote production failures. |
| Doc | `docs/cycle_15b_shard_design_probe_results.md` | Records why 15.B2 remains blocked after two-shard identity failures. |
| Doc | `docs/production_inference_benchmark.md` | Benchmark authority and Cycle 15.B1.C1/C2 evidence. |
| Doc | `docs/redis_broader_optimization_opportunities.md` | Original Redis boundary-state cache roadmap entry. |
| Doc | `docs/cycle_17_redis_streams_progress_sampling_investigation.md` | Redis discipline to reuse: bounded keys, fallback, counters, rollback. |
| File | `backend/apps/video_analysis/services/offline_sharding.py` | Current parent merge and track-map implementation. |
| File | `backend/apps/video_analysis/tasks.py` | Current incremental tracker state and boundary-summary write point. |
| File | `backend/apps/video_analysis/models.py` | Existing identity, alias, ReID-decision, and embedding fields available to a future proof. |
| File | `backend/apps/video_analysis/management/commands/cycle15b1_sharded_ingest.py` | Offline-only sharded runtime command. |
| File | `tools/prod/prod_analyze_cycle15b1_stitching.py` | Read-only identity-loss and oracle-relabel analysis. |
| Test | `backend/tests/unit/video_analysis/test_cycle15b1_shard_merge.py` | Current merge and majority-vote contract coverage. |
| Contract | `docs/architecture/cycle18_boundary_packet_v0.schema.json` | Machine-readable Phase A packet V0 design contract. |
| Example | `docs/architecture/cycle18_boundary_packet_v0.example.json` | Synthetic read-only packet proving unresolved identity is explicit. |
| Historical projection | `docs/architecture/cycle18_boundary_packet_v0.historical_cycle15b1c2.json` | Fail-closed projection using only retained Cycle 15.B1.C2 aggregate evidence. |
| Projection source | `docs/architecture/cycle18_boundary_packet_v0.historical_cycle15b1c2.source.json` | Structured retained-evidence manifest for reproducible local projection. |
| Projection helper | `tools/prod/prod_project_cycle18_boundary_packet.py` | Local-file-only generator that validates aggregate consistency before writing a packet. |
| Projection test | `backend/tests/unit/pipeline/test_prod_project_cycle18_boundary_packet.py` | Proves reproducibility and rejects inconsistent or unknown source fields. |
| Recovery auditor | `tools/prod/prod_audit_cycle18_recovered_evidence.py` | Audits a recovered local stitching export against retained scope and aggregates before projection. |
| Recovery test | `backend/tests/unit/pipeline/test_prod_audit_cycle18_recovered_evidence.py` | Proves scope mismatch and aggregate drift fail closed. |
| Contract gate | `tools/prod/prod_check_cycle18_boundary_contract.py` | Runs contract-artifact/reference, projection-drift, packet-validator, recovered-evidence fail-closed, structural drift, and optional external-schema checks in one local command. |
| Contract gate test | `backend/tests/unit/pipeline/test_prod_check_cycle18_boundary_contract.py` | Proves the local contract gate passes with the tracked artifacts. |
| Structural drift checker | `tools/prod/prod_check_cycle18_schema_validator_drift.py` | Compares all V0 object shapes and manual missing/unknown-field rejection without runtime access. |
| Structural drift test | `backend/tests/unit/pipeline/test_prod_check_cycle18_schema_validator_drift.py` | Proves the local structural drift corpus passes. |
| Doc reference checker | `tools/prod/prod_check_cycle18_doc_references.py` | Verifies repo-relative file and optional line references in Agent 19-owned docs plus only the bounded Agent 19 `AGENTS.md` paragraph. |
| Doc reference test | `backend/tests/unit/pipeline/test_prod_check_cycle18_doc_references.py` | Proves all four bounded reference sources resolve without scanning Agent 20. |
| Measured-value checker | `tools/prod/prod_check_cycle18_measured_benchmark_values.py` | Verifies the Cycle 18 measurement table exactly matches authoritative completed production values. |
| Measured-value test | `backend/tests/unit/pipeline/test_prod_check_cycle18_measured_benchmark_values.py` | Proves measured-value parity, drift rejection, and local-only isolation. |
| Identity-candidate probe | `tools/prod/prod_probe_cycle18_identity_candidates.py` | Reproducible read-only production probe comparing implementable boundary mappings and the baseline-label oracle upper bound. |
| Identity-candidate probe test | `backend/tests/unit/pipeline/test_prod_probe_cycle18_identity_candidates.py` | Proves deterministic policy maps and fail-closed candidate selection. |
| Production probe summary | `docs/architecture/cycle18_identity_candidate_probe.production_summary.json` | Tracked summary of the production probe evidence, source jobs, evidence hash, and no-selection result. |
| Deep root-cause investigation | `docs/cycle_18_identity_association_root_cause_investigation.md` | Separates invalid raw local-ID comparison from measured residual association loss and records adopted best practices. |
| Label-invariant tracking probe | `tools/prod/prod_probe_cycle18_label_invariant_tracking.py` | Performs read-only maximum-weight one-to-one association across independent local-label namespaces. |
| Label-invariant tracking tests | `backend/tests/unit/pipeline/test_prod_probe_cycle18_label_invariant_tracking.py` | Proves namespace recovery, residual fragmentation reporting, and no runtime decision authority. |
| Label-invariant production summary | `docs/architecture/cycle18_label_invariant_tracking_probe.production_summary.json` | Preserves production metrics, evidence hashes, and the two confirmed root causes. |
| Agent 20 override ledger | `docs/agent_20_cycle_18_override_turn.md` | Records the benchmark lock, release, production decision, rollback, and next action for the bounded override. |
| Agent 20 production summary | `docs/architecture/cycle18_one_to_one_trackmap.production_summary.json` | Preserves production metrics, evidence hashes, rollback proof, and the `NOT_ACCEPTED` decision for `one_to_one`. |
| Production benchmark log | `docs/production_inference_benchmark.md` | Records the completed Cycle 18 `one_to_one` production benchmark and section 12.6 decision table. |
| Validator | `tools/prod/prod_validate_cycle18_boundary_packet.py` | Standalone read-only packet, digest, bounds, and duplicate validator. |
| Test | `backend/tests/unit/pipeline/test_prod_validate_cycle18_boundary_packet.py` | Focused fail-closed validator contract coverage. |
| Figure generator | `tools/prod/prod_generate_cycle_figures.py` | Creates the required benchmark figure bundle and manifest from raw Cycle 18.B artifacts. |
| Figure generator test | `backend/tests/unit/pipeline/test_prod_generate_cycle_figures.py` | Proves strict unavailable-family handling and PNG/manifest/Markdown generation. |
| Runtime packet redesign | `backend/apps/video_analysis/services/offline_sharding.py` | Stages Cycle 18.C packet byte-budget sampling and inter-shard-edge selection. |
| Runtime packet tests | `backend/tests/unit/video_analysis/test_cycle15b1_shard_merge.py` | Proves packet validity under a tight byte budget and exclusion of terminal video edges. |
| File | `.gitignore` | Explicitly exposes the four CI-read contract/projection JSON files despite the blanket `*.json` ignore rule. |
| Turn ledger | `docs/agent_19_cycle_18_turn.md` | Agent 19 contract-only ownership and non-overlap boundaries. |
| Constitution | `.specify/memory/constitution.md` §8.6 | Streaming-source compatibility and live-profile restrictions. |
| Constitution | `.specify/memory/constitution.md` §12.5 / §12.6 | Production benchmark authority and decision table. |

## Why Runtime Implementation Is Blocked

| Blocker | Evidence | Consequence |
|---|---|---|
| Two-shard runtime is not correct. | Cycle 15.B1, 15.B1.C1, and 15.B1.C2 are `NOT ACCEPTED`. | Do not run 15.B2 or more sharded runtime variants. |
| Geometry was not the main failure. | Boundary probe showed geometry preserved while track-sensitive F1 collapsed. | Redis must target identity state, not only frame ownership. |
| Majority vote failed. | 15.B1.C2 mapped only `10/36` shard-1 tracks and increased `StudentTracks` to `64`. | Simple boundary voting is not enough. |
| Sharding is offline-only. | Constitution §8.6 forbids whole-file sharding on live streams. | Any future Cycle 18 runtime flag must stay disabled in live profiles. |

## Real Production Measurements Carried Into Cycle 18

Before the Agent 20 override, no Cycle 18 runtime candidate had been deployed
or benchmarked. These are real completed upstream Cycle 15 production
measurements that define Cycle 18's identity-state entry gate. They did not
create a Cycle 18 optimization decision; the pre-override decision statement
was `NO_DECISION_PRODUCTION_BENCHMARK_REQUIRED`. Exact values are
machine-checked against `docs/production_inference_benchmark.md:2945`,
`docs/production_inference_benchmark.md:3089`, and
`docs/production_inference_benchmark.md:3151` by
`tools/prod/prod_check_cycle18_measured_benchmark_values.py`.

### Production benchmark authority carried forward

| Candidate | Replay key | Parent or baseline job | Production decision |
|---|---|---|---|
| Accepted pre-shard baseline | `cycle15b-pre-shard-baseline-20260603T193531Z` | `74561b05-105f-4ca8-aeaf-f510f4f802de` | comparator only |
| 15.B1 context-32 | `cycle15b1-two-shard-runtime-repeat-20260603T211319Z` | `e602a0ca-6efc-4cb0-8d30-9466fe76287b` | `NOT ACCEPTED` |
| 15.B1.C1 context-256 | `cycle15b1c1-context256-20260603T222123Z` | `401498f1-d5e4-4b95-8a46-ad3fcbbc2c25` | `NOT ACCEPTED` |
| 15.B1.C2 majority vote | `cycle15b1c2-majority-vote-20260603T223932Z` | `78388c2c-d7f5-42b7-afa4-321216d23b11` | `NOT ACCEPTED` |

### Measured performance and identity results

| Metric | Accepted pre-shard baseline | 15.B1 context-32 | 15.B1.C1 context-256 | 15.B1.C2 majority vote |
|---|---:|---:|---:|---:|
| DB-completed FPS | `5.619787` | `7.866851` | `7.905` | `7.833` |
| DB completed elapsed | `808.038 s` | `577.232 s` | `574.481 s` | `579.761 s` |
| Step 2 frame wall | `467.449833 s` | `233.037627 s` | `242.392 s` | `243.909 s` |
| Step 2 through-pose wall | `641.154064 s` | `324.763211 s` | `338.209 s` | `340.942 s` |
| GPU average utilization | `11.846 %` | `17.495 %` | `17.636 %` | `17.891 %` |
| GPU peak utilization | `57.000 %` | `89.000 %` | `86.000 %` | `87.000 %` |
| Behavior RTT mean | `83.530 ms` | `89.718 ms` | `89.717 ms` | `90.372 ms` |
| StudentTracks | `53` | `52` | `52` | `64` |

The sharded candidates measured substantial throughput and GPU-utilization
improvements, but every candidate failed identity correctness. The exact
baseline-agreement F1@IoU0.5 measurements were:

| Candidate | attention_tracking | hand_raising | person_detection | sitting_standing |
|---|---:|---:|---:|---:|
| 15.B1 context-32 | `59.473 %` | `60.700 %` | `61.387 %` | `53.455 %` |
| 15.B1.C1 context-256 | `58.997 %` | `61.109 %` | `61.767 %` | `53.730 %` |
| 15.B1.C2 majority vote | `58.997 %` | `61.109 %` | `60.933 %` | `53.730 %` |

The retained majority-vote diagnostics provide the only locally reproducible
aggregate boundary evidence for the historical V0 projection:

| Measured boundary diagnostic | Value | Cycle 18 consequence |
|---|---:|---|
| C2 shard-1 source tracks | `36` | A full recovered packet must contain `36` source-track records. |
| C2 mapped to existing parent | `10` | Mapping coverage is insufficient for an identity proof. |
| C2 offset fallbacks | `26` | Offset fallback must fail identity-merge readiness. |

Measured entry-gate result: Cycle 18 must explain and recover the identity
evidence behind the `10/36` mapped tracks and `26` offset fallbacks before any
runtime Redis cache or new sharding benchmark is valid. Throughput values alone
cannot unblock runtime implementation.

### Derived comparisons versus accepted baseline

The following deltas are calculated from the carried values above as
`(candidate - baseline) / baseline * 100`, rounded to two decimal places.
Negative wall-time and RTT deltas are improvements; positive FPS and GPU
utilization deltas are improvements. StudentTrack count is a correctness
diagnostic, so neither direction is treated as an optimization success.

| Metric | 15.B1 context-32 delta | 15.B1.C1 context-256 delta | 15.B1.C2 majority-vote delta |
|---|---:|---:|---:|
| DB-completed FPS | `+39.98 %` | `+40.66 %` | `+39.38 %` |
| DB completed elapsed | `-28.56 %` | `-28.90 %` | `-28.25 %` |
| Step 2 frame wall | `-50.15 %` | `-48.15 %` | `-47.82 %` |
| Step 2 through-pose wall | `-49.35 %` | `-47.25 %` | `-46.82 %` |
| GPU average utilization | `+47.69 %` | `+48.88 %` | `+51.03 %` |
| GPU peak utilization | `+56.14 %` | `+50.88 %` | `+52.63 %` |
| Behavior RTT mean | `+7.41 %` | `+7.41 %` | `+8.19 %` |
| StudentTracks | `-1.89 %` | `-1.89 %` | `+20.75 %` |

### Candidate correctness comparison

| Candidate | DB FPS | FPS delta | Minimum F1@IoU0.5 | Maximum F1@IoU0.5 | StudentTracks | Track delta | Identity result |
|---|---:|---:|---:|---:|---:|---:|---|
| 15.B1 context-32 | `7.866851` | `+39.98 %` | `53.455 %` | `61.387 %` | `52` | `-1.89 %` | `FAILED_IDENTITY_CORRECTNESS` |
| 15.B1.C1 context-256 | `7.905` | `+40.66 %` | `53.730 %` | `61.767 %` | `52` | `-1.89 %` | `FAILED_IDENTITY_CORRECTNESS` |
| 15.B1.C2 majority vote | `7.833` | `+39.38 %` | `53.730 %` | `61.109 %` | `64` | `+20.75 %` | `FAILED_IDENTITY_CORRECTNESS_AND_MAPPING_COVERAGE` |

| C2 boundary ratio | Fraction | Measured rate |
|---|---:|---:|
| C2 mapping coverage | `10/36` | `27.78 %` |
| C2 offset fallback rate | `26/36` | `72.22 %` |

The best measured carried throughput was 15.B1.C1 at `7.905` DB-completed
FPS, `+40.66 %` over the accepted pre-shard baseline. It still failed the
identity gate: its four baseline-agreement F1 values ranged only from
`53.730 %` to `61.767 %`. Cycle 18 therefore has a measured performance
opportunity and a measured correctness blocker, not a runtime optimization
decision.

### Cycle 18 closure assessment

The following closure assessment is the Agent 19 pre-override state carried
forward for historical contract evidence. The Agent 20 override decision is
recorded separately below.

| Closure gate | Evidence-backed state | Evidence or consequence |
|---|---|---|
| Real production values and comparisons | `PASS` | `tools/prod/prod_check_cycle18_measured_benchmark_values.py` verifies exact carried values and derived comparisons. |
| Cycle 18 runtime candidate | `MISSING` | No Cycle 18 runtime candidate was deployed or benchmarked. |
| Recovered full boundary packet | `MISSING` | `packet_contract_valid` and `packet_track_count_match` remain unmet in `tools/prod/prod_audit_cycle18_recovered_evidence.py`. |
| Review result | `NO_RUNTIME_CANDIDATE_SELECTED` | The recovered production identity-candidate probe below found no implementation-eligible identity policy. |
| Production benchmark lock | `NOT_HELD` | Agent 19 has no production environment or benchmark authority. |
| Cycle 18 production benchmark | `MISSING` | The carried Cycle 15 measurements cannot create a Cycle 18 runtime decision. |
| Constitution §12.6 decision table | `MISSING` | A Cycle 18 candidate, target gate, before/after evidence, and decision do not exist. |
| Cycle 18 closure state | `CLOSURE_BLOCKED_NO_CYCLE18_RUNTIME_BENCHMARK` | Constitution §12.5/§12.6 forbids `CLOSED` or `COMPLETE` without the missing production evidence. |

### Recovered production identity-candidate probe

The read-only production probe
`cycle18-identity-candidate-probe-20260604T152503Z` used accepted baseline job
`74561b05-105f-4ca8-aeaf-f510f4f802de` and completed majority-vote parent job
`78388c2c-d7f5-42b7-afa4-321216d23b11`. The reproducible helper is
`tools/prod/prod_probe_cycle18_identity_candidates.py`; the tracked evidence
summary and production JSON SHA-256 are in
`docs/architecture/cycle18_identity_candidate_probe.production_summary.json`.
The production evidence paths are:

- `/home/bamby/grad_project/backend/logs/cycle18-identity-candidate-probe-20260604T152503Z/identity_candidate_probe.json`
- `/home/bamby/grad_project/backend/logs/cycle18-identity-candidate-probe-20260604T152503Z/identity_candidate_probe.md`
- `/home/bamby/grad_project/backend/logs/cycle18-identity-candidate-probe-20260604T152503Z/probe.log`

This evidence is `PROBE_ONLY`; it performs no Redis writes, DB mutation,
runtime flag change, worker restart, or production benchmark.

| Read-only identity policy | Minimum all-model F1 | Minimum shard-1 F1 | Selection consequence |
|---|---:|---:|---|
| Existing majority vote | `53.730 %` | `2.917 %` | Already failed production correctness. |
| Boundary majority plus sequential new IDs | `54.409 %` | `3.943 %` | Does not repair identity. |
| Boundary greedy one-to-one plus sequential new IDs | `62.021 %` | `19.978 %` | Best implementable policy, still far from near-parity. |
| Baseline-label whole-track oracle upper bound | `73.086 %` | `49.445 %` | Not implementable because it uses accepted baseline labels; still fails identity agreement. |

The child shards also contain no durable identity evidence from which a Redis
boundary cache could make a governed decision:

| Child job | Tracks | Embeddings | Aliases | Lifecycle events | Canonical IDs | ReID scores | ReID decisions |
|---|---:|---:|---:|---:|---:|---:|---:|
| `3dfe14db-9816-4a65-b6e2-fe74ab5a2f4e` | `38` | `0` | `0` | `0` | `0` | `0` | `0` |
| `fdbfd73d-e42b-4942-91ed-d114a47c21fa` | `36` | `0` | `0` | `0` | `0` | `0` | `0` |

Final Phase A selection result: `NO_RUNTIME_CANDIDATE_SELECTED`. Redis cannot
recover identity evidence that the child shards never produced. Starting a
runtime implementation or benchmark now would repeat a measured correctness
failure without an evidence-backed mechanism capable of fixing it.

## Agent 19 Phase A Evidence Audit

The current boundary packet and merge path are adequate for geometry
diagnostics, but not for a new identity-state proof.

### Current packet and merge behavior

| Current behavior | Resolvable evidence | Contract consequence |
|---|---|---|
| Boundary rows contain only local `tracking_id`, `xyxy`, and detection confidence. | `backend/apps/video_analysis/services/offline_sharding.py:244` | A future packet must add identity and lifecycle evidence; more IoU rows alone repeat the failed candidates. |
| The packet carries decode/authoritative ranges and context size, but no schema version, packet digest, source job ID, shard index, or completion state inside the summary. | `backend/apps/video_analysis/services/offline_sharding.py:283` | A future consumer must reject stale, partial, cross-job, or tampered packets before matching. |
| Incremental tracker state includes `bbox`, `center`, `last_frame`, `last_iou`, optional tracker embedding, and `next_track_id`; none of it is serialized into the boundary summary. | `backend/apps/video_analysis/tasks.py:1425` | A tracker-state design proof must explicitly decide which bounded hints are portable and which remain algorithm-local. |
| The parent merge chooses one parent ID for each whole child track using best-IoU or majority vote. | `backend/apps/video_analysis/services/offline_sharding.py:328`, `backend/apps/video_analysis/services/offline_sharding.py:366`, `backend/apps/video_analysis/services/offline_sharding.py:436` | The next proof must support ambiguity and sub-track/segment evidence; whole-track relabeling is insufficient. |
| An unresolved child track receives an offset ID and is still copied into parent rows. | `backend/apps/video_analysis/services/offline_sharding.py:512` | Offset fallback may remain diagnostic, but it cannot count as a correct identity merge or pass an acceptance gate. |
| First-shard local IDs pass through as parent IDs without an identity proof. | `backend/apps/video_analysis/services/offline_sharding.py:440` | The first shard must be treated as a source namespace, not silently assumed canonical. |
| Existing `StudentTrack`, `TrackAlias`, and `ReIdDecision` models expose lifecycle, confidence, alias, score, threshold, and provenance fields. | `backend/apps/video_analysis/models.py:213`, `backend/apps/video_analysis/models.py:263`, `backend/apps/video_analysis/models.py:283` | The proof should reuse these identity concepts instead of inventing an unrelated Redis-only authority. |
| The current merge creates parent tracks with minimal merge metadata and does not copy the child identity fields above. | `backend/apps/video_analysis/services/offline_sharding.py:525` | Identity decisions and provenance must be explicit before any future parent-row rewrite. |
| The oracle relabel probe records ambiguous geometry and duplicate majority targets, and still cannot restore full agreement. | `tools/prod/prod_analyze_cycle15b1_stitching.py:231` | Ambiguity and collision evidence are required outputs, not optional debug data. |

### Missing identity-state fields

| Missing field group | Required content | Why it is required |
|---|---|---|
| Packet provenance | `schema_version`, `packet_id`, `payload_sha256`, `parent_job_id`, `source_job_id`, `shard_index`, `generated_at_utc` | Prevent cross-job, stale-schema, and mismatched duplicate consumption. |
| Boundary completion | `completion_state`, authoritative/decode ranges, first/last included frame, observation count | Prove the packet is complete for its declared boundary. |
| Tracker-state hints | Algorithm/config digest, last box, center, last frame, last IoU, next-ID watermark, bounded sampled observations | Allow a future tracker-state proof without treating algorithm-local hints as identity truth. |
| Appearance evidence | Immutable embedding reference or bounded feature digest, model/version, dimension, age, and quality state | Geometry alone failed; any appearance use must be traceable and versioned. |
| Identity lifecycle | Local/canonical IDs if known, lifecycle state, identity confidence, ReID score/threshold/decision, continuity state | Preserve existing identity semantics and expose unresolved state. |
| Candidate decisions | Candidate parent IDs, geometry score/votes, appearance score, lifecycle gate, decision, and rejection reason | Make every proposed merge explainable and fail-closed. |
| Collision diagnostics | One-to-many and many-to-one candidates, duplicate target count, ambiguous segments, offset fallback count | The failed majority-vote run demonstrated that aggregate mapped counts are insufficient. |
| Bounds | `ttl_seconds`, `max_bytes`, track cap, observations-per-track cap, and truncation reason | Redis coordination state must remain disposable and bounded. |

## Proposed Boundary Packet V0

This is a design target only. It is not an implemented Redis schema and does
not authorize a runtime cache.

| Level | Proposed fields | Rule |
|---|---|---|
| Packet identity | `schema_version`, `packet_id`, `payload_sha256`, `parent_job_id`, `source_job_id`, `shard_index` | All fields required; consumer rejects any mismatch. |
| Coverage | `boundary_side`, `decode_range`, `authoritative_range`, `first_frame`, `last_frame`, `generated_at_utc`, `completion_state` | `completion_state` must be complete before matching. |
| Runtime fingerprint | `tracker_algorithm`, `tracker_config_digest`, `identity_feature_contract` | Different tracker/feature contracts cannot be compared silently. |
| Track record | `local_track_id`, optional `canonical_track_id`, lifecycle/confidence fields, first/last observation, observation count | Local ID is namespaced by source job and shard. |
| Tracker hints | Last box/center/frame/IoU plus bounded observation samples | Non-authoritative; used only by an explicit tracker-state proof. |
| Appearance reference | Artifact or PostgreSQL embedding reference plus model/version/dimension/digest/age | Raw vector inclusion requires a separately justified byte bound. |
| Candidate diagnostics | Candidate IDs and per-gate scores/decisions/reasons | Ambiguous or conflicting candidates remain unresolved. |
| Packet diagnostics | Truncation, missing fields, collisions, unresolved count, fallback count | No missing diagnostic is converted to zero or success. |
| Redis bounds | `ttl_seconds`, `max_bytes`, track cap, observation cap | Oversized packets fail the candidate proof; they are not silently truncated without a reason. |

### Phase A read-only contract artifacts

The V0 design now has machine-readable artifacts. They remain isolated from
Django, Redis, PostgreSQL, parent merge, and sharded orchestration.

| Artifact | Contract role |
|---|---|
| `docs/architecture/cycle18_boundary_packet_v0.schema.json` | Declares required packet, authority, coverage, fingerprint, bounds, track, candidate, and diagnostic fields. |
| `docs/architecture/cycle18_boundary_packet_v0.example.json` | Synthetic packet that is structurally valid while keeping one identity explicitly unresolved. It is not runtime or benchmark evidence. |
| `docs/architecture/cycle18_boundary_packet_v0.historical_cycle15b1c2.json` | Projects the documented C2 parent/shard IDs, ranges, and `26` offset fallbacks while explicitly marking unavailable per-track evidence. |
| `docs/architecture/cycle18_boundary_packet_v0.historical_cycle15b1c2.source.json` | Records only retained resolvable aggregates and missing raw-evidence categories; it contains no invented per-track data. |
| `tools/prod/prod_project_cycle18_boundary_packet.py` | Reproduces the historical projection from the strict source manifest, rejects inconsistent track aggregates, imports neither Django nor Redis, and can check tracked projection drift without writing output. |
| `tools/prod/prod_audit_cycle18_recovered_evidence.py` | Confirms recovered stitching JSON matches parent/shard scope and retained track aggregates; an optional local V0 packet must also match scope, ranges, and retained source-track count and pass the V0 validator before `full_projection_ready=true`. It emits exact recovery requirements and unmet checks, while `--require-full-projection` gives automation a fail-closed exit status. Identity-merge readiness remains separate. |
| `tools/prod/prod_check_cycle18_boundary_contract.py` | Runs the local contract-artifact inventory, tracked-projection drift, source-reference, expected-failure, recovered-evidence fail-closed, structural drift, and optional external JSON Schema checks in one command. |
| `tools/prod/prod_check_cycle18_schema_validator_drift.py` | Verifies `12` object shapes plus `92` missing/unknown-field mutations while listing intentional manual semantic gates separately. |
| `tools/prod/prod_check_cycle18_doc_references.py` | Verifies Agent 19-owned doc file references and optional line suffixes plus only the bounded Agent 19 `AGENTS.md` paragraph. |
| `tools/prod/prod_check_cycle18_measured_benchmark_values.py` | Verifies exact upstream production replay/job IDs, performance metrics, F1 values, and retained C2 diagnostics carried into this document. |
| `tools/prod/prod_probe_cycle18_identity_candidates.py` | Compares deterministic boundary identity policies and the baseline-label oracle upper bound without writes. |
| `docs/architecture/cycle18_identity_candidate_probe.production_summary.json` | Preserves the production probe evidence hash, exact minimum F1 values, zero child identity-evidence counts, and no-selection result. |
| `tools/prod/prod_probe_cycle18_label_invariant_tracking.py` | Compares independent local-label namespaces through a deterministic maximum-weight one-to-one assignment. |
| `docs/architecture/cycle18_label_invariant_tracking_probe.production_summary.json` | Preserves the production label-invariant metrics and hashed evidence location. |
| `tools/prod/prod_validate_cycle18_boundary_packet.py` | Reads local JSON only; validates canonical digest, packet size, completion, bounds, observations, identity gates, diagnostic consistency, and duplicate conflicts. |
| `backend/tests/unit/pipeline/test_prod_validate_cycle18_boundary_packet.py` | Defines focused coverage for unresolved-valid separation, digest/truncation rejection, offset rejection, governed merge-ready state, and conflicting duplicates. |
| `backend/tests/unit/pipeline/test_prod_project_cycle18_boundary_packet.py` | Covers deterministic projection, aggregate consistency, drift detection, and local-only import isolation. |
| `backend/tests/unit/pipeline/test_prod_audit_cycle18_recovered_evidence.py` | Covers recovered-export scope, aggregate matching, fail-closed readiness, source references, and local-only import isolation. |
| `backend/tests/unit/pipeline/test_prod_check_cycle18_boundary_contract.py` | Covers the one-command local contract gate. |
| `backend/tests/unit/pipeline/test_prod_check_cycle18_schema_validator_drift.py` | Covers the structural schema/manual-validator drift checker. |
| `backend/tests/unit/pipeline/test_prod_check_cycle18_doc_references.py` | Covers four bounded Agent 19 reference sources and non-overlap with Agent 20. |
| `backend/tests/unit/pipeline/test_prod_check_cycle18_measured_benchmark_values.py` | Covers source parity, documented-value drift rejection, and local-only isolation. |
| `backend/tests/unit/pipeline/test_prod_probe_cycle18_identity_candidates.py` | Covers deterministic policy maps and no-selection behavior when identity evidence is absent. |
| `backend/tests/unit/pipeline/test_prod_probe_cycle18_label_invariant_tracking.py` | Covers namespace recovery, real fragmentation, and no-selection behavior. |
| `.gitignore` exceptions | Keep all six JSON artifacts visible to the focused CI test under constitution §18. |

Validator output deliberately separates two states:

| State | Meaning |
|---|---|
| `valid=true` | The packet satisfies the bounded V0 contract. |
| `identity_merge_ready=true` | In addition to being valid, every track has one accepted candidate, valid continuity, accepted ReID identity, and valid appearance evidence. |

The synthetic example resolves to `valid=true` and
`identity_merge_ready=false`. This proves that a well-formed packet cannot be
mistaken for authorization to rewrite parent identity.

The historical Cycle 15.B1.C2 projection resolves to `valid=false` and
`identity_merge_ready=false`. Resolvable retained evidence provides the parent
job, shard-1 job, context-256 two-shard ranges, shard-1 track count,
mapped-track count, and `26` offset fallbacks, but the local workspace contains
neither the cited production stitching JSON nor the historical job rows. The
projection therefore uses `null` for unavailable diagnostic counters, carries
no invented track records, and fails closed on non-empty `missing_fields` plus
offset fallback.

The recovered-evidence intake audit also fails closed independently of the
packet validator. A future packet is `full_projection_ready` only when the
recovered stitching export matches retained scope and aggregates, the packet
matches parent/source/shard scope and the retained decode/authoritative ranges,
its track count equals the retained source-track count, and the V0 validator
reports `valid=true`. `identity_merge_ready` remains a separate result, so a
complete unresolved packet cannot authorize a parent identity rewrite.
The CLI option `--require-full-projection` exits nonzero for the retained
aggregate-only historical path while leaving ordinary scope/aggregate audit
mode available for evidence intake.
For the tracked historical packet, the machine-readable unmet projection set is
exactly `packet_contract_valid` and `packet_track_count_match`; matching scope,
ranges, and retained aggregates are already proven locally.

### Identity decision rules

| Rule | Required behavior |
|---|---|
| Source namespace | Identity is keyed by `(source_job_id, shard_index, local_track_id)` until a governed alias decision exists. |
| First shard | Shard-0 IDs are not automatically canonical outside their source namespace. |
| Ambiguous match | Persist an unresolved decision with candidates and rejection reasons; do not force a merge. |
| Duplicate target | A many-to-one target requires an explicit collision verdict before row rewrite. |
| Whole-track disagreement | Split into bounded sub-track evidence or remain unresolved; do not relabel an internally inconsistent whole track. |
| Appearance conflict | Geometry cannot override a failed appearance/lifecycle gate silently. |
| Redis loss | Redis unavailability cannot alter PostgreSQL terminal status or accepted baseline evidence. |

## Safe Phase A Contract Work

Cycle 18 may produce only a design proof with these fields:

| Boundary-state field | Purpose |
|---|---|
| `schema_version` | Reject stale Redis payloads. |
| `source_job_id` / `shard_index` | Trace every snapshot to one child shard. |
| `first_frame` / `last_frame` | Define finite boundary coverage. |
| `track_observations` | Store first/last per-track boxes and confidence summaries. |
| `identity_features` | Store compact, non-authoritative identity features or embedding references. |
| `completion_marker` | Prove each shard wrote its boundary packet. |
| `diagnostics` | Preserve ambiguous mappings and fallback reasons. |
| `ttl_seconds` / `max_bytes` | Enforce bounded Redis memory. |

Required failure behavior:

| Failure | Required behavior |
|---|---|
| Redis unavailable | Fall back to current non-accepted diagnostic path; do not accept runtime. |
| Missing or expired key | Mark boundary-state unavailable; no silent merge. |
| Partial shard packet | Fail the candidate benchmark or fall back with explicit evidence. |
| Stale schema or runtime fingerprint | Reject the packet before candidate matching. |
| Payload digest mismatch | Reject the packet and record integrity failure. |
| Oversized or truncated packet | Reject unless the declared truncation policy proves the required identity evidence is intact. |
| Ambiguous identity | Do not merge; record ambiguity and keep source provenance. |
| Duplicate completion | Treat as idempotent only if payload hashes match. |
| Duplicate completion with different hash | Fail closed and preserve both packet references for investigation. |
| One-to-many or many-to-one collision | Do not rewrite parent identity until an explicit collision decision exists. |
| Missing appearance or lifecycle evidence | Mark the corresponding gate unavailable; do not infer a pass from geometry. |

## Runtime Implementation Entry Gate

Cycle 18 remains blocked until a review accepts a new identity-state design
proof containing all of the following:

| Entry-gate item | Required proof |
|---|---|
| Read-only packet artifact | A bounded JSON example generated from historical jobs, with no Redis or parent-row mutation. |
| Validator contract | Schema, digest, bounds, completion, fingerprint, and duplicate rules are testable. |
| Identity experiment | Historical boundary evidence shows how ambiguity, collisions, and sub-track disagreement are represented. |
| Authority model | PostgreSQL identity/alias/ReID records remain durable; Redis is disposable coordination only. |
| Failure proof | Redis missing, expired, stale, oversized, conflicting, and duplicate packets all fail closed. |
| Live profile proof | The proposed flag is explicitly disabled in every live profile. |
| Review result | The design proof is explicitly selected for implementation; code existence alone cannot unblock runtime. |

Current entry-gate state:

| Entry-gate item | State | Evidence |
|---|---|---|
| Read-only packet artifact | `AVAILABLE_FAIL_CLOSED` | Synthetic example plus retained-evidence C2 projection exist; the historical projection intentionally fails validation because raw per-track evidence is unavailable locally. |
| Validator contract | `AVAILABLE_FOR_REVIEW` | Standalone validator and focused tests exist; Agent 20 later connected them to the default-off packet producer evidence run. |
| Identity experiment | `ROOT_CAUSE_CONFIRMED_RUNTIME_CONSUMER_MISSING` | The label-invariant production probe confirms raw-label comparison error and real residual association loss; Agent 20 later production-validated packets, but they remain not identity-merge-ready. |
| Authority model | `DECLARED` | Packet requires PostgreSQL authority and disposable coordination-only Redis role. |
| Failure proof | `PARTIAL` | Focused tests cover digest, truncation, offset fallback, diagnostics, and duplicate conflict; runtime Redis failures are intentionally unimplemented. |
| Live profile proof | `DECLARED_ONLY` | Contract is offline-only; no future flag implementation exists to validate. |
| Review result | `NO_RUNTIME_CANDIDATE_SELECTED` | The production read-only candidate probe found no implementable policy capable of restoring identity agreement; runtime implementation remains blocked. |

## Deep Root-Cause Continuation

The user-authorized deep analysis is recorded in
`docs/cycle_18_identity_association_root_cause_investigation.md`. Primary-source
research led to an applied read-only label-invariant evaluator and constitution
§4.6. The evaluator treats local tracker IDs as source-scoped opaque labels,
matches observations by frame/model/IoU, and then uses a deterministic
maximum-weight one-to-one assignment between the baseline and candidate label
namespaces.

Production probe authority:

| Field | Value |
|---|---|
| Baseline job | `74561b05-105f-4ca8-aeaf-f510f4f802de` |
| Candidate job | `78388c2c-d7f5-42b7-afa4-321216d23b11` |
| Evidence directory | `/home/bamby/grad_project/backend/logs/cycle18-label-invariant-probe-20260604T163806Z` |
| JSON SHA-256 | `689a32083b17351ccaebf7d65372c2cab9f0e44380fb933b33e217277cd7bc7f` |
| Decision authority | `PROBE_ONLY` |
| Runtime candidate selected | `false` |

Measured split:

| Scope | Minimum geometry F1 | Minimum raw-label F1 | Minimum global-assignment F1 | Largest residual association gap |
|---|---:|---:|---:|---:|
| All frames | `99.802 %` | `53.730 %` | `69.830 %` | `29.972 pp` |
| Shard 1 authoritative | `100.000 %` | `2.917 %` | `79.876 %` | `20.124 pp` |

This proves two root causes simultaneously. Raw local-label equality is an
invalid independent-run identity gate, but removing that namespace error still
leaves real fragmentation and merge ambiguity. The evaluation rule is fixed;
the runtime consumer remains missing. No Redis cache, parent-row mutation,
sharding acceptance, or merge-ready runtime candidate was introduced.

## Dependency Contract

Cycle 18 depends on:

| Dependency | Contract |
|---|---|
| Cycle 15 | A new identity-state design proof must exist before runtime work. |
| Cycle 17 | Reuse Redis key discipline, counters, fallback, and rollback conventions only. |
| Cycle 20 | Do not couple to streaming persistence or embedding overlap. |
| Cycle 21 | Do not assume more workers; any concurrency change requires its own benchmark. |

## Future Owned Files

If a later design proof unblocks implementation, likely files are:

| File | Possible ownership |
|---|---|
| `backend/apps/video_analysis/services/offline_sharding.py` | Parent merge integration or rollback path. |
| Dedicated boundary-cache helper; path not selected | A separate implementation turn must choose and create a resolvable module path. |
| `backend/apps/video_analysis/tasks.py` | Guarded, non-authoritative snapshot writes only. |
| `backend/apps/video_analysis/management/commands/cycle15b1_sharded_ingest.py` | Offline-only orchestration. |
| `backend/config/settings/base.py` | Disabled-by-default flags. |
| `tools/prod/prod_enable_parallel_flow.sh` | Offline/live profile discipline. |
| `tools/prod/prod_collect_benchmark_metrics.py` | Boundary-state metrics. |
| `tools/prod/prod_watch_benchmark_metrics.sh` | Read-only diagnostics. |

## Benchmark Evidence Required

No Cycle 18 implementation may be accepted without:

| Gate | Required evidence |
|---|---|
| Production authority | Completed Linux RTX 5090 `combined.mp4` benchmark. |
| Identity correctness | Ground-truth-backed HOTA/AssA, IDF1, ID switches, fragmentation, and shard diagnostics pass; proxy-only label-invariant metrics are reported separately and cannot alone accept identity. |
| DB parity | Frames, detections, boxes, embeddings, and terminal state preserved. |
| Redis bounds | Command count, wall, bytes, memory, errors, TTL, and fallback count recorded. |
| Sharding safety | Parent merge wall, child critical path, and rollback proof recorded. |
| Live safety | Offline-only flag disabled in live profile and documented. |

## Rollback Strategy

Future rollback must be one env/profile change plus worker restart:

```text
OFFLINE_VIDEO_SHARDING_ENABLED=0
OFFLINE_VIDEO_SHARD_BOUNDARY_REDIS_ENABLED=0
```

Redis keys must be TTL-bound and disposable. Candidate jobs must reach terminal
state without mutating accepted baseline evidence.

## Agent 19 Governed Decision Before Override

| Decision field | Final state |
|---|---|
| Valid pre-benchmark decision statement | `NO_DECISION_PRODUCTION_BENCHMARK_REQUIRED` |
| Phase A candidate selection | `NO_RUNTIME_CANDIDATE_SELECTED` |
| Root-cause result | `RAW_LABEL_CONTRACT_DEFECT_CONFIRMED` plus `RESIDUAL_ASSOCIATION_GAP_CONFIRMED` |
| Runtime implementation | `BLOCKED` |
| Production benchmark lock | `NOT_HELD` |
| Cycle 18 production benchmark | `MISSING` |
| Constitution §12.6 decision table | `MISSING` |
| Closure eligibility | `false` |
| Closure state | `CLOSURE_BLOCKED_NO_CYCLE18_RUNTIME_BENCHMARK` |

The final Phase A review does not select a Redis boundary-state runtime
candidate. The first production read-only probe proved that the best
implementable boundary policy reaches only `62.021 %` minimum all-model raw
track-label F1 and `19.978 %` minimum shard-1 raw track-label F1. The deeper
label-invariant production probe then proved that raw label equality materially
understates independent-run agreement: shard-1 minimum F1 rose from `2.917 %`
to `79.876 %` after one-to-one global assignment. It also proved a real
remaining association gap of up to `20.124 pp` in shard 1. Both child shards
still have zero embeddings, aliases, lifecycle events, canonical IDs, ReID
scores, and ReID decisions.

Therefore, at the Agent 19 pre-override point, Cycle 18 could not be honestly
marked `CLOSED`, `COMPLETE`, `ACCEPTED`, `NOT ACCEPTED`, `REJECTED`, or
`SKIPPED`. Agent 20 later created two bounded production-evidence decisions:
the one-to-one track-map candidate is `NOT_ACCEPTED`, and the default-off
boundary packet producer is
`ACCEPTED_EVIDENCE_ONLY_NOT_IDENTITY_MERGE_READY`. The next valid action is a
separate appearance-backed identity association consumer; Redis remains
disposable coordination only and sharding remains disabled.

## Agent 20 Override Production Decision

The user explicitly overrode Cycle 18 after the Agent 19 handoff. Agent 20's
override ledger is `docs/agent_20_cycle_18_override_turn.md`.

The candidate is `OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=one_to_one` in
`backend/apps/video_analysis/services/offline_sharding.py`. It uses existing
child/parent boundary vote evidence and performs a deterministic maximum-vote
one-to-one assignment so two child tracks cannot both claim the same parent
track before offset fallback is applied. Focused coverage lives in
`backend/tests/unit/video_analysis/test_cycle15b1_shard_merge.py`.

This candidate is a bounded parent merge policy only. It is not a Redis
identity-state cache, not a new identity-feature producer, and not live-profile
behavior.

### Agent 20 benchmark release

```text
BENCHMARK_LOCK
agent: Agent 20
cycle: Cycle 18 one-to-one boundary track-map candidate
replay_key: cycle18-one-to-one-trackmap-20260604T174231Z
baseline_metrics: /home/bamby/grad_project/backend/logs/cycle15b-pre-shard-baseline-20260603T193531Z/metrics.json
candidate_env_delta: OFFLINE_VIDEO_SHARDING_ENABLED=1, OFFLINE_VIDEO_SHARD_COUNT=2, OFFLINE_VIDEO_SHARD_CONTEXT_FRAMES=256, OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=one_to_one, TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=1, EMBEDDING_PREFETCH_TRACK_LOOKUP=1, EMBEDDING_REDIS_SIDE_EFFECT_COALESCING=1
started_at_utc: 2026-06-04T17:42:31Z
expected_cleanup: wrapper trap restores OFFLINE_VIDEO_SHARDING_ENABLED=0, OFFLINE_VIDEO_SHARD_COUNT=1, OFFLINE_VIDEO_SHARD_CONTEXT_FRAMES=32, OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=best_iou and restarts workers
```

```text
BENCHMARK_RELEASE
agent: Agent 20
cycle: Cycle 18 one-to-one boundary track-map candidate
replay_key: cycle18-one-to-one-trackmap-20260604T174231Z
job_id: aa5f5328-ae81-49da-a44b-ae308859035b
status: completed; 4541/4541 frames
metrics_json: /home/bamby/grad_project/backend/logs/cycle18-one-to-one-trackmap-20260604T174231Z/metrics.json
metrics_md: /home/bamby/grad_project/backend/logs/cycle18-one-to-one-trackmap-20260604T174231Z/metrics.md
model_agreement_json: /home/bamby/grad_project/backend/logs/cycle18-one-to-one-trackmap-20260604T174231Z/model_agreement_baseline_vs_one_to_one.json
model_agreement_md: /home/bamby/grad_project/backend/logs/cycle18-one-to-one-trackmap-20260604T174231Z/model_agreement_baseline_vs_one_to_one.md
label_invariant_json: /home/bamby/grad_project/backend/logs/cycle18-one-to-one-trackmap-20260604T174231Z/label_invariant_tracking.json
label_invariant_md: /home/bamby/grad_project/backend/logs/cycle18-one-to-one-trackmap-20260604T174231Z/label_invariant_tracking.md
summary_json: docs/architecture/cycle18_one_to_one_trackmap.production_summary.json
rollback_verified: yes; OFFLINE_VIDEO_SHARDING_ENABLED=0, OFFLINE_VIDEO_SHARD_COUNT=1, OFFLINE_VIDEO_SHARD_CONTEXT_FRAMES=32, OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=best_iou
released_at_utc: 2026-06-04T17:59:08Z
```

The tracked evidence summary is
`docs/architecture/cycle18_one_to_one_trackmap.production_summary.json`.

### Production result

| Metric | Accepted pre-shard baseline | One-to-one candidate | Delta | Decision impact |
|---|---:|---:|---:|---|
| DB-completed FPS | `5.619787` | `7.913961` | `+40.82 %` | Throughput improved. |
| DB completed elapsed | `808.038 s` | `573.796 s` | `-28.99 %` | Total wall improved. |
| Step 2 frame wall | `467.449833 s` | `241.830808 s` | `-48.27 %` | Inference wall improved. |
| Step 2 through pose upload | `641.154064 s` | `338.453209 s` | `-47.21 %` | End-to-end Step 2 segment improved. |
| Behavior RTT mean | `83.530 ms` | `91.679 ms` | `+9.76 %` | Latency regressed. |
| Behavior RTT p95 | `129.514 ms` | `149.768 ms` | `+15.64 %` | Tail latency regressed. |
| GPU avg util | `11.846 %` | `17.825 %` | `+50.47 %` | Utilization improved. |
| GPU peak util | `57.000 %` | `88.000 %` | `+54.39 %` | Peak utilization improved. |
| Detection rows | `72744` | `72816` | `+0.10 %` | Row parity within noise. |
| BBox rows | `72744` | `72816` | `+0.10 %` | Row parity within noise. |
| Embedding rows | `72578` | `72650` | `+0.10 %` | Row parity within noise. |
| StudentTracks | `53` | `53` | `0.00 %` | Count stayed stable but association quality failed. |
| Minimum raw model F1@IoU0.5 | `100.000 %` proxy target | `53.788 %` | Failed | Signal gate failed. |
| Minimum label-invariant all-model F1 | `100.000 %` proxy target | `71.744 %` | Failed | Association gate failed. |
| Minimum label-invariant shard-1 F1 | `100.000 %` proxy target | `79.876 %` | Failed | Boundary association still failed. |
| Rollback proof | Required | Verified | Pass | Sharding disabled and track-map mode restored to `best_iou`. |

Track-map diagnostics:

| Shard | Mode | Tracks | Mapped to existing parent | Offset fallbacks | Result |
|---|---|---:|---:|---:|---|
| Shard 0 | `identity` | `38` | `38` | `0` | First shard remained pass-through. |
| Shard 1 | `one_to_one` | `36` | `21` | `15` | Coverage improved over majority vote but still left `41.67 %` fallback. |

Raw model-agreement F1@IoU0.5 versus the accepted pre-shard baseline:

| Model | F1 |
|---|---:|
| `attention_tracking` | `59.278 %` |
| `hand_raising` | `61.223 %` |
| `person_detection` | `64.861 %` |
| `sitting_standing` | `53.788 %` |

Label-invariant tracking probe after the candidate:

| Metric | Value |
|---|---:|
| Minimum all-model geometry F1 | `99.802 %` |
| Minimum all-model raw-label F1 | `53.788 %` |
| Minimum all-model global-assignment F1 | `71.744 %` |
| Minimum shard-1 geometry F1 | `100.000 %` |
| Minimum shard-1 raw-label F1 | `3.581 %` |
| Minimum shard-1 global-assignment F1 | `79.876 %` |
| Largest all-model residual gap | `28.256 pp` |
| Largest shard-1 residual gap | `20.124 pp` |

Decision: **Cycle 18 `one_to_one` boundary track-map candidate is NOT
ACCEPTED**. It materially improved throughput and GPU utilization and kept
`StudentTracks` at `53`, but it failed the identity/model-agreement gates.
The residual association gap persisted even after label-invariant matching,
and behavior RTT regressed. Keep sharding disabled and keep production restored
to `OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=best_iou`.

Cycle 18 as a broader identity-state problem remains open. The next valid work
is not another boundary-vote merge policy; it is a reviewed boundary
tracklet-state producer carrying bounded motion, temporal, appearance,
lifecycle, candidate-cost, and ambiguity evidence before any new runtime
candidate or production benchmark.

## Boundary Packet Producer Continuation

Agent 20 staged the first bounded producer for the existing V0 packet contract.
It is guarded by `OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_ENABLED=0` and remains
offline-only. The producer writes no Redis keys, does not change parent merge
decisions, and does not make a canonical identity decision. It emits
tracklet-shaped packet evidence from the child boundary summary so the existing
validator can distinguish contract validity from merge readiness.

Local implementation references:

| Artifact | State |
|---|---|
| Runtime producer | `backend/apps/video_analysis/services/offline_sharding.py` adds `build_cycle18_boundary_packet(...)` and attaches `cycle18_boundary_packet` when explicitly enabled. |
| Metadata write | `backend/apps/video_analysis/tasks.py` stores `offline_video_shard_boundary_packet` on child jobs when the packet is emitted. |
| Settings gate | `backend/config/settings/base.py` defaults `OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_ENABLED=0` and bounds max tracks, observations, bytes, and TTL. |
| Production wrapper | `tools/prod/prod_run_cycle15b1_two_shard_runtime_benchmark.sh` supports `--boundary-packet-enabled` and writes `boundary_packet_validation.json/.md` when enabled. |
| Local test | `backend/tests/unit/video_analysis/test_cycle15b1_shard_merge.py` validates a generated runtime packet with `tools/prod/prod_validate_cycle18_boundary_packet.py`. |

This producer is not a sharding acceptance decision. It is a prerequisite
evidence path for a future identity-state candidate.

```text
BENCHMARK_LOCK
agent: Agent 20
cycle: Cycle 18 boundary packet producer evidence run
replay_key: cycle18-boundary-packet-producer-20260604T181738Z
baseline_metrics: /home/bamby/grad_project/backend/logs/cycle15b-pre-shard-baseline-20260603T193531Z/metrics.json
candidate_env_delta: OFFLINE_VIDEO_SHARDING_ENABLED=1, OFFLINE_VIDEO_SHARD_COUNT=2, OFFLINE_VIDEO_SHARD_CONTEXT_FRAMES=256, OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=best_iou, OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_ENABLED=1, OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_MAX_TRACKS=128, OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_MAX_OBSERVATIONS_PER_TRACK=512, OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_MAX_BYTES=1048576, TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=1, EMBEDDING_PREFETCH_TRACK_LOOKUP=1, EMBEDDING_REDIS_SIDE_EFFECT_COALESCING=1
started_at_utc: 2026-06-04T18:17:38Z
expected_cleanup: wrapper trap restores OFFLINE_VIDEO_SHARDING_ENABLED=0, OFFLINE_VIDEO_SHARD_COUNT=1, OFFLINE_VIDEO_SHARD_CONTEXT_FRAMES=32, OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=best_iou, OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_ENABLED=0 and restarts workers
```

```text
BENCHMARK_RELEASE
agent: Agent 20
cycle: Cycle 18 boundary packet producer evidence run
replay_key: cycle18-boundary-packet-producer-20260604T181738Z
job_id: 2cf0b4b3-5e81-41c4-9eda-9e7faa97c224
status: completed; 4541/4541 frames
metrics_json: /home/bamby/grad_project/backend/logs/cycle18-boundary-packet-producer-20260604T181738Z/metrics.json
metrics_md: /home/bamby/grad_project/backend/logs/cycle18-boundary-packet-producer-20260604T181738Z/metrics.md
boundary_packet_validation_json: /home/bamby/grad_project/backend/logs/cycle18-boundary-packet-producer-20260604T181738Z/boundary_packet_validation.json
boundary_packet_validation_md: /home/bamby/grad_project/backend/logs/cycle18-boundary-packet-producer-20260604T181738Z/boundary_packet_validation.md
model_agreement_json: /home/bamby/grad_project/backend/logs/cycle18-boundary-packet-producer-20260604T181738Z/model_agreement_baseline_vs_boundary_packet.json
model_agreement_md: /home/bamby/grad_project/backend/logs/cycle18-boundary-packet-producer-20260604T181738Z/model_agreement_baseline_vs_boundary_packet.md
label_invariant_json: /home/bamby/grad_project/backend/logs/cycle18-boundary-packet-producer-20260604T181738Z/label_invariant_tracking.json
label_invariant_md: /home/bamby/grad_project/backend/logs/cycle18-boundary-packet-producer-20260604T181738Z/label_invariant_tracking.md
summary_json: docs/architecture/cycle18_boundary_packet_producer.production_summary.json
rollback_verified: yes; OFFLINE_VIDEO_SHARDING_ENABLED=0, OFFLINE_VIDEO_SHARD_COUNT=1, OFFLINE_VIDEO_SHARD_CONTEXT_FRAMES=32, OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=best_iou, OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_ENABLED=0
released_at_utc: 2026-06-04T18:34:32Z
```

Production packet validation:

| Child job | Packet valid | Merge ready | Tracks | Observations | Unresolved tracks | Payload bytes |
|---|---:|---:|---:|---:|---:|---:|
| `db77789f-202b-44b6-b493-f6977f9a75e0` | `true` | `false` | `34` | `2363` | `34` | `354854` |
| `2e9a92ec-5ea1-4891-9688-18dd3e7e9743` | `true` | `false` | `32` | `3177` | `32` | `471240` |

Producer benchmark metrics:

| Metric | Value |
|---|---:|
| DB-completed FPS | `7.902540` |
| DB completed elapsed | `574.625 s` |
| Step 2 frame wall | `245.410728 s` |
| Step 2 through pose upload | `337.523178 s` |
| Parent merge wall | `110.292038 s` |
| GPU average utilization | `17.523 %` |
| GPU peak utilization | `87.000 %` |
| Detection / bbox / embedding rows | `72816 / 72816 / 72650` |
| StudentTracks | `52` |
| Behavior RTT mean / p95 / p99 | `91.822 / 150.355 / 164.780 ms` |

Correctness remained blocked:

| Gate | Result |
|---|---|
| Raw model-agreement minimum F1@IoU0.5 | `53.730 %` |
| Label-invariant all-model global-assignment minimum | `69.163 %` |
| Label-invariant shard-1 global-assignment minimum | `71.884 %` |
| Packet identity merge readiness | `0/2` packets merge-ready |

Decision: **Cycle 18 boundary packet producer is
ACCEPTED_EVIDENCE_ONLY_NOT_IDENTITY_MERGE_READY**. It resolves the missing
contract-valid packet producer evidence path, but it does not make the sharded
runtime correct and does not authorize 15.B2. The next implementation must add
governed appearance evidence plus a parent-side association consumer with
candidate costs, lifecycle gates, ambiguity handling, and globally one-to-one
merge-ready decisions.

## 2026-06-05 Appearance-Packet Candidate Staged Locally

The next blocker-resolution candidate is staged locally only. It is not a
production decision, not a Cycle 18 closure decision, and not a sharding
acceptance claim.

| Item | State | Reference |
|---|---|---|
| Boundary appearance producer | `STAGED_DEFAULT_OFF` | `backend/apps/video_analysis/services/offline_sharding.py` |
| Preview-frame evidence carrier | `STAGED` | `backend/apps/pipeline/multi_model.py`, `backend/apps/video_analysis/tasks.py` |
| Parent-side association consumer | `STAGED_DEFAULT_OFF` as `OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=appearance_packet` | `backend/apps/video_analysis/services/offline_sharding.py` |
| Runtime gates | `OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_ENABLED=0`, `OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_APPEARANCE_ENABLED=0` by default | `backend/config/settings/base.py`, `tools/prod/prod_enable_parallel_flow.sh` |
| Benchmark wrapper support | `STAGED`; cleanup restores appearance flags to disabled defaults and emits metrics, packet validation, model agreement, label-invariant tracking, rollback proof, and figure bundle/manifest | `tools/prod/prod_run_cycle15b1_two_shard_runtime_benchmark.sh` |
| Local focused validation | `PASSED` | `backend/tests/unit/video_analysis/test_cycle15b1_shard_merge.py` |

What changed:

| Blocker | Local resolution |
|---|---|
| Packet `appearance_reference` was always `unavailable`. | Boundary packets can now write bounded per-track crop-descriptor feature files and store a digest-backed reference when `OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_APPEARANCE_ENABLED=1`. |
| No parent-side packet consumer existed. | `appearance_packet` mode verifies feature-file digests, combines boundary geometry votes with appearance score/margin gates, writes candidate decisions back into the packet, and leaves missing or ambiguous matches unresolved. |
| Prior `one_to_one` candidate forced geometry-only merges. | The new mode is separate from `one_to_one`; it requires verified appearance evidence and fails closed before offset fallback. |
| Live-profile risk. | All new flags are offline sharding flags and remain disabled by default in profile reset paths. |

Local validation run on 2026-06-05:

| Validation | Result | Resolvable reference |
|---|---|---|
| Python compile | `PASS` | `backend/apps/pipeline/multi_model.py`, `backend/apps/video_analysis/services/offline_sharding.py`, `backend/apps/video_analysis/tasks.py`, `tools/prod/prod_validate_cycle18_boundary_packet.py`, `tools/prod/prod_collect_benchmark_metrics.py` |
| Shard/Cycle 18 unit suite | `9 passed` | `backend/tests/unit/video_analysis/test_cycle15b1_shard_merge.py` |
| Focused Cycle 18 slice | `12 passed` | `backend/tests/unit/pipeline/test_prod_probe_cycle18_label_invariant_tracking.py`, `backend/tests/unit/video_analysis/test_cycle15b1_shard_merge.py` |
| Measured-value checker | `PASS`; still reports `cycle18_runtime_benchmark_present=false` | `tools/prod/prod_check_cycle18_measured_benchmark_values.py` |
| Benchmark wrapper syntax | `PASS` | `tools/prod/prod_run_cycle15b1_two_shard_runtime_benchmark.sh` |
| Figure generator focused unit test | `4 passed` | `backend/tests/unit/pipeline/test_prod_generate_cycle_figures.py` |

Final local decision for this step:

| Decision field | Value |
|---|---|
| Candidate state | `STAGED_LOCAL_ONLY` |
| Production benchmark lock | `NOT_HELD` |
| Section 12.6 optimization decision | `MISSING` |
| Cycle 18 closure state | `OPEN_PENDING_PRODUCTION_BENCHMARK` |
| Required next evidence | Full `combined.mp4` Linux RTX 5090 benchmark with `OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=appearance_packet`, packet validation, model agreement, label-invariant association metrics, DB/GPU/RTT metrics, rollback proof, and generated figure bundle/manifest embedded in the responsible benchmark/result docs. |

## 2026-06-05 Cycle 18.B Production Benchmark Lock

```text
BENCHMARK_LOCK
agent: current Codex runtime session
cycle: Cycle 18.B appearance-backed boundary association
replay_key: cycle18b-appearance-packet-20260605T151057Z
baseline_metrics: /home/bamby/grad_project/backend/logs/cycle15b-pre-shard-baseline-20260603T193531Z/metrics.json
candidate_env_delta: OFFLINE_VIDEO_SHARDING_ENABLED=1, OFFLINE_VIDEO_SHARD_COUNT=2, OFFLINE_VIDEO_SHARD_CONTEXT_FRAMES=256, OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=appearance_packet, OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_ENABLED=1, OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_APPEARANCE_ENABLED=1, TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=1, EMBEDDING_PREFETCH_TRACK_LOOKUP=1, EMBEDDING_REDIS_SIDE_EFFECT_COALESCING=1
started_at_utc: 2026-06-05T15:10:57Z
expected_cleanup: wrapper restores sharding disabled, shard count 1, context 32, track-map best_iou, boundary packet disabled, boundary appearance disabled, and restarts Celery workers
```

```text
BENCHMARK_RELEASE
agent: current Codex runtime session
cycle: Cycle 18.B appearance-backed boundary association
replay_key: cycle18b-appearance-packet-20260605T151057Z
job_id: 85be8348-dc0b-4319-9974-f1a206203884
status: completed_but_candidate_not_accepted
metrics_json: /home/bamby/grad_project/backend/logs/cycle18b-appearance-packet-20260605T151057Z/metrics.json
metrics_md: /home/bamby/grad_project/backend/logs/cycle18b-appearance-packet-20260605T151057Z/metrics.md
model_agreement_json: /home/bamby/grad_project/backend/logs/cycle18b-appearance-packet-20260605T151057Z/model_agreement.json
model_agreement_md: /home/bamby/grad_project/backend/logs/cycle18b-appearance-packet-20260605T151057Z/model_agreement.md
label_invariant_json: /home/bamby/grad_project/backend/logs/cycle18b-appearance-packet-20260605T151057Z/label_invariant_tracking.json
label_invariant_md: /home/bamby/grad_project/backend/logs/cycle18b-appearance-packet-20260605T151057Z/label_invariant_tracking.md
packet_validation_json: /home/bamby/grad_project/backend/logs/cycle18b-appearance-packet-20260605T151057Z/boundary_packet_validation.json
packet_validation_md: /home/bamby/grad_project/backend/logs/cycle18b-appearance-packet-20260605T151057Z/boundary_packet_validation.md
rollback_verified: true
figure_manifest: docs/figures/benchmark_artifacts/cycle18b-appearance-packet-20260605T151057Z/figure_manifest.json
wrapper_note: driver.exit=1 because the initial wrapper had a post-run figure-manifest syntax bug after benchmark and evidence collection; the wrapper was fixed and figures were generated from completed raw artifacts
released_at_utc: 2026-06-05T15:24:30Z
```

### Cycle 18.B Decision

Decision: **NOT ACCEPTED**. The production run completed and preserved the
sharding performance upside, but the correctness gates failed.

| Metric | Accepted pre-shard baseline | Cycle 18.B candidate | Delta | Decision impact |
|---|---:|---:|---:|---|
| DB-completed FPS | `5.619787` | `7.409522` | `+31.85 %` | Pass throughput direction |
| DB completed elapsed | `808.038 s` | `612.860 s` | `-24.15 %` | Pass total-wall direction |
| Step 2 frame wall | `467.449833 s` | `248.323884 s` | `-46.88 %` | Pass inference-wall direction |
| Step 2 through-pose wall | `641.154064 s` | `376.362543 s` | `-41.30 %` | Pass through-pose direction |
| Behavior RTT mean | `83.530 ms` | `90.765 ms` | `+8.66 %` | Regressed latency |
| GPU average utilization | `11.846 %` | `16.325 %` | `+37.81 %` | Pass utilization direction |
| GPU peak utilization | `57.000 %` | `91.000 %` | `+59.65 %` | Pass utilization direction |
| StudentTracks | `53` | `65` | `+22.64 %` | Fails identity gate |

| Correctness gate | Value | Result |
|---|---:|---|
| Valid boundary packets | `0/2` | Fail |
| Merge-ready boundary packets | `0/2` | Fail |
| Minimum model-agreement F1@IoU0.5 | `53.730 %` | Fail |
| Minimum all-model global-assignment F1 | `69.752 %` | Fail |
| Minimum shard-1 global-assignment F1 | `79.876 %` | Fail |
| Rollback verified | `true` | Pass |

Packet payloads exceeded the configured byte cap and were marked truncated:
`301968` bytes for shard0 and `299423` bytes for shard1. Shard1 also retained
`23` unresolved tracks. The next Cycle 18 task must first make appearance
packets valid under the configured byte budget and then improve association
readiness before another sharding runtime benchmark.

Figure evidence:

![Cycle 18.B Decision Delta](figures/benchmark_artifacts/cycle18b-appearance-packet-20260605T151057Z/cycle18b_appearance_packet__decision_delta.png)

![Cycle 18.B Correctness Gate](figures/benchmark_artifacts/cycle18b-appearance-packet-20260605T151057Z/cycle18b_appearance_packet__correctness_gate.png)

### Cycle 18.C Next Work Package

Cycle 18.C is the next sorted latency cycle. It is a redesign-and-proof cycle,
not a rerun of the failed Cycle 18.B `appearance_packet` profile.

| Field | Value |
|---|---|
| Cycle | `18.C packet-budget and association-readiness redesign` |
| State | `STAGED_LOCAL_ONLY_AFTER_18B_NOT_ACCEPTED` |
| Streaming compatibility | `offline-only` because it depends on offline sharding and cross-shard boundary state |
| Primary blocker | Boundary packets were present but invalid: `0/2` valid and `0/2` merge-ready. |
| Packet-budget blocker | Shard packets serialized at `301968` and `299423` bytes and were marked truncated. |
| Association blocker | Shard1 retained `23` unresolved tracks and the candidate produced `65` StudentTracks versus baseline `53`. |
| Correctness blocker | Minimum model-agreement F1@IoU0.5 was `53.730 %`; minimum all-model global-assignment F1 was `69.752 %`. |
| First implementation target | Make packets valid within the configured byte budget without dropping identity-critical evidence. |
| Second implementation target | Improve shard-1 association readiness before parent merge, including unresolved-track diagnostics and fail-closed ambiguity handling. |
| Production benchmark gate | Full `combined.mp4` Linux RTX 5090 benchmark with packet validation, model agreement, label-invariant tracking, DB/GPU/RTT metrics, rollback proof, and figure bundle. |
| Expected gain if gates pass | Recover the measured sharding envelope: DB FPS `5.620 -> 7.410+`, Step 2 wall `467.450 s -> 248.324 s`, and GPU avg `11.846 % -> 16.325 %+`. |
| Acceptance rule | No acceptance if packet validity, merge readiness, StudentTrack parity, model agreement, or label-invariant identity gates fail, regardless of FPS. |

### 2026-06-05 Cycle 18.C Local Redesign

Cycle 18.C is staged locally only. It is not a production benchmark, not a
§12.6 decision, and not a sharding acceptance claim.

| Field | Value |
|---|---|
| Cycle state | `BENCHMARK_LOCK_HELD / NO_DECISION_PENDING_PRODUCTION_RUN` |
| Benchmark lock | `HELD` for replay `cycle18c-packet-budget-active-edge-20260605T162825Z` |
| Figure Planner | `Huygens` sub-agent, read-only plan; no files edited |
| Figure Implementer | `Archimedes` sub-agent, generator/test lane only |
| Runtime owner | Agent 19 |
| Streaming compatibility | `offline-only`; sharding remains disabled for live profiles |
| Decision authority | `NO_PRODUCTION_DECISION`; full Linux RTX 5090 `combined.mp4` benchmark still required |

Implemented local changes:

| Blocker | Local change | Reference |
|---|---|---|
| Boundary packets exceeded the default byte budget and were marked truncated. | Packet construction now deterministically samples non-critical observations until the serialized packet fits `OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_MAX_BYTES`; track rows, tracker hints, and appearance references are retained. | `backend/apps/video_analysis/services/offline_sharding.py` |
| Shard boundary packets could include video start/end evidence that is not an inter-shard merge edge. | Shard metadata now carries `shard_count`, and packet/summary windows include only actual inter-shard edges for first, middle, and final shards. | `backend/apps/video_analysis/services/offline_sharding.py` |
| Figure evidence lacked Cycle 18.C packet/readiness and label-invariant plots. | The generator now supports packet budget/readiness, identity label-invariant, resource-tail, Redis profile, unavailable-summary, historical/context runs, run labels, and figure role metadata. | `tools/prod/prod_generate_cycle_figures.py` |
| Production wrapper would have labeled Cycle 18.C figure output with the historical Cycle 18.B slug. | The wrapper now accepts figure slug, status, label, planner, and implementer metadata for the generated manifest while preserving old defaults. | `tools/prod/prod_run_cycle15b1_two_shard_runtime_benchmark.sh` |

Local validation:

| Validation | Result |
|---|---|
| Python compile | `PASS` for `backend/apps/video_analysis/services/offline_sharding.py`, `tools/prod/prod_generate_cycle_figures.py`, `backend/tests/unit/video_analysis/test_cycle15b1_shard_merge.py`, and `backend/tests/unit/pipeline/test_prod_generate_cycle_figures.py` |
| Focused runtime + figure + shard-planning tests | `18 passed` across `backend/tests/unit/video_analysis/test_cycle15b1_shard_merge.py`, `backend/tests/unit/pipeline/test_prod_generate_cycle_figures.py`, and `backend/tests/unit/pipeline/test_prod_plan_video_shards.py` |
| Wrapper syntax | `PASS` for `bash -n tools/prod/prod_run_cycle15b1_two_shard_runtime_benchmark.sh` |
| Packet byte-budget regression | `PASS`; generated packet remains contract-valid under a tight byte cap and records observation sampling without setting `bounds.truncated=true` |
| Active-edge regression | `PASS`; a final shard packet/summary excludes the terminal video edge and reports `boundary_side=left` |
| Figure generator regression | `PASS`; generated manifest includes figure roles, run metadata, packet/readiness metrics, identity metrics, Redis/resource metrics, and unavailable summary |

Remaining blockers before any Cycle 18.C decision:

| Blocker | Required proof |
|---|---|
| Production packet validity | New production `boundary_packet_validation.json` must show all packets valid under the configured byte cap. |
| Identity merge readiness | New production packet validation must show merge-ready packets; unresolved active boundary tracks must be explained and fail closed. |
| StudentTrack parity | Candidate `StudentTracks` must stay at or near the accepted baseline (`53`) without hidden identity merges. |
| Model and label-invariant correctness | Model F1@IoU0.5 and label-invariant global-assignment gates must pass, including shard-1 residual association. |
| Figure evidence | Cycle 18.C figure bundle and manifest must be generated from the raw production artifacts and embedded in this doc plus `docs/production_inference_benchmark.md`. |
| Rollback | Production wrapper must prove sharding, packet, and appearance flags return to disabled defaults. |

Cycle 18.C benchmark lock:

```text
BENCHMARK_LOCK
cycle: 18.C packet-budget and association-readiness redesign
agent: 19
state: HELD
claimed_at_utc: 2026-06-05T16:28:25Z
replay_key: cycle18c-packet-budget-active-edge-20260605T162825Z
candidate_code_sha: 8dd37244
candidate_env_delta: OFFLINE_VIDEO_SHARDING_ENABLED=1, OFFLINE_VIDEO_SHARD_COUNT=2, OFFLINE_VIDEO_SHARD_CONTEXT_FRAMES=256, OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=appearance_packet, OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_ENABLED=1, OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_APPEARANCE_ENABLED=1, TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=1, EMBEDDING_PREFETCH_TRACK_LOOKUP=1, EMBEDDING_REDIS_SIDE_EFFECT_COALESCING=1
figure_metadata: cycle_slug=cycle18c_packet_budget_active_edge, status=NO_DECISION_PENDING_REVIEW, planner=Huygens, implementer=Archimedes
required_evidence: metrics_json, metrics_md, sharded_summary_json, gpu_csv, boundary_packet_validation_json_md, model_agreement_json_md, label_invariant_json_md, rollback_json_md, figure_manifest, figure_markdown, generated_pngs
decision_authority: NO_DECISION_UNTIL_SECTION_12_6_EVIDENCE_TABLE
```

Cycle 15.B1 and 15.B2 stay blocked until Cycle 18.C proves the two-shard
identity boundary is valid. Cycle 20 remains staged behind this blocker unless a
new benchmark shows post-stage persistence/embedding has become the dominant
latency source.
