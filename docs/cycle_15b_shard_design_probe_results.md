# Cycle 15.B Shard Design Probe Results

**Last updated:** 2026-06-04

**Status:** DESIGN PROOF PASSED / TWO-SHARD RUNTIME NOT ACCEPTED /
BOUNDARY-STITCHING SUBCYCLE ACTIVE.

**Streaming compatibility:** `offline-only`. The probe and every future
sharded runtime candidate MUST remain disabled for RTSP, RTSPS, WHEP/WebRTC,
HLS fallback, and all live stream profiles.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Parent investigation | `docs/cycle_15b_video_sharding_design_proof_investigation.md` | Defines the sharding safety contract. |
| 15.B1 investigation | `docs/cycle_15b1_two_shard_design_proof_investigation.md` | Defines the two-shard scenario. |
| 15.B2 investigation | `docs/cycle_15b2_four_shard_design_proof_investigation.md` | Defines the four-shard scenario. |
| Shard planner | `tools/prod/prod_plan_video_shards.py` | Computes deterministic shard ranges. |
| Probe wrapper | `tools/prod/prod_run_cycle15b_shard_design_probe.sh` | Production read-only probe runner. |
| Unit test | `backend/tests/unit/pipeline/test_prod_plan_video_shards.py` | Verifies exact coverage and overhead math. |
| Production evidence | `/home/bamby/grad_project/backend/logs/cycle15b-shard-design-20260603T191500Z/shard_plan.json` | Raw production shard plan. |
| Production markdown | `/home/bamby/grad_project/backend/logs/cycle15b-shard-design-20260603T191500Z/shard_plan.md` | Human-readable production evidence. |
| Pre-shard baseline metrics | `/home/bamby/grad_project/backend/logs/cycle15b-pre-shard-baseline-20260603T193531Z/metrics.json` | Full production baseline metrics for the accepted single-job runtime before any sharded runtime implementation. |
| Pre-shard baseline markdown | `/home/bamby/grad_project/backend/logs/cycle15b-pre-shard-baseline-20260603T193531Z/metrics.md` | Human-readable baseline evidence bundle. |
| Stitching probe helper | `tools/prod/prod_analyze_cycle15b1_stitching.py` | Read-only helper that separates geometry agreement from track-sensitive identity agreement. |
| Stitching probe evidence | `/home/bamby/grad_project/backend/logs/cycle15b1c-stitching-probe-20260603T220206Z/stitching_probe.json` | Production proof that 15.B1 failed due to shard-1 identity labels, not geometry. |

## Production Evidence

| Field | Value |
|---|---|
| Deployed SHA | `f55fb85` |
| Evidence directory | `/home/bamby/grad_project/backend/logs/cycle15b-shard-design-20260603T191500Z` |
| Video | `/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4` |
| Frames | `4541` |
| FPS | `30.0` |
| Overlap policy | `pre_roll_only_context_not_persisted` |
| Runtime mutation | none |
| Inference jobs submitted | none |

## Scenario Comparison

| Scenario | Shards | Authoritative frames | Decoded frames | Context-only frames | Context overhead | Coverage |
|---|---:|---:|---:|---:|---:|---|
| 15.B1 | `2` | `4541` | `4573` | `32` | `0.704691 %` | `True` |
| 15.B2 | `4` | `4541` | `4637` | `96` | `2.114072 %` | `True` |

## 15.B1 Two-Shard Ranges

| Shard | Authoritative range | Decode range | Pre-roll | Authoritative | Decoded | Context-only |
|---:|---|---|---:|---:|---:|---:|
| `0` | `[0, 2271)` | `[0, 2271)` | `0` | `2271` | `2271` | `0` |
| `1` | `[2271, 4541)` | `[2239, 4541)` | `32` | `2270` | `2302` | `32` |

## 15.B2 Four-Shard Ranges

| Shard | Authoritative range | Decode range | Pre-roll | Authoritative | Decoded | Context-only |
|---:|---|---|---:|---:|---:|---:|
| `0` | `[0, 1136)` | `[0, 1136)` | `0` | `1136` | `1136` | `0` |
| `1` | `[1136, 2271)` | `[1104, 2271)` | `32` | `1135` | `1167` | `32` |
| `2` | `[2271, 3406)` | `[2239, 3406)` | `32` | `1135` | `1167` | `32` |
| `3` | `[3406, 4541)` | `[3374, 4541)` | `32` | `1135` | `1167` | `32` |

## Decision Boundary

| Candidate | Decision | Reason |
|---|---|---|
| 15.B1 two-shard | DESIGN PROOF PASSED | Exact authoritative coverage with the lowest context overhead. |
| 15.B2 four-shard | DESIGN PROOF PASSED, NOT FIRST IMPLEMENTATION | Exact coverage, but 3x the context-only frames and 3 shard boundaries. |
| Runtime sharding | BLOCKED | Track stitching, duplicate suppression, DB idempotency, parent/shard terminal state, and rollback are not implemented. |

