# Cycle 18.D Combined-Cost Boundary Association Investigation

**Last updated:** 2026-06-05

**Status:** `PRODUCTION_BENCHMARK_COMPLETE` /
`BENCHMARK_LOCK_RELEASED` / `NOT_ACCEPTED`. This cycle implemented a new
default-off boundary association consumer that addressed part of the confirmed
Cycle 18.C root cause, but the completed native-Linux RTX 5090 `combined.mp4`
benchmark did not pass the packet-validity, merge-readiness, model-agreement,
or identity gates required by constitution §12.5 / §12.6. The accepted
production profile is unchanged (`OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=best_iou`,
sharding disabled).

**Streaming compatibility:** `offline-only`. The association depends on finite
shard boundaries and whole-file parent coordination. It MUST stay disabled for
RTSP, RTSPS, WHEP/WebRTC, and HLS live profiles (constitution §8.6).

## Why This Cycle Exists

Cycle 18.C (`appearance_packet`) was **NOT ACCEPTED**. The production benchmark
`cycle18c-packet-budget-active-edge-20260605T162825Z` fixed packet byte
validity but failed every correctness gate: only `1/2` packets merge-ready,
shard-1 mapped `10/36` tracks with `26/36` offset fallbacks, `StudentTracks`
regressed `53 -> 64`, minimum model-agreement F1@IoU0.5 stayed `53.730 %`, and
minimum shard-1 global-assignment F1 stayed `79.876 %`. Evidence:
[docs/agent_19_cycle_18_turn.md](agent_19_cycle_18_turn.md) and
[docs/cycle_18_redis_boundary_state_cache_investigation.md](cycle_18_redis_boundary_state_cache_investigation.md).

The deep root-cause audit
([docs/cycle_18_identity_association_root_cause_investigation.md](cycle_18_identity_association_root_cause_investigation.md))
isolated the failure mechanism. This cycle implements the redesign the prior
handoff required.

## Blocker → Root Cause → Cycle 18.D Response

| Cycle 18.C blocker | Root cause | Cycle 18.D response |
|---|---|---|
| Merge-ready packets `1/2` | Shard-1 association left most tracks unresolved. | Fuse geometry + appearance + motion so confident non-appearance matches still resolve; write packet identity for assigned tracks. |
| `StudentTracks 53 -> 64` | `26` unmapped shard-1 tracks took offset IDs that inflate canonical count. | Better association reduces fallbacks; ambiguous tracks stay `unresolved` (§4.6) instead of forcing/duplicating a merge. |
| Shard-1 mapping `10/36` | `appearance_packet` fails closed whenever the weak `cv2_16x16_rgb_descriptor` is unavailable or below the `0.90` cosine gate. | A missing modality is dropped and remaining evidence re-normalised; a strong geometry + motion agreement can associate without appearance. |
| Min model-agreement F1 `53.730 %` | Invalid raw-label equality gate plus real association loss. | Evaluation stays label-invariant (probe unchanged); this cycle targets the real association loss only. |
| Min shard-1 global-assignment F1 `79.876 %` | Fragmentation/merge ambiguity from geometry-only voting over a context window. | Global one-to-one assignment over fused cost reduces both fragmentation and merge collisions; no parent is ever reused. |

## Design

The new mode is `OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=combined_cost`.

1. **Candidate generation** reuses the existing per-frame boundary IoU vote rows
   (`_boundary_vote_rows`). A child track only proposes parents it geometrically
   co-located with at shared boundary frames.
2. **Motion evidence** (`_boundary_motion_score`) compares per-step displacement
   vectors of the child and candidate-parent boxes across shared frames,
   normalised by box diagonal. It separates two people who momentarily overlap
   but move apart. Fewer than two shared frames returns `unavailable` (not zero),
   per constitution §1.6.
3. **Appearance evidence** reuses the governed packet appearance prototypes
   (`_load_appearance_vector` + cosine). When unavailable it is dropped, not
   scored as zero.
