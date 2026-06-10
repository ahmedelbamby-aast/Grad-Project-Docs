# Cycle 015 Throughput Remediation Investigation

**Created:** 2026-06-10
**Last updated:** 2026-06-10
**Status:** `highest_priority_remediation_planned`
**Streaming compatibility:** `stream-safe-with-config`

## Source-of-truth references

| Kind | Reference |
|---|---|
| Doc | `docs/xai_anomaly/cycle_015_0_current_run_bottleneck_report.md` |
| Doc | `docs/xai_anomaly/cycle_015_0_investigation.md` |
| Doc | `specs/015-xai-anomaly-score/spec.md` |
| Doc | `specs/015-xai-anomaly-score/plan.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |
| Doc | `specs/015-xai-anomaly-score/contracts/benchmark-evidence-contract.md` |
| Doc | `docs/production_inference_benchmark.md` |
| Doc | `docs/inference_parallelization_plan.md` |
| Doc | `docs/cycle_9_and_10_improvements_todo.md` |
| File | `tools/prod/prod_watch_benchmark_metrics.sh` |
| File | `tools/prod/prod_run_xai_cycle015_0.sh` |
| File | `backend/apps/video_analysis/tasks.py` |
| File | `backend/apps/pipeline/services/triton_client.py` |
| File | `backend/apps/telemetry/celery_integration.py` |
| File | `backend/apps/telemetry/writer.py` |

## Target

The current mandatory target is `>=15 FPS` DB-completed end-to-end throughput
on canonical production `combined.mp4` at frame stride `1`. The practical batch
envelope is `32` frames completing their full authoritative cycle in `<=2`
seconds. This includes inference, pose/behavior postprocess, PostgreSQL writes,
embeddings and derived records, telemetry/artifacts, reconciliation, and
terminal lifecycle state.

The 2026-06-10 Cycle 015.0 baseline run
`cycle015-0-baseline-final-20260610T184443Z` / job
`c080fac3-b5d0-43db-8d0a-0e037fa92ddd` is the current blocking evidence:
final watcher output showed about `1.773` DB-completed FPS, `2499.409 s`
Step 2 through-pose wall, `1610.740 s` Step 2 frame wall, `888.668 s`
pose tail, `4921.712 s` summed postprocess time, and `2133.013 s` summed
inference time.

## Bottleneck Ranking

| Rank | Bottleneck | Evidence | Initial remediation lane |
|---:|---|---|---|
| 1 | Postprocess wall | `4921.712 s`, `68.753%` of summed stage timing | profiler-backed split, compact outputs, reduce fanout construction cost |
| 2 | Inference/orchestration wall | `2133.013 s`, GPU idle, behavior RTT mean `107.446 ms` | batch/dispatch redesign, reduce Python/gRPC boundary cost, keep GPU fed |
| 3 | Pose tail | `888.668 s`, `rtmpose_model` `4580` calls | overlap or bounded parallel pose without live window waits |
| 4 | Fanout amplification | `127469` detections/bboxes for `4541` frames | route/threshold diff before any fanout cap or compression |
| 5 | Per-frame scene segmentation | `4541` `yoloe_scene_seg` calls, `25.365 ms` mean RTT | prove required vs profile drift, then cache/move/disable if allowed |
| 6 | Observability lag | live telemetry unavailable until task end | live-flush or watcher-side active stage/RTT collection |
| 7 | Lifecycle convergence | completed rows/summaries while `status=embedding` | terminal-state transition/reconciler fix |

## Required Implementation Plan

### Phase A: Profiler-backed attribution

- Capture a production or production-equivalent profiler split for
  `postprocess_s`, including output decode, class mapping, NMS/filtering,
  Python object construction, row staging, and fanout loops.
- Extend the production watcher/collector to surface active per-model RTT,
  stage wall, row counts, terminal status, and 32-frame full-cycle envelopes
  before task end. **Implemented 2026-06-10 for active stage/model visibility**:
  `backend/apps/video_analysis/tasks.py` now writes throttled
  `benchmark_live_rollup` stage timing metadata, `backend/apps/telemetry/session.py`
  writes throttled `benchmark_live_model_rollup` per-model RTT aggregates, and
  `tools/prod/prod_watch_benchmark_metrics.sh` prints `Active In-Flight Stage
  Evidence` plus `Active In-Flight Model RTT` tables. Evidence:
  `backend/tests/unit/telemetry/test_telemetry_layer.py` and
  `backend/tests/unit/scripts/test_prod_watch_benchmark_metrics_contract.py`.
- Treat missing benchmark summary, GPU CSV, metrics JSON, agreement JSON, and
  decode JSON as explicit unavailable metrics, not silent omissions.

### Phase B: Critical-path remediation candidates

- Reduce postprocess wall first. Candidate families: compact server outputs,
  vectorized decode/filtering, bounded row staging, and fanout compression that
  preserves source-output correctness.
- Reduce inference/orchestration wall second. Candidate families: fewer Python
  dispatch boundaries, larger safe true batches only when they do not violate
  live latency, and route-level removal of accidental per-frame work.
- Bound pose tail third. Candidate families: overlap pose with frame inference,
  bounded per-frame pose chunking, or parallel pose workers with explicit live
  queue limits.
- Fix lifecycle convergence in parallel because an ambiguous terminal state
  invalidates DB-completed throughput even when rows are present.

### Phase C: Route/profile diff before behavior changes

- Diff the Cycle 015.0 runtime route snapshot against the last accepted
  throughput profile before disabling `yoloe_scene_seg`, changing thresholds,
  or suppressing fanout.
- Any fanout or route change must keep source detection/track/pose/scene output
  parity gates explicit. If parity changes are intended, they require a
  governed scientific acceptance path, not a performance-only decision.

### Phase D: Acceptance benchmark

- Run native Linux RTX 5090, `combined.mp4`, frame stride `1`.
- Record DB-completed FPS and the `32 frames <=2s` full-cycle envelope.
- Record phase wall, per-model RTT/call rate, GPU/VRAM/power, CPU/RSS,
  PostgreSQL/Redis, embeddings/derived rows, lifecycle state, correctness,
  identity, rollback, figures, manifest, and benchmark-ledger row.
- Accept no remediation as complete until `>=15 FPS` and `32 frames <=2s`
  are both met without correctness, streaming, rollback, or evidence regression.

## Current Decision

Cycle 015 downstream additive XAI/anomaly work remains blocked from production
acceptance on the critical path. The next implementation cycle must attack the
measured throughput bottlenecks above or prove, with production evidence, that
it does not widen them.