## Next Step

Start a narrow 15.B1 runtime implementation plan only after specifying:

| Required design item | Minimum rule |
|---|---|
| Parent job | Owns terminal state and aggregate evidence. |
| Shard jobs | Must persist only authoritative frame ranges. |
| Context frames | Decode only; no authoritative DB writes. |
| Idempotency | Writes key on parent job, frame index, model label, and semantic detection identity. |
| Track stitching | Boundary tracks merge only with explicit evidence; otherwise stay unresolved. |
| Live profile | Explicitly disabled; no RTSP/RTSPS/live route may inherit sharding. |

No FPS, latency, throughput, or GPU-utilization improvement can be claimed from
this dry-run. Those require a future full production benchmark after a sharded
runtime candidate exists.

## Pre-Shard Baseline Benchmark

This benchmark was started after the design proof to create the required
single-job comparator for a future 15.B1 runtime implementation. It did not
run sharded inference and therefore does not accept, reject, skip, or close
runtime sharding.

| Field | Value |
|---|---|
| Replay key | `cycle15b-pre-shard-baseline-20260603T193531Z` |
| Job id | `74561b05-105f-4ca8-aeaf-f510f4f802de` |
| Runtime deployed SHA | `74afc9d` |
| Video | `/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4` |
| Pipeline mode | `crop_frame` |
| Status | `completed` |
| Evidence directory | `/home/bamby/grad_project/backend/logs/cycle15b-pre-shard-baseline-20260603T193531Z` |

| Metric | Value |
|---|---:|
| Processed frames | `4541/4541` |
| DB completed elapsed | `808.038 s` |
| DB completed FPS | `5.620` |
| Step 2 frame wall | `467.450 s` |
| Step 2 frame-loop FPS | `9.714` |
| Step 2 through-pose wall | `641.154 s` |
| Run complete wall | `707.716 s` |
| GPU average utilization | `11.846 %` |
| GPU peak utilization | `57.000 %` |
| Peak VRAM | `15725.000 MiB` |
| Behavior ensemble mean RTT | `83.530 ms` |
| Behavior ensemble p95 RTT | `129.514 ms` |
| Pose tail over frame loop | `173.704 s` |
| Embedding created span | `98.578 s` |
| Detection rows | `72744` |
| Bounding-box rows | `72744` |
| Embedding rows | `72578` |
| Student tracks | `53` |

| BBox model | Rows |
|---|---:|
| `attention_tracking` | `11770` |
| `hand_raising` | `8799` |
| `person_detection` | `19162` |
| `sitting_standing` | `33013` |

| Model | Calls | Mean RTT | P95 RTT | Shape |
|---|---:|---:|---:|---|
| `behavior_ensemble_gaze_slice_topk` | `3597` | `83.530 ms` | `129.514 ms` | `[32, 3, 320, 320]` |
| `rtmpose_model` | `1199` | `44.778 ms` | `47.186 ms` | `[16, 3, 256, 192]` |
| `person_detector` | `910` | `12.394 ms` | `19.478 ms` | `[1, 3, 640, 640]` |

| Decision item | Result |
|---|---|
| Runtime sharding decision | `NO_DECISION_BASELINE_ONLY` |
| Why no acceptance | No sharded runtime candidate was implemented or run. |
| What this benchmark enables | Future 15.B1 runtime must beat `5.620` DB FPS, `467.450 s` Step 2 frame wall, `641.154 s` through-pose wall, `83.530 ms` behavior RTT mean, and preserve DB/model parity. |
| Observed bottleneck signal | The completed baseline still spends `173.704 s` after the frame loop in pose-tail/upload work and `98.578 s` creating embeddings, so sharding must account for post-frame coordination rather than only splitting the frame loop. |

## 15.B1 Runtime Readiness

Cycle 15.B1 runtime Phase A is now tracked in
`docs/cycle_15b1_two_shard_runtime_investigation.md`. Production readiness
evidence at
`/home/bamby/grad_project/backend/logs/cycle15b1-runtime-readiness-20260603T200611Z`
returned `blocked_no_runtime_candidate` with `8` critical blockers and `0`
warnings. A two-shard runtime benchmark remains invalid until the missing env
gate, authoritative frame-window support, parent/shard lineage, runtime wrapper,
merge helper, and row-idempotency blockers are implemented.

Follow-up safe-default slice: settings/profile management now declare the
disabled sharding env keys, `tools/prod/prod_set_cycle15b1_sharding_defaults.sh`
sets only those keys, and `process_video_upload` fails closed with
`cycle15b1_sharding_runtime_not_implemented` if sharding is requested before a
runtime exists. The readiness audit is still `blocked_no_runtime_candidate`
with `5` critical blockers: runtime candidate, benchmark wrapper, parent merge
helper, detection/bbox provenance, and embedding provenance. Production
deployed SHA `74631e6` and wrote the safe-default readiness audit to
`/home/bamby/grad_project/backend/logs/cycle15b1-runtime-readiness-safe-default-20260603T202308Z`.