4. **Available-component fusion** computes
   `combined = Σ(wₖ·scoreₖ) / Σ(wₖ)` over only the available modalities, so a
   missing appearance descriptor no longer collapses an otherwise strong match.
   Default weights: geometry `0.40`, appearance `0.40`, motion `0.20`.
5. **Global one-to-one assignment** (`_min_cost_one_to_one`, min-cost max-flow)
   selects a globally optimal, collision-free child→parent mapping. Ambiguous,
   below-threshold (`combined_score < 0.55`), low-margin (`< 0.05`), or
   geometry-gate-failing tracks remain `unresolved` and fall to offset
   namespace allocation — recorded as namespace management, not a merge.
6. **Explainability**: every candidate records geometry/appearance/motion/combined
   components and an append-only decision reason.

This follows the Deep SORT / BoT-SORT / clip-association doctrine already cited
in the root-cause investigation: combine motion and appearance, reason over
bounded tracklets, and keep ambiguity unresolved.

## Source-of-Truth References

| Kind | Reference | Role |
|---|---|---|
| Runtime | `backend/apps/video_analysis/services/offline_sharding.py` | `_boundary_motion_score`, `_min_cost_one_to_one`, `_combined_cost_assignment`, `_track_map_config`, `_build_track_map`. |
| Settings | `backend/config/settings/base.py` | Default-off `OFFLINE_VIDEO_SHARD_COMBINED_*` weights and thresholds. |
| Tests | `backend/tests/unit/video_analysis/test_cycle15b1_shard_merge.py` | `test_cycle18d_*` matcher, motion, fusion, and unresolved-contention coverage. |
| Profile reset | `tools/prod/prod_enable_parallel_flow.sh` | Restores combined-cost defaults with the accepted non-sharded profile. |
| Root cause | `docs/cycle_18_identity_association_root_cause_investigation.md` | Confirmed root causes and required future-candidate contract. |
| Prior decision | `docs/cycle_18_redis_boundary_state_cache_investigation.md` | Cycle 18.B/18.C NOT-ACCEPTED evidence. |
| Constitution | `.specify/memory/constitution.md` §4.6 / §8.6 / §12.5 / §12.6 | Association doctrine, streaming guard, and benchmark decision authority. |

## Local Validation

| Check | Result |
|---|---|
| `py_compile` of `offline_sharding.py`, `base.py`, focused test | Passed. |
| `test_cycle18d_*` focused tests | Passed: `4` (matcher contention, motion separation, geometry+motion assignment without appearance, contested-track unresolved). |
| Full `test_cycle15b1_shard_merge.py` suite | Passed: `15`; existing `best_iou` / `majority_vote` / `one_to_one` / `appearance_packet` paths unchanged. |

Local tests validate decision **logic and shape only**. They cannot establish
identity correctness — that requires the production benchmark below.

## 2026-06-05 Production Benchmark Lock

```text
BENCHMARK_LOCK
agent: 18
cycle: 18.D combined-cost boundary association
state: HELD
claimed_at_utc: 2026-06-05T17:41:15Z
replay_key: cycle18d-combined-cost-20260605T174115Z
baseline_metrics: /home/bamby/grad_project/backend/logs/cycle15b-pre-shard-baseline-20260603T193531Z/metrics.json
candidate_env_delta: OFFLINE_VIDEO_SHARDING_ENABLED=1, OFFLINE_VIDEO_SHARD_COUNT=2, OFFLINE_VIDEO_SHARD_CONTEXT_FRAMES=256, OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=combined_cost, OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_ENABLED=1, OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_APPEARANCE_ENABLED=1, TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=1, EMBEDDING_PREFETCH_TRACK_LOOKUP=1, EMBEDDING_REDIS_SIDE_EFFECT_COALESCING=1
expected_cleanup: OFFLINE_VIDEO_SHARDING_ENABLED=0, OFFLINE_VIDEO_SHARD_COUNT=1, OFFLINE_VIDEO_SHARD_CONTEXT_FRAMES=32, OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=best_iou, OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_ENABLED=0, OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_APPEARANCE_ENABLED=0, Celery workers restarted
required_evidence: metrics_json, metrics_md, sharded_summary_json, gpu_csv, boundary_packet_validation_json_md, model_agreement_json_md, label_invariant_json_md, rollback_json_md, figure_manifest, figures_md, generated_pngs, production_benchmark_section
decision_authority: NO_DECISION_UNTIL_SECTION_12_6_EVIDENCE_TABLE
```

