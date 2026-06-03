# Cycle 14.D Server-Side Compact Postprocessing Results

**Last updated:** 2026-06-03

**Status:** PHASE A COMPLETE / NO IMPLEMENTATION SELECTED. This is not an
acceptance, rejection, skip, or closure decision for an implemented
optimization. It is a measurement gate result: none of the 14.D sub-candidates
has enough production evidence to justify code before Cycle 15 starts.

## Source-of-Truth References

| Kind | Reference | Why it matters |
|---|---|---|
| Investigation | `docs/cycle_14d_server_side_compact_postproc_investigation.md` | Defines the D1/D2/D3 split before code. |
| Accepted comparison baseline | `docs/cycle_14b_rtmpose_scenario_results.md` | Batch `16` remains the accepted production profile. |
| Batch-size follow-up | `docs/cycle_14c_pose_batch_size_matrix_results.md` | Proves batch `8` and batch `32` are not accepted. |
| Batch-32 repair | `docs/cycle_14c3_batch32_parallel_chunks_results.md` | Proves internal provider chunk parallelism is not accepted. |
| Phase A wrapper | `tools/prod/prod_run_cycle14d_phase_a_measurements.sh` | Reproducible production measurement wrapper for this result. |
| Decode probe | `tools/prod/prod_probe_behavior_decode_cost.py` | Measures behavior output parse and Python decode/NMS cost. |
| RTT probe | `tools/prod/probe_rtt_decompose_topk.py` | Measures behavior ensemble RTT, server work, and output shape. |
| Benchmark collector | `tools/prod/prod_collect_benchmark_metrics.py` | Defines the benchmark evidence schema used by accepted cycles. |
| Plan | `docs/inference_parallelization_plan.md` | Owns the cycle order and production-decision rule. |
| Execution log | `docs/crop_frame_optimization_execution.md` | Tracks current optimization status and next cycle. |

## Production Evidence

| Field | Value |
|---|---|
| Evidence directory | `/home/bamby/grad_project/backend/logs/cycle14d-phase-a-20260603T175051Z` |
| Accepted baseline replay used by probes | `cycle14b-cross-frame-batch16-r2-20260603T150000Z` |
| Wrapper | `tools/prod/prod_run_cycle14d_phase_a_measurements.sh` |
| Deployed SHA for wrapper | `f4b9037` |
| Job submitted | none; measurement-only probes against existing accepted production state |
| Production mutation | none; wrapper did not edit env, restart services, submit jobs, or change routes |

## Runtime Feasibility

| Check | Production measurement | Decision impact |
|---|---:|---|
| Python backend visible in Triton backend log | `False` | 14.D1 Python BLS is blocked for code until a controlled Triton runtime switch/rebuild is benchmarked. |
| `behavior_ensemble_gaze_slice_topk` readiness | `READY` | Current accepted behavior route is healthy. |
| Top-K child readiness | `READY` | Measurements ran against the accepted exact-slice + Top-K graph. |
| RTMPose readiness | `READY` | Accepted Cycle 14.B2 pose path remained loaded. |

## Behavior RTT and Server Work

| Metric | Value |
|---|---:|
| Ensemble RTT mean | `63.664 ms` |
| Ensemble RTT p95 | `69.210 ms` |
| Ensemble transport + server mean | `59.338 ms` |
| Ensemble deserialize mean | `2.331 ms` |
| Ensemble output bytes mean | `326400.0` |
| `behavior_ensemble_gaze_slice_topk` server avg | `29.976 ms/exec` |
| `gaze_horizontal_model` server avg | `18.957 ms/exec` |
| `gaze_vertical_model` server avg | `16.389 ms/exec` |
| `gaze_depth_model` server avg | `16.253 ms/exec` |
| `posture_model` server avg | `16.352 ms/exec` |

## Decode and Output Cost

| Metric | Value |
|---|---:|
| Probe batches | `80` |
| Probe crops | `1360` |
| RTT with parse mean | `64.033 ms` |
| Infer wait mean | `61.445 ms` |
| `as_numpy` mean | `0.118 ms` |
| Decode total mean | `3.357 ms/batch` |
| Decode total p95 | `4.868 ms/batch` |
| Decode cost per crop | `0.197497 ms/crop` |
| Output bytes per crop | `19200.0` |
| Estimated compact bytes per crop | `14.2` |
| Estimated output-byte reduction | `99.926 %` |

Per-model decode cost stayed small after the accepted Top-K route:

| Model | Decode mean | Decode p95 | Boxes mean/batch |
|---|---:|---:|---:|
| `posture_detection` | `1.562 ms/batch` | `2.503 ms/batch` | `6.862` |
| `gaze_horizontal` | `0.518 ms/batch` | `0.755 ms/batch` | `0.000` |
| `gaze_vertical` | `0.743 ms/batch` | `1.216 ms/batch` | `2.663` |
| `gaze_depth` | `0.534 ms/batch` | `0.814 ms/batch` | `0.500` |

## Sub-Cycle Triage

| Sub-cycle | Phase A result | Reason |
|---|---|---|
| 14.D1 Python BLS | BLOCKED FOR CODE | The pinned production Triton runtime log did not expose a Python backend. A BLS implementation would first require a controlled Triton runtime switch/rebuild and its own benchmark gate. |
| 14.D2 TensorRT/plugin compacting | NO IMPLEMENTATION SELECTED | Output bytes can drop sharply, but the measured client parse/decode cost is only `3.357 ms/batch`; Phase A did not prove a concrete TensorRT/plugin graph that reduces server execution or wait. |
| 14.D3 fused behavior output contract | NO IMPLEMENTATION SELECTED | The current Top-K route already returns compact `[N, C, 100]` tensors. A new fused contract would need to reduce RTT/wait, not just bytes; Phase A did not prove that path. |

## Decision

Cycle 14.D remains unimplemented. The evidence says compact output bytes are
not the next dominant measured lever after exact-slice + Top-K and accepted
RTMPose batch `16`.

This decision does **not** reject BLS, plugins, or fused output permanently. It
only prevents speculative implementation now. A future 14.D sub-cycle may be
reopened if one of these is true:

| Reopen condition | Required proof before code |
|---|---|
| Python BLS | Production Triton runtime exposes Python backend, or a controlled runtime switch/rebuild is benchmarked as stable. |
| TensorRT/plugin compacting | A concrete graph/plugin design estimates a server-wait or execution reduction, not only response-byte reduction. |
| Fused behavior output | A parser/route design shows RTT or provider-wait reduction while preserving exact DB/model parity. |

## Next Cycle

Start Cycle 15 as the next sorted investigation. It must compare CUDA shared
memory and video sharding architecture options before code, because Cycle 14.D
did not select an implementable compact-output candidate.