Superseding implementation note (2026-06-03): the five safe-default blockers
are now resolved locally by the Cycle 15.B1 runtime candidate. The implemented
candidate adds `offline_sharding.py`, the `cycle15b1_sharded_ingest`
management command, `prod_merge_cycle15b1_shards.py`, the
`prod_run_cycle15b1_two_shard_runtime_benchmark.sh` wrapper, and migration
`0014_cycle15b1_shard_provenance.py`. Local readiness now reports
`ready_for_runtime_benchmark=True` with `0` blockers. This does not change the
baseline decision: runtime sharding remains `NOT DECIDED` until the real
production benchmark completes and writes metrics/parity evidence.

Superseding runtime benchmark decision (2026-06-03): Cycle 15.B1 two-shard
runtime is now **NOT ACCEPTED** by production benchmark. Replay
`cycle15b1-two-shard-runtime-repeat-20260603T211319Z` / parent job
`e602a0ca-6efc-4cb0-8d30-9466fe76287b` completed `4541/4541` frames and proved
the expected parallelism: DB FPS `5.620 → 7.867`, Step 2 critical-path frame
wall `467.450 s → 233.038 s`, through-pose wall `641.154 s → 324.763 s`, GPU
average `11.846 % → 17.495 %`, and GPU peak `57 % → 89 %`. It failed the
correctness gate: StudentTracks changed `53 → 52`, behavior RTT mean regressed
`83.530 ms → 89.718 ms`, and baseline-agreement F1@IoU0.5 fell to
`59.473 %` (`attention_tracking`), `60.700 %` (`hand_raising`),
`61.387 %` (`person_detection`), and `53.455 %` (`sitting_standing`). Evidence
is under
`/home/bamby/grad_project/backend/logs/cycle15b1-two-shard-runtime-repeat-20260603T211319Z/`.
The design proof remains useful, but runtime sharding must stay disabled until
boundary identity/track stitching is fixed and benchmarked.

Superseding boundary-stitching probe (2026-06-04): Cycle 15.B1.C has started
as `PROBE_ONLY` evidence. The helper
`tools/prod/prod_analyze_cycle15b1_stitching.py` compared the accepted
pre-shard baseline job `74561b05-105f-4ca8-aeaf-f510f4f802de` against the
rejected sharded parent job `e602a0ca-6efc-4cb0-8d30-9466fe76287b`. Evidence:
`/home/bamby/grad_project/backend/logs/cycle15b1c-stitching-probe-20260603T220206Z/stitching_probe.json`
and `.md`. The probe shows shard 0 preserves track labels, while shard 1 keeps
geometry but loses identity: shard-1 geometry F1 is `100.000 %` for the four
persisted models, but track-sensitive F1 is `4.043 %`, `2.974 %`,
`21.308 %`, and `4.124 %`. 15.B2 remains blocked until a concrete
identity-stitching candidate passes a full production benchmark.

Deep stitching follow-up (2026-06-04): helper commit `3baa4cdc` adds an oracle
relabeling upper-bound analysis. Production evidence:
`/home/bamby/grad_project/backend/logs/cycle15b1c-deep-stitching-20260603T221605Z/deep_stitching_probe.json`
and `.md`. Shard-1 oracle relabeling improves track F1 to `75.458 %`,
`72.531 %`, `56.733 %`, and `84.021 %` for attention, hand, person, and
sitting/standing respectively, but does not restore parity. The next executable
subcycle remains inside 15.B1.C: run a full production context-window benchmark
with `--context-frames 256`; only if that fails should the system build a
guarded canonicalizer with ambiguity gates.

Context-window benchmark follow-up (2026-06-04): 15.B1.C1 ran as
`cycle15b1c1-context256-20260603T222123Z`, parent job
`401498f1-d5e4-4b95-8a46-ad3fcbbc2c25`. It completed and improved throughput
but is **NOT ACCEPTED** because model-agreement F1 remained `53.730 %` -
`61.767 %`, `StudentTracks` remained `52` vs baseline `53`, and behavior RTT
regressed. The next sharding work is 15.B1.C2 majority-vote canonicalization
behind `OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=majority_vote`; 15.B2 remains
blocked.

Majority-vote benchmark follow-up (2026-06-04): 15.B1.C2 ran as
`cycle15b1c2-majority-vote-20260603T223932Z`, parent job
`78388c2c-d7f5-42b7-afa4-321216d23b11`, and is also **NOT ACCEPTED**. It
completed and preserved throughput, but shard 1 mapped only `10/36` tracks to
existing parent IDs, `StudentTracks` increased to `64`, and model-agreement F1
remained `53.730 %` - `61.109 %`. Cycle 15.B2 remains blocked; further
sharding requires a new identity-state design proof before runtime benchmarking.
