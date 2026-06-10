# Cycle 015.0 Current Production Run Bottleneck Report

**Created:** 2026-06-10
**Last updated:** 2026-06-10
**Run observed:** `cycle015-0-baseline-final-20260610T184443Z`
**Job:** `c080fac3-b5d0-43db-8d0a-0e037fa92ddd`
**Status at final watcher collection:** DB/run-summary complete; persisted
`VideoAnalysisJob.status` still `embedding`
**Scope:** live read-only bottleneck diagnosis for the current production
Cycle 015.0 baseline run on `combined.mp4`.

## Evidence References

| Kind | Reference |
|---|---|
| Doc | `docs/xai_anomaly/cycle_015_0_investigation.md` |
| Doc | `specs/015-xai-anomaly-score/plan.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |
| Doc | `docs/production_inference_benchmark.md` |
| Doc | `docs/inference_parallelization_plan.md` |
| Doc | `docs/cycle_9_and_10_improvements_todo.md` |
| File | `tools/prod/prod_run_xai_cycle015_0.sh` |
| File | `tools/prod/prod_check_subjective_progress.sh` |
| File | `tools/prod/prod_watch_benchmark_metrics.sh` |
| File | `backend/apps/video_analysis/tasks.py` |
| File | `backend/apps/telemetry/celery_integration.py` |
| File | `backend/apps/telemetry/writer.py` |
| File | `backend/apps/pipeline/services/triton_client.py` |

## Mid-Run Production Snapshot

Read-only production collection was performed through `tools/prod/prod-ssh.ps1`
against `prod-grad` on 2026-06-10.

The latest active production job at collection time was not the stale
`all_merged.mp4` job returned by the subjective-progress script default. The
actual latest `combined.mp4` run was:

| Field | Value |
|---|---:|
| Replay key | `cycle015-0-baseline-final-20260610T184443Z` |
| Job ID | `c080fac3-b5d0-43db-8d0a-0e037fa92ddd` |
| Runtime commit | `ad709364` |
| Branch | `015-xai-anomaly-score` |
| Pipeline mode | `crop_frame` |
| Processed frames at first DB snapshot | `2525/4541` |
| Frame-basis progress at first DB snapshot | `55.604%` |
| DB `progress_percent` at first DB snapshot | `41.7%` |
| Elapsed at first DB snapshot | `804.538 s` from creation |
| Overall FPS at first DB snapshot | `3.138445` |

The later 20-second watch window reported:

| Field | Value |
|---|---:|
| Processed frames | `2625/4541` |
| Frame-basis progress | `57.81%` |
| Elapsed | `0:14:09` |
| Overall FPS | `3.093` |
| Window delta | `50` frames in `20` s |
| Window FPS | `2.500` |
| Window ETA from frame loop only | `0:12:46` |

The production benchmark watcher reported at `2650/4541` frames:

| Signal | Value |
|---|---:|
| `processed_frames` | `2650` |
| `db_frame_rows` | `0` |
| `detection_rows` | `0` |
| `bbox_rows` | `0` |
| `embedding_rows` | `0` |
| `student_tracks` | `0` |
| `model_call_events` | `4` |
| `frame_lifecycle_events` | `0` |
| `run_session_summaries` | `1` |
| `TelemetrySession` visible while running | `none` |
| Benchmark summary JSON | unavailable while running |
| Metrics JSON | unavailable while running |
| Inference audit path | unavailable while running |

Five live GPU samples during the same run were:

| Sample | GPU util % | VRAM MiB | Power W |
|---:|---:|---:|---:|
| 1 | `2` | `16131/32607` | `95.35` |
| 2 | `0` | `16131/32607` | `82.15` |
| 3 | `6` | `16131/32607` | `83.57` |
| 4 | `2` | `16131/32607` | `92.77` |
| 5 | `2` | `16131/32607` | `79.50` |

Operational process checks:

| Check | Result |
|---|---:|
| Celery worker processes | `23` |
| Triton server processes | `1` |
| Triton recent errors | none found; only TensorRT logger warnings |
| Celery recent error/timeout/OOM lines | none found in tailed logs |

## Final Watcher Snapshot

The run later reached `4541/4541` frames and emitted completed run summaries,
but the persisted job status still reported `embedding`. This is a separate
lifecycle convergence defect and must be treated as part of the bottleneck
evidence because DB-completed throughput requires terminal lifecycle state.

| Field | Value |
|---|---:|
| Final replay key | `cycle015-0-baseline-final-20260610T184443Z` |
| Final job ID | `c080fac3-b5d0-43db-8d0a-0e037fa92ddd` |
| Final watcher status | `embedding` |
| `processed_frames` | `4541/4541` |
| `progress_percent` | `100` |
| Elapsed from processing | `00:42:41` |
| Elapsed from creation | `00:42:42` |
| Final DB-completed FPS | `1.773` |
| GPU util at final watcher | `0%` |
| VRAM at final watcher | `16131/32607 MiB` |
| Power at final watcher | `12.89 W` |
| Frame rows | `4541` |
| Detection rows | `127469` |
| Bounding-box rows | `127469` |
| Embedding rows | `65000` at watcher collection |
| Pose kinematics rows | `73267` |
| Student tracks | `56` |
| Run session summaries | `2` |
| Frame lifecycle events | `0` |
| Inference audit | `/home/bamby/grad_project/backend/data/videos/c080fac3-b5d0-43db-8d0a-0e037fa92ddd/inference_audit.json` |

Final phase and stage timing from the watcher:

| Metric | Value |
|---|---:|
| `step2_frame_wall_s` | `1610.740` |
| `step2_through_pose_s` | `2499.409` |
| `run_complete_wall_s` | `2560.737` |
| Step 2 frame FPS | `2.819` |
| Step 2 through-pose FPS | `1.817` |
| Pose-tail wall | `888.668 s` |
| Pose-tail share of through-pose wall | `35.555%` |
| Post-Step-2 to run-complete wall | `61.328 s` |
| `decode_s` | `0.071` |
| `preprocess_s` | `103.708` |
| `inference_s` | `2133.013` |
| `postprocess_s` | `4921.712` |
| Summed stage timing | `7158.504 s` |
| Postprocess share of summed stage timing | `68.753%` |
| Inference share of summed stage timing | `29.797%` |
| Postprocess equivalent per-frame cost | `1083.839 ms/frame` |
| Inference equivalent per-frame cost | `469.723 ms/frame` |
| Async dispatch calls | `0` |

Final per-model RTT/call-rate evidence:

| Model | Calls | Mean RTT ms | P95 ms | P99 ms |
|---|---:|---:|---:|---:|
| `rtmpose_model` | `4580` | `42.922` | `45.639` | `82.767` |
| `yoloe_scene_seg` | `4541` | `25.365` | `46.130` | unavailable |
| `behavior_ensemble` | `3597` | `107.446` | `171.281` | `179.643` |
| `person_detector` | `910` | `17.181` | unavailable | unavailable |
| `posture_model` | `1` | unavailable | unavailable | unavailable |
| `gaze_horizontal_model` | `1` | unavailable | unavailable | unavailable |
| `gaze_vertical_model` | `1` | unavailable | unavailable | unavailable |
| `gaze_depth_model` | `1` | unavailable | unavailable | unavailable |

Final bounding-box fanout by model:

| Model | Rows |
|---|---:|
| `person_detection` | `73881` |
| `sitting_standing` | `33012` |
| `attention_tracking` | `11775` |
| `hand_raising` | `8801` |

The watcher did not resolve a benchmark summary JSON, GPU CSV, metrics JSON,
agreement JSON, or decode JSON for this replay key at collection time. The run
directory contained wrapper/preflight/runtime-guard artifacts, so any final
decision package must either attach the missing artifacts or record a specific
unavailable reason for each missing metric.

## Interpretation

The user-observed `19.4% after 6 minutes` already implied roughly
`2.45` frame-loop FPS if interpreted as frame-basis progress over `4541`
frames. The later production snapshot confirmed the same bottleneck class:
the current frame loop was moving between `2.5` and `3.1` FPS. The completed
watcher view was worse on authoritative DB-completed throughput: about `1.773`
FPS. This is far below the current required target of `>=15 FPS`
DB-completed end-to-end throughput and the practical `32 frames <=2s` full-cycle
envelope now recorded in `AGENTS.md`, `.specify/memory/constitution.md`, and
the Cycle 015 spec artifacts.

This is not a GPU saturation problem. The GPU had loaded engines and about
`16.1 GiB` VRAM in use, but utilization stayed between `0%` and `6%` during
the observed samples. A saturated RTX 5090 would show high sustained GPU
utilization and higher board power. This run instead shows the classic
application/orchestration bottleneck already described in the benchmark
history: CPU-side frame/crop orchestration, Triton request/response handling,
postprocessing, and non-overlapped post stages dominate while the GPU waits.

The final decomposition sharpens that diagnosis: summed `postprocess_s` is the
largest measured cost, followed by `inference_s`, with a large pose tail after
the frame loop. The exact code-level subcause inside postprocessing still
requires profiling, but it is no longer acceptable to treat raw model RTT alone
as the main bottleneck.

## Root Causes

### RC-1: Frame-loop throughput is below both SLA and accepted baseline

**Finding:** `IMPLEMENTED / MEASURED`. The live run was processing only
`2.5` FPS in the 20-second watch window and about `3.1` FPS on elapsed basis.

**Cause:** The current Cycle 015.0 baseline is spending most wall time in the
existing crop-frame inference path before any authoritative persistence rows
are written. The watcher showed `2650` processed frames but `0` frame,
detection, bbox, embedding, and track rows. In `backend/apps/video_analysis/tasks.py`,
the crop-frame path updates `processed_frames` during the frame loop, while
the later progress stages move to `persisting_aggregated_boxes`, rendering,
and completion after the frame loop. Current progress therefore measures
front-half frame-loop progress, not completed evidence.

**Impact:** The final wall time will be worse than the observed frame-loop
ETA because persistence, embedding, render, metrics collection, post-run
preflight, rollback, and figure generation remain after frame inference.

### RC-2: GPU is idle because host-side orchestration is the limiter

**Finding:** `IMPLEMENTED / MEASURED`. Live GPU samples were `0-6%` util while
the job was actively processing.

**Cause:** The current runtime is not feeding enough GPU work to Triton. This
matches prior benchmark evidence in `docs/production_inference_benchmark.md`
and `docs/inference_parallelization_plan.md`: crop-frame throughput has been
limited by CPU/Python/gRPC request orchestration, behavior/pose scheduling,
postprocessing, and stage sequencing rather than raw TensorRT compute.

**Impact:** Adding more model engines or relying on GPU memory occupancy will
not fix this run. The bottleneck sits before or around Triton dispatch and
response handling.

### RC-3: Persistence and embedding overlap are disabled, so final wall has a serial tail

**Finding:** `IMPLEMENTED / MEASURED`. The production watcher reported:
`OFFLINE_STREAM_POST_STAGES=False`, `OFFLINE_STREAM_POST_STAGE_TIMELINE=False`,
and no Cycle 20 overlap timestamps for the current job.

**Cause:** Cycle 20.E final-stable overlap was previously measured and rejected,
so the production default keeps streaming post-stage overlap disabled. That is
correct for governance, but it means this run still waits for post-frame-loop
persistence, embedding, render, and finalization instead of overlapping them
with frame inference.

**Impact:** The current progress percentage underestimates total remaining work.
At `2650/4541`, the run had not started authoritative DB row persistence.

### RC-4: Live per-model RTT telemetry is unavailable during the active task

**Finding:** `PARTIAL / MEASURED`. The watcher found no active
`TelemetrySession` rows for the running job and only four `ModelCallEvent`
rows at `2650` processed frames.

**Cause:** `backend/apps/telemetry/celery_integration.py` starts and ends
`TelemetrySession` at Celery task boundaries, and `backend/apps/telemetry/writer.py`
bulk-inserts telemetry on session end. That design preserves final telemetry,
but during this active task it leaves the live watcher without model RTT
percentiles. The current run therefore cannot attribute the live frame-loop
slowdown to a specific model from persisted telemetry until the task ends or
the benchmark collector emits final artifacts.

**Impact:** The current bottleneck can be classified as host/orchestration
and non-overlapped stage work, but per-model live ranking is unavailable.
This is an observability bottleneck, not necessarily the runtime bottleneck.

### RC-5: The baseline run is on the Cycle 015 feature branch, not the last accepted throughput branch

**Finding:** `IMPLEMENTED / MEASURED`. Production reported branch
`015-xai-anomaly-score` at commit `ad709364`.

**Cause:** Cycle 015.0 intentionally runs a baseline/candidate pair on the
same reviewed SHA to prove XAI runtime truth and BSIL activation. The active
run is therefore a Cycle 015.0 baseline evidence run, not an accepted throughput
optimization rerun. The missing `docs/xai_anomaly/cycle_015_0_results.md`
also means there is not yet a completed Cycle 015.0 decision package.

**Impact:** Do not compare this run as an accepted optimization result. Treat it
as active evidence collection that is currently exposing a serious throughput
regression or configuration gap relative to the accepted profile.

### RC-6: Postprocess wall is the dominant measured stage cost

**Finding:** `IMPLEMENTED / MEASURED`. Final watcher evidence reports
`postprocess_s=4921.712`, which is `68.753%` of summed stage timing and an
equivalent `1083.839 ms/frame`.

**Cause:** The measured stage is dominated by client-side postprocess,
aggregation, decoding, row construction, and fanout handling around large
detection/bounding-box volumes. The exact function-level split still needs a
profiler-backed investigation; this report does not claim a single code symbol
as the root cause without that evidence.

**Impact:** Optimizing only GPU compute, direct Triton RTT, or request count
cannot meet the `>=15 FPS` target while postprocess remains at this scale.

### RC-7: Pose-tail wall is a major post-frame-loop bottleneck

**Finding:** `IMPLEMENTED / MEASURED`. The watcher reports
`step2_frame_wall_s=1610.740`, `step2_through_pose_s=2499.409`, and a
`888.668 s` pose tail, equal to `35.555%` of the through-pose wall.

**Cause:** Pose work is not fully absorbed into the frame loop and remains a
large serialized or insufficiently overlapped tail. The `rtmpose_model` call
count (`4580`) and mean RTT (`42.922 ms`) make this tail large enough to block
the throughput target even if behavior RTT improves.

**Impact:** The remediation plan must either overlap, batch, or bound pose-tail
work without violating streaming-source constraints and correctness gates.

### RC-8: Lifecycle/status convergence failed at completion

**Finding:** `IMPLEMENTED / MEASURED`. The final DB watcher showed completed
frames, completed run summaries, and `completed_at`, while
`VideoAnalysisJob.status` still reported `embedding`.

**Cause:** The terminal state transition and/or reconciler is not converging
cleanly for this run. The earlier mid-run observation also showed progress
percent and processed-frame basis diverging, including a transient processed
frame count regression during the status transition.

**Impact:** DB-completed throughput cannot be considered healthy while terminal
state is ambiguous. The benchmark collector must record terminal lifecycle
state explicitly and the implementation must fix the state convergence defect.

### RC-9: Per-frame `yoloe_scene_seg` calls add a measurable critical-path load

**Finding:** `PARTIAL / MEASURED`. The run made `4541` `yoloe_scene_seg` calls
with `25.365 ms` mean RTT.

**Cause:** The final watcher proves the model was active for every frame. It
does not by itself prove whether that route is intended for this Cycle 015.0
baseline or a profile/configuration mismatch, so this remains a route-diff
investigation item.

**Impact:** The remediation pass must diff the route/config snapshot against
the accepted throughput profile and decide whether this per-frame load is
required, can be cached, can be moved off the critical path, or should be
disabled for the baseline.

### RC-10: Detection fanout amplifies postprocess, embedding, and DB work

**Finding:** `IMPLEMENTED / MEASURED`. The run persisted `127469` detections
and bounding boxes for `4541` frames, with `73881` `person_detection` rows and
`33012` `sitting_standing` rows.

**Cause:** High row fanout increases postprocess work, embedding volume,
derived-row volume, and PostgreSQL write pressure. The exact reason for this
fanout must be compared against route thresholds, model outputs, and the last
accepted profile before changing thresholds.

**Impact:** Any 15 FPS plan must cap or compress fanout safely, preserve
source-output correctness, and avoid unsupported threshold changes that would
invalidate scientific comparisons.

## Ruled-Out Causes From This Snapshot

| Suspect | Current evidence | Status |
|---|---|---|
| Duplicate Celery worker storm | `23` Celery worker processes, matching the documented expected parent+child count | Not supported |
| Triton crash or missing process | exactly `1` Triton server process | Not supported |
| Triton hard error / OOM | tailed Triton log showed only TensorRT logger warnings | Not supported |
| Celery timeout/OOM in current tail | tailed Celery logs showed no error/timeout/OOM lines | Not supported |
| PostgreSQL persistence already slow during frame loop | authoritative frame/detection/bbox rows were still `0`, so persistence had not begun during the mid-run snapshot | Not the mid-run frame-loop limiter |

## Immediate Action Items

1. Keep the current run classified as `staged_awaiting_production_evidence`
   until it either completes or fails. Do not record an acceptance decision from
   live progress alone.
2. Attach or explain missing benchmark summary JSON, GPU CSV, metrics JSON,
   agreement JSON, decode JSON, and final preflight artifacts before writing
   `docs/xai_anomaly/cycle_015_0_results.md`.
3. Treat `1.773` DB-completed FPS, `2499.409 s` through-pose wall,
   `4921.712 s` summed postprocess time, and `2133.013 s` summed inference
   time as the current remediation baseline.
4. Diff the active `.env` and route
   snapshot against the last accepted profile. Focus first on profile flags
   that change crop-frame batching, pose tail, behavior route, telemetry
   flushing, scene segmentation, fanout thresholds, and post-stage sequencing.
5. Add or enable live-flush model-call telemetry for long production
   benchmarks, or extend the watcher to read in-memory/progress artifacts,
   because end-of-task telemetry is too late for active bottleneck isolation.
6. Fix lifecycle/status convergence where completed run evidence coexists with
   persisted `status=embedding`.
7. Do not try to fix this with worker-count scaling first. The snapshot shows
   one active video loop and an idle GPU; extra workers do not create more
   independent work unless the pipeline is redesigned to expose it.

## Decision Boundary

This report identifies the current bottleneck class and immediate causes from
live production evidence. It is not a Cycle 015.0 benchmark decision. The cycle
still requires completed baseline/candidate production runs, final metrics,
figures, rollback proof, benchmark-ledger entry, and
`docs/xai_anomaly/cycle_015_0_results.md`.
