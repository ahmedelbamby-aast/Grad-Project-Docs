# Cycle 015 Throughput Remediation Investigation

**Created:** 2026-06-10
**Last updated:** 2026-06-11
**Status:** `cycle_015_17_not_accepted`
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
| File | `tools/prod/prod_start_celery_workers.sh` |
| File | `backend/apps/video_analysis/tasks.py` |
| File | `backend/apps/video_analysis/persistence_pipeline.py` |
| File | `backend/config/celery.py` |
| File | `backend/tests/unit/video_analysis/test_async_persistence_seam.py` |
| File | `backend/tests/unit/video_analysis/test_persistence_pipeline.py` |
| File | `tools/prod/prod_generate_xai_cycle015_17_figures.py` |
| Doc | `docs/xai_anomaly/cycle_015_17_figure_plan.md` |
| Doc | `docs/xai_anomaly/cycle_015_17_figure_implementation.md` |
| Doc | `docs/xai_anomaly/cycle_015_17_results.md` |
| Artifact | `docs/figures/benchmark_artifacts/cycle015-17-prod-r2-20260611/figures/MANIFEST.json` |
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
- **Extended 2026-06-10 for postprocess attribution**: the same live rollup now
  carries `postprocess_phase_timings_ms` and per-frame means for model output
  decode, crop payload build, behavior inference wait, behavior response decode,
  person cadence/cache, tracking, scene output decode/callback, and LPM. The
  watcher prints each nested phase or the explicit unavailable reason
  `job.metadata.benchmark_live_rollup.postprocess_phase_timings_ms_absent`.
  Evidence: `backend/tests/unit/video_analysis/test_cycle20_post_stage_timeline.py`
  and `backend/tests/unit/scripts/test_prod_watch_benchmark_metrics_contract.py`.
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

## Verdict On The Write-Behind / Producer-Consumer Hypothesis (2026-06-11)

**The owner's understanding of the flush behavior is CORRECT.** The mid-run
watcher evidence proves it: at `2650/4541` processed frames the authoritative
tables held `0` frame, detection, bbox, embedding, and track rows. Data
("anything flying through the system" — frames, detections, boxes, embeddings,
telemetry) reaches PostgreSQL only in post-frame-loop stages
(`persisting_aggregated_boxes` and later), and telemetry bulk-inserts at Celery
session end (RC-4).

**But the plain conclusion ("add write-behind, get throughput") is NOT
supported by our own measurements — twice:**

1. **The serial DB tail is small in this run.** Post-Step-2 to run-complete
   wall was `61.328 s` of a `2560.737 s` run (~2.4%). Eliminating ALL of it
   moves `1.773` DB-completed FPS by a few percent. The measured giants are
   in-loop host-side postprocess (`4921.712 s`, 68.753%), inference
   orchestration with an idle GPU (`2133.013 s`, GPU 0-6% util), and the pose
   tail (`888.668 s`).
2. **The in-process write-behind was already built and benchmarked — Cycle
   20.E — and NOT ACCEPTED:** it persisted `4541/4541` overlap packets with `0`
   failures, yet DB FPS regressed `5.86%`, total elapsed regressed `6.22%`,
   and Step 2 through-pose wall regressed `12.29%`. Overlapping persistence on
   the SAME process/CPU as the frame loop steals exactly the resource the loop
   is starved of.

**Accepted better decision — Cross-Process Staged Persistence & Postprocess
Offload (Cycle 015.17).** The producer-consumer idea is right; the boundary was
wrong. The recommended fan-out becomes:

```text
Inference worker (frame loop = decode -> dispatch -> minimal ordered tracking)
      |
      v  compact frame packets (bounded Redis Stream lanes, persistq:{job}:{lane})
      +--> db_rows consumer     (separate Celery worker process: row fanout + bulk writes)
      +--> embedding consumer   (separate process: existing embedding stage)
      +--> analytics consumer   (later slice: kinematics/BSIL/XAI derived work)
      +--> artifacts consumer   (later slice: JSON/labels/render artifacts)
```

What changes vs 20.E: consumers run in **separate Celery worker processes**
(other CPU cores), so the frame loop sheds its postprocess fanout instead of
sharing its core with a background thread. This is also exactly the
"independent work" precondition Cycle 21 (worker scaling) has been blocked on.
What stays from the evidence: ordered, stateful tracking stays on the producer
path (sharding/identity lessons from 15.B1); behavior in-flight stays bounded
(12.B lesson); streams are MAXLEN-bounded with drain + serial reconcile before
terminal state (fail-closed; RC-8 lifecycle convergence is a gate).

Implemented 2026-06-11 (default-off): the tracked-frame callbacks now publish
compact revisioned packets during Step 2. The dedicated
`pipeline.offline.persistence` Celery worker builds `StudentTrack`,
`Detection`, `BoundingBox`, and label-file fanout outside the inference
process. The ordered tracking assignment remains in the producer. The
consumer acknowledges and deletes terminal Redis Stream entries, skips stale
revisions, and uses explicit long-job task limits. Step 3 publishes any final
signature changes, drains the lane, compares exact persisted box signatures,
and serially repairs missing or stale frames before later stages proceed.
The embedding control slice dispatches the existing idempotent embedding
stage onto the dedicated persistence worker, with standard-task fallback
after consumer apply failures.

Figure planning and implementation evidence are separated in
`docs/xai_anomaly/cycle_015_17_figure_plan.md` and
`docs/xai_anomaly/cycle_015_17_figure_implementation.md`.

The native RTX 5090 stride-1 production decision is **NOT ACCEPTED**. The
corrected r2 candidate drained all `8185` `db_rows` packets with zero failures
and zero serial reconciliation, but DB-completed FPS regressed
`1.655863 -> 1.643340`, Step 2 frame wall regressed
`1626.406689 s -> 1629.966177 s`, and student-track rows diverged
`138 -> 149`. The runner also rolled back before the downstream embedding
stage reached its actual terminal state, contaminating candidate GPU-tail and
rollback timing authority. Full evidence and figures are recorded in
`docs/xai_anomaly/cycle_015_17_results.md`.

## Current Decision

Cycle 015 downstream additive XAI/anomaly work remains blocked from production
acceptance on the critical path. Cycle 015.17 remains default-off and is not an
accepted architecture lane. The next implementation cycle must fix terminal
lifecycle coordination and attack the measured Step 2 bottlenecks, or prove
with production evidence that it does not widen them. The target ladder remains
`>=15` FPS DB-completed first, then the semi-realtime `25-30` FPS last-stage
target.
