# Cycle 14.D Server-Side Compact Postprocessing Investigation

**Last updated:** 2026-06-03

**Status:** PHASE A STARTED. No implementation, benchmark decision, acceptance,
rejection, skip, or closure exists for Cycle 14.D yet.

## Problem Statement

Cycle 14.C3 closed the batch-32 follow-up as not accepted. Accepted production
therefore remains Cycle 14.B2 batch `16`. The next sorted cycle in
`docs/crop_frame_optimization_execution.md` is Cycle 14.D:
server-side compact postprocessing / BLS / TRT plugin that reduces wait or
server execution, not only output bytes.

This cycle must not repeat the already-rejected pattern of optimizing bytes or
request count without improving the dominant wall-time metrics. It must first
split the candidate space, prove which scenario is plausible in the current
production Triton runtime, and only then implement one sub-cycle.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Current accepted batch | `docs/cycle_14b_rtmpose_scenario_results.md` | Batch `16` is the accepted RTMPose cross-frame batching profile. |
| Batch-size matrix | `docs/cycle_14c_pose_batch_size_matrix_results.md` | Batch `8`, batch `32`, and C3 are all not accepted. |
| C3 result | `docs/cycle_14c3_batch32_parallel_chunks_results.md` | Proves internal chunk parallelism worsened RTMPose latency. |
| Compact prior work | `docs/cycle_9b_compact_postproc_results.md` | Proves Python decode/NMS was small after Top-K and no compact backend has been accepted. |
| Execution log | `docs/crop_frame_optimization_execution.md` | Sorts Cycle 14.D after Cycle 14.C. |
| Plan doc | `docs/inference_parallelization_plan.md` | Requires production benchmark authority for any decision. |
| Triton runtime code | `backend/apps/pipeline/services/pose_runtime.py` | Shows the current RTMPose provider boundary after accepted batch `16`. |
| Behavior route code | `backend/apps/video_analysis/tasks.py` | Owns behavior and pose orchestration around crop-frame inference. |
| Triton repository | `backend/models/triton_repository_cuda12/` | Determines whether a BLS, ensemble, or TRT-only implementation is feasible. |

## Current Evidence Boundary

The accepted batch-16 baseline remains the comparison authority:
`cycle14b-cross-frame-batch16-r2-20260603T150000Z`.

Cycle 14.C3 also provides a fresh caution: increasing parallel RTMPose
pressure can preserve correctness while regressing latency. Therefore Cycle
14.D must be scenario-driven and must reject any candidate that only improves a
secondary metric while worsening DB FPS, Step 2 through-pose, or model RTT.

## Candidate Split

Cycle 14.D is split before implementation because the best mechanism is not
known.

| Sub-cycle | Candidate | Required Phase A proof |
|---|---|---|
| 14.D1 | Triton Python BLS compact behavior postprocessing | Production Triton binary includes Python backend or a controlled runtime switch is benchmarkable. |
| 14.D2 | TensorRT-only compact postprocessing / plugin path | A concrete TensorRT graph can reduce behavior wait/server execution, not only response bytes. |
| 14.D3 | Server-side fused behavior output contract | A route exists that preserves exact DB/model parity while reducing model RTT or provider wait. |

If none of the sub-cycles can prove expected impact before code, Cycle 14.D
must remain investigation-only and the roadmap should move to Cycle 15.

## Required Measurements Before Code

| Measurement | Purpose |
|---|---|
| Current Triton backend list | Determine whether Python BLS is possible without a runtime rebuild. |
| Current behavior model RTT/server stats | Prove behavior wait is worth attacking after Cycle 14.B2/14.C3. |
| Current RTMPose tail vs behavior wait | Avoid optimizing behavior if RTMPose remains the dominant unresolved tail. |
| Current output bytes and decode/NMS wall | Confirm the Top-K path has not shifted back into a byte/decode bottleneck. |

## Acceptance Criteria

No 14.D sub-cycle can be accepted until it has:

| Gate | Requirement |
|---|---|
| Production benchmark | Completed Linux RTX 5090 `combined.mp4` benchmark. |
| Correctness | DB parity and model-agreement parity against accepted batch `16`. |
| Performance | Measurable DB FPS improvement without RTT/p95 regression. |
| Root-cause match | Evidence that the implemented sub-cycle addressed the measured bottleneck. |
| Rollback | Production restored or intentionally kept with documented env/model route. |

## Next Step

Run Phase A measurements for 14.D1/14.D2/14.D3. Do not implement BLS, plugins,
or fused output contracts until the measurement table identifies a single
highest-ROI candidate.
