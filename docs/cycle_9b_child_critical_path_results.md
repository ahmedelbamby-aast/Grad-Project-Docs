# Cycle 9b Child Critical-Path Results

**Status:** **STEP 1 MEASUREMENT COMPLETE — NO OPTIMIZATION ACCEPTED.**

This document records the required Cycle 9b B.3 Step 1 measurement: identify
which behavior child dominates the `behavior_ensemble` critical path before
choosing any child-specific optimization. No code or engine change is accepted
by this document.

## Evidence

| Item | Value |
|---|---|
| Production run used for post-run stats | `cycle10-lpm-violationonly-crop-frame-20260601T221110` |
| Job ID | `21666815-f4bd-4f5f-b90e-b9101b4d899d` |
| Deployed SHA during run | `31edac44c66233baadd3a26ddd57b51b1a043d66` |
| Production status | `completed` |
| Triton mode | offline `39100/39101/39102` |
| Behavior input | `TRITON_CROP_BEHAVIOR_INPUT_SIZE=320` |
| Ensemble flag | `TRITON_BEHAVIOR_ENSEMBLE=1` through the optimized flow |
| Direct probe | `backend/logs/probe_rtt_decompose_cycle9b_child_20260602T.json` |
| Probe command | `backend/.venv/bin/python tools/prod/probe_rtt_decompose.py --iters 20 --n-crops 17 --size 320 --mode both --host 127.0.0.1:39101 --json backend/logs/probe_rtt_decompose_cycle9b_child_20260602T.json` |

## Production Triton Stats Snapshot

Captured immediately after the production run. `success.ns / count` is usable;
the child-model `compute_*` fields were `0` and child `queue.ns` was clearly
invalid in this Triton ensemble build, so they are not used for the decision.

| Child model | Executions | Success avg / exec | Top batch bucket |
|---|---:|---:|---:|
| `posture_model` | `3599` | `30.378 ms` | `bs=32`, `1782` executions |
| `gaze_horizontal_model` | `3599` | `31.725 ms` | `bs=32`, `1782` executions |
| `gaze_vertical_model` | `3599` | `27.933 ms` | `bs=32`, `1782` executions |
| `gaze_depth_model` | `3598` | `30.332 ms` | `bs=32`, `1782` executions |
| `behavior_ensemble` | `3597` | `39.077 ms` | `bs=32`, `1782` executions |

Batching is already frequently hitting the deployed ensemble cap: `bs=32` was
the dominant bucket for all four children. This weakens larger-batch tuning as
the first Cycle 9b lever.

## Direct RTT Decomposition Probe

The direct probe sends the production-shaped tensor
`[17, 3, 320, 320]` (`20.89 MB`) to each behavior child for 20 iterations.

### Sequential Calls

| Model | RTT mean | RTT p95 | Serialize mean | Transport+server mean | Deserialize mean |
|---|---:|---:|---:|---:|---:|
| `posture_model` | `44.37 ms` | `52.91 ms` | `2.56 ms` | `41.67 ms` | `0.14 ms` |
| `gaze_horizontal_model` | `50.79 ms` | `62.77 ms` | `1.99 ms` | `47.84 ms` | `0.95 ms` |
| `gaze_vertical_model` | `42.10 ms` | `46.60 ms` | `2.00 ms` | `39.96 ms` | `0.14 ms` |
| `gaze_depth_model` | `43.31 ms` | `49.71 ms` | `1.96 ms` | `41.19 ms` | `0.15 ms` |

### Concurrent Calls

| Model | RTT mean | RTT p95 | Serialize mean | Transport+server mean | Deserialize mean |
|---|---:|---:|---:|---:|---:|
| `posture_model` | `131.08 ms` | `139.18 ms` | `2.09 ms` | `128.78 ms` | `0.20 ms` |
| `gaze_horizontal_model` | `128.10 ms` | `132.20 ms` | `6.58 ms` | `120.39 ms` | `1.13 ms` |
| `gaze_vertical_model` | `115.35 ms` | `118.94 ms` | `6.53 ms` | `108.73 ms` | `0.09 ms` |
| `gaze_depth_model` | `117.82 ms` | `128.45 ms` | `6.45 ms` | `111.26 ms` | `0.11 ms` |

### Server Delta From Probe

The probe also captured Triton stats deltas over the probe window:

| Model | Delta executions | Server avg / exec |
|---|---:|---:|
| `posture_model` | `40` | `12.133 ms` |
| `gaze_horizontal_model` | `40` | `16.058 ms` |
| `gaze_vertical_model` | `40` | `11.759 ms` |
| `gaze_depth_model` | `40` | `11.909 ms` |

## Decision

**Dominant child:** `gaze_horizontal_model`.

Evidence:

- It has the highest direct server delta: `16.058 ms/exec`.
- It is `32.4 %` slower than `posture_model`, `36.6 %` slower than
  `gaze_vertical_model`, and `34.8 %` slower than `gaze_depth_model`.
- It also has the widest dense output tensor (`[84, 2100]`), which matches
  the Cycle 9 post-mortem and the tensor anatomy document.

## Next Implementation Target

The next Cycle 9b implementation should target `gaze_horizontal_model` first,
preferably through the low-risk output-fusion path already specified as B.2.b:
re-export or otherwise route a narrow gaze-horizontal head instead of returning
the inherited `84`-channel dense output.

No optimization is accepted yet. The next implementation must still produce:

1. Accuracy parity evidence.
2. Production benchmark before/after.
3. Step 2 wall and DB-completed FPS comparison.
4. Per-class DB parity, especially `attention_tracking`.
5. Rollback through an env-controlled engine/output variant.