Figure evidence roles for this run:

| Role | Owner | Evidence boundary |
|---|---|---|
| Figure Planner | Agent 18 current benchmark session | Required plots: decision delta, wall breakdown, correctness gate, packet budget/readiness, identity label-invariant, model RTT, GPU/resource, Redis/resource availability, unavailable summary. Inputs are the baseline metrics JSON, candidate metrics JSON/MD, sharded summary, GPU CSV, boundary packet validation, model agreement, label-invariant tracking, and rollback status from this replay. Missing metrics must render as `unavailable`, not zero. Markdown embeds target this document and `docs/production_inference_benchmark.md`. |
| Figure Implementer | Agent 18 current benchmark session | No generator code change is planned before the run. The implementation evidence is the existing `tools/prod/prod_generate_cycle_figures.py` invocation through `tools/prod/prod_run_cycle15b1_two_shard_runtime_benchmark.sh`, the generated figure manifest/digests, generated PNGs, and wrapper-produced `figures.md`. |

Role separation is unavailable in this session because the user requested this
agent to run the production benchmark directly and sub-agent spawning was not
explicitly requested. The plan and implementation evidence are therefore kept
as separate rows above per constitution §7.1.1 / §12.6.

## 2026-06-05 Production Benchmark Result

```text
BENCHMARK_RELEASE
agent: 18
cycle: 18.D combined-cost boundary association
state: CYCLE_18D_BENCHMARK_LOCK_RELEASED / CYCLE_18D_NOT_ACCEPTED
released_at_utc: 2026-06-05T17:54:05Z
replay_key: cycle18d-combined-cost-20260605T174115Z
candidate_code_sha: d976817b
parent_job_id: 94098d79-fed1-4a67-a0c6-9f0f067f2990
child_job_ids: a3d6e334-08bf-48d3-a804-1f86a7dcca33, 9c68766d-6b91-4090-af51-04c8180eff50
status: completed_but_candidate_not_accepted
metrics_json: /home/bamby/grad_project/backend/logs/cycle18d-combined-cost-20260605T174115Z/metrics.json
metrics_md: /home/bamby/grad_project/backend/logs/cycle18d-combined-cost-20260605T174115Z/metrics.md
sharded_summary_json: /home/bamby/grad_project/backend/logs/cycle18d-combined-cost-20260605T174115Z/sharded_summary.json
gpu_csv: /home/bamby/grad_project/backend/logs/cycle18d-combined-cost-20260605T174115Z/gpu_monitor.csv
packet_validation_json: /home/bamby/grad_project/backend/logs/cycle18d-combined-cost-20260605T174115Z/boundary_packet_validation.json
model_agreement_json: /home/bamby/grad_project/backend/logs/cycle18d-combined-cost-20260605T174115Z/model_agreement.json
label_invariant_json: /home/bamby/grad_project/backend/logs/cycle18d-combined-cost-20260605T174115Z/label_invariant_tracking.json
rollback_json: /home/bamby/grad_project/backend/logs/cycle18d-combined-cost-20260605T174115Z/rollback_status.json
figure_manifest: docs/figures/benchmark_artifacts/cycle18d-combined-cost-20260605T174115Z/figure_manifest.json
figure_markdown: /home/bamby/grad_project/backend/logs/cycle18d-combined-cost-20260605T174115Z/figures.md
rollback_verified: true
```

