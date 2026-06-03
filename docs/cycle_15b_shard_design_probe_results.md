# Cycle 15.B Shard Design Probe Results

**Last updated:** 2026-06-03

**Status:** DESIGN PROOF PASSED / RUNTIME IMPLEMENTATION STILL BLOCKED. No
sharded inference runtime has been implemented, production-benchmarked,
accepted, rejected, skipped, or closed.

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