### Cycle 18.D Production Metrics

| Metric | Accepted pre-shard baseline | Cycle 18.C prior sharded profile | Cycle 18.D candidate | Delta vs baseline | Delta vs 18.C |
|---|---:|---:|---:|---:|---:|
| DB-completed FPS | `5.619787` | `7.477400` | `7.502768` | `+33.51 %` | `+0.34 %` |
| DB completed elapsed | `808.038 s` | `607.297 s` | `605.243 s` | `-25.10 %` | `-0.34 %` |
| Step 2 frame wall | `467.449833 s` | `244.490729 s` | `244.259645 s` | `-47.75 %` | `-0.09 %` |
| Step 2 through-pose wall | `641.154064 s` | `369.948615 s` | `370.990517 s` | `-42.14 %` | `+0.28 %` |
| GPU average utilization | `11.846 %` | `19.177 %` | `17.035 %` | `+43.80 %` | `-11.17 %` |
| GPU peak utilization | `57.000 %` | `93.000 %` | `85.000 %` | `+49.12 %` | `-8.60 %` |
| Detection rows | `72744` | `72816` | `72816` | `+0.10 %` | `0.00 %` |
| BBox rows | `72744` | `72816` | `72816` | `+0.10 %` | `0.00 %` |
| Embedding rows | `72578` | `72650` | `72650` | `+0.10 %` | `0.00 %` |
| StudentTracks | `53` | `64` | `56` | `+5.66 %` | `-12.50 %` |
| Behavior RTT mean | `83.530 ms` | unavailable | `91.495 ms` | `+9.54 %` | unavailable |
| Behavior RTT p95 | `129.514 ms` | unavailable | `149.335 ms` | `+15.30 %` | unavailable |

Cycle 18.D preserved the sharded throughput envelope and improved StudentTrack
count versus Cycle 18.C, but correctness gates still failed.

### Cycle 18.D Correctness Gates

| Gate | Cycle 18.C | Cycle 18.D | Result |
|---|---:|---:|---|
| Valid boundary packets | `2/2` | `1/2` | Fails packet-validity gate |
| Merge-ready boundary packets | `1/2` | `0/2` | Fails identity-merge gate |
| Shard-1 mapped to existing parent IDs | `10/36` | `18/36` | Improves coverage but still leaves half unmapped |
| Shard-1 offset fallbacks | `26/36` | `18/36` | Improves fallback rate but still fails |
| Minimum model-agreement F1@IoU0.5 | `53.730 %` | `53.788 %` | Fails model-agreement gate |
| Minimum all-model global-assignment F1 | `69.830 %` | `72.414 %` | Improves but fails label-invariant identity gate |
| Minimum shard-1 global-assignment F1 | `79.876 %` | `79.876 %` | Residual shard-1 association gap unchanged |
| Minimum shard-1 raw-label F1 | `2.917 %` | `3.581 %` | Local-ID discontinuity remains |
| Rollback verified | `true` | `true` | Pass safety gate |

Boundary packet validation shows why the candidate cannot be accepted:

| Child job | Packet valid | Merge-ready | Packet bytes | Tracks | Observations | Unresolved tracks | Key reason |
|---|---|---|---:|---:|---:|---:|---|
| `a3d6e334-08bf-48d3-a804-1f86a7dcca33` | `true` | `false` | `181162` | `24` | `1128` | `24` | First-shard packet is not identity-merge-ready. |
| `9c68766d-6b91-4090-af51-04c8180eff50` | `false` | `false` | `236173` | `24` | `1424` | `6` | Schema rejected new `combined_score` / `motion_score` candidate fields, and unresolved tracks remain. |

Cycle 18.D therefore reveals a new contract gap: the runtime emitted useful
combined-cost diagnostics, but the governed boundary-packet schema did not
accept the new candidate fields. Even if the schema gap were fixed, the
remaining unresolved tracks and shard-1 residual association gap would still
block acceptance.

### Cycle 18.D Figure Evidence

![Cycle 18.D Decision Delta](figures/benchmark_artifacts/cycle18d-combined-cost-20260605T174115Z/cycle18d_combined_cost__decision_delta.png)

![Cycle 18.D Packet Readiness](figures/benchmark_artifacts/cycle18d-combined-cost-20260605T174115Z/cycle18d_combined_cost__packet_readiness.png)

![Cycle 18.D Packet Budget](figures/benchmark_artifacts/cycle18d-combined-cost-20260605T174115Z/cycle18d_combined_cost__packet_budget.png)

![Cycle 18.D Identity Label-Invariant](figures/benchmark_artifacts/cycle18d-combined-cost-20260605T174115Z/cycle18d_combined_cost__identity_label_invariant.png)

![Cycle 18.D Correctness Gate](figures/benchmark_artifacts/cycle18d-combined-cost-20260605T174115Z/cycle18d_combined_cost__correctness_gate.png)

![Cycle 18.D GPU Profile](figures/benchmark_artifacts/cycle18d-combined-cost-20260605T174115Z/cycle18d_combined_cost__gpu_profile.png)

### Cycle 18.D Final Decision

Decision: **NOT ACCEPTED**. The candidate completed `4541/4541` frames and
improved DB FPS versus the accepted pre-shard baseline (`5.619787 -> 7.502768`)
while reducing Step 2 frame wall (`467.449833 s -> 244.259645 s`). It also
improved shard-1 mapping versus Cycle 18.C (`10/36 -> 18/36`) and reduced
StudentTracks (`64 -> 56`), but it failed the required identity and evidence
contracts: only `1/2` packets was schema-valid, `0/2` packets was merge-ready,
`18/36` shard-1 tracks still fell back to offset IDs, minimum model-agreement
F1 stayed at `53.788 %`, minimum shard-1 global-assignment F1 stayed
`79.876 %`, and rollback was required to restore the accepted profile.

No sharding gain is accepted from Cycle 18.D. Cycle 15.B1 and 15.B2 remain
blocked. Next work must update the Cycle 18 boundary packet schema for the
combined-cost fields and then reduce unresolved/offset fallback tracks enough
to pass packet validity, merge readiness, model agreement, and label-invariant
identity in a new production benchmark.

## Required Production Benchmark Gate

`combined_cost` cannot be accepted until a completed native-Linux RTX 5090
two-shard `combined.mp4` benchmark records, versus the accepted baseline:

| Gate | Required evidence |
|---|---|
| Identity correctness | Ground-truth-backed HOTA/AssA, IDF1, ID switches, fragmentation; label-invariant global-assignment F1 reported separately and not used alone to accept. |
| Mapping coverage | Shard-1 offset-fallback rate materially reduced; `StudentTracks` returns toward the accepted `53`. |
| Model agreement | Minimum F1@IoU0.5 recovered to the acceptance threshold. |
| DB parity | Frames, detections, boxes, embeddings, and terminal state preserved. |
| Performance | FPS, Step 2 wall, RTT, GPU, and memory deltas with figure bundle + manifest. |
| Rollback | One env/profile change restores `best_iou` and disabled sharding. |

Until then this cycle is `STAGED_LOCAL_ONLY`. Re-running a previously failed
profile as new evidence is forbidden; `combined_cost` is a distinct mechanism,
so its first completed benchmark is the only valid decision point.

## Rollback

`combined_cost` is inert unless explicitly selected. To disable, set
`OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=best_iou` (and keep
`OFFLINE_VIDEO_SHARDING_ENABLED=0`) and restart Celery workers. No persisted
baseline evidence is mutated by staging this code.
