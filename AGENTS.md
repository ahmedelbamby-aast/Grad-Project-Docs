# Agents Execution Guide

**Last updated:** 2026-06-02

## Purpose
This file defines how agents should execute tests quickly and safely in this repository.

## Database Authority
- PostgreSQL is the only authoritative relational database for this repository.
- SQLite is not an accepted runtime, test, migration, benchmark, or acceptance backend.
- Agents must not switch tests or scripts to SQLite for convenience, speed, or local fallback.
- Any change touching persistence, migrations, ORM queries, constraints, transactions, or evidence storage must be validated against PostgreSQL semantics.

### Production PostgreSQL Test-Role Prerequisite
- Full backend pytest suites require a PostgreSQL role that can create test databases (`CREATEDB`).
- Current production app role (`exam_user`) may not have enough privileges to create roles or databases.
- If tests fail with `permission denied to create role` or `permission denied to create database`, agents must not attempt SQLite fallback.
- Required DBA/bootstrap action (performed by a privileged PostgreSQL role):
  - Create a dedicated CI/test role (example: `exam_ci_user`) with `LOGIN` + `CREATEDB`.
  - Grant ownership/privileges for runtime DB access as needed.
  - Update environment keys consistently (local/prod CI shells) to use the same PostgreSQL host/port and test role for pytest.

## CI Credential Handling
- Authenticated GitHub CI inspection uses `GITHUB_TOKEN` loaded from local ignored file `.local/secrets/github.env`.
- The credential file MUST remain untracked and MUST contain a rotated token, never a token exposed in chat, logs, evidence, or Markdown.
- Agents MUST NOT write raw CI tokens into tracked files, command transcripts, or production evidence packages.

<!-- SPECKIT START -->
## Active Spec Kit Plan
- Active feature plan: [specs/011-bsil-semantic-runtime/plan.md](specs/011-bsil-semantic-runtime/plan.md)
<!-- SPECKIT END -->

## ⭐ Highest-Priority Plan — Inference Pipeline Parallelization
- **This remains the top-priority runtime/performance plan, but the repo-side
  implementation is now signed complete as of 2026-05-31.**
- Plan: [docs/inference_parallelization_plan.md](docs/inference_parallelization_plan.md)
- Latest bottleneck investigation:
  [docs/crop_frame_rtx5090_bottleneck_investigation.md](docs/crop_frame_rtx5090_bottleneck_investigation.md).
  Production job `80027072-a9d4-4be7-9099-4354acd1170b` eventually completed
  `4541/4541` frames, but was falsely marked stale mid-run because progress
  writes used `QuerySet.update()` without refreshing `updated_at`. The real
  crop-frame throughput limiter is the Python/gRPC boundary around many
  `640x640` person-crop behavior/gaze requests, not decode, crop slicing, DB
  persistence, Triton readiness, or GPU OOM.
- Optimization execution started 2026-06-01: code now contains a guarded
  `TRITON_CROP_BEHAVIOR_INPUT_SIZE` path plus
  `tools/prod/prod_enable_roi_crop_behavior.sh`, which rebuilds posture/gaze
  TensorRT engines and matching Triton configs before enabling smaller
  crop-frame behavior tensors. Production benchmark
  `roi320-running-crop-frame-20260601T012133` (job
  `77650001-3c4b-4b0a-94aa-b4eb899b90df`) completed `4541/4541` frames on
  `/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4`
  with no stale reconciler error and improved throughput/RTT/traffic, but
  average GPU utilization regressed to `3.95%` (peak `34%`). This phase is
  **partial production success only, not final acceptance**.
- **2026-06-01 Cycles 1–5 ACCEPTED** on top of ROI-320: bundled telemetry
  writer fix (TelemetryModelCall now actually persists — was silently dropping
  ~30 k rows / job), `BATCH_QUEUE_MAX_CONCURRENCY` 4→2 (saturation knee from
  prod inflight sweep), `BATCH_QUEUE_MAX_FRAMES` 1→2 (two-frame true_batch
  packing), `OFFLINE_TRIM_EVERY_N_BATCHES=8` (amortize gc/malloc_trim),
  `_build_true_batch_payload` memoization (collapse 4× redundant `b"".join`
  across fan-out behavior models). Replay key
  `cycle2to5-crop-frame-20260601T012045`, job
  `74ec0432-995c-487e-9d77-1048ec109fb1`. **Step-2 FPS 2.09 → 5.14 (+146 %),
  overall FPS 1.308 → 2.644 (+102 %), avg GPU util 3.95 % → 7.55 % (+91 %),
  detection-row parity within 0.01 %.** Evidence:
  `docs/production_inference_benchmark.md` §11,
  `docs/crop_frame_optimization_execution.md`,
  `docs/rtt_root_cause_investigation_77650001.md`.
- **2026-06-01 Cycle 9 NOT ACCEPTED** — Triton ensemble for the 4 behavior
  models. Step 2 wall 852.8 s → 858.1 s (+0.6 %, failed the ≥ 10 % reduction
  acceptance gate). Behavior app calls 14 391 → 3 597 (−75 %), behavior RTT
  143–168 ms → 107.9 ms avg, DB-completed FPS 3.46 → 4.09 (+18.1 %), GPU peak
  36 % → 43 %, all correctness counters unchanged. The flag `TRITON_BEHAVIOR_ENSEMBLE`
  remains deployed but the cycle does not constitute SLA progress because
  the four sub-models still execute identically and still return dense YOLO
  outputs — the reduction was app-side request fragmentation, which was not
  the dominant Step 2 limiter. Replay key
  `cycle9-behavior-ensemble-crop-frame-20260601T180847`, job
  `c1651663-e08a-4e29-9ee3-fd0f09884b98`. Five concrete follow-up options
  (server-side compact postprocessing / BLS, output fusion, child critical-path
  optimization, larger ensemble batches, discipline rule "stop optimizing
  gRPC call count alone") are documented in `docs/cycle_9_results.md`.
- **2026-06-02 Cycle 9b Step-1 measurement complete (not an accepted
  optimization)**: production stats and direct gRPC decomposition in
  `docs/cycle_9b_child_critical_path_results.md` identify
  `gaze_horizontal_model` as the dominant behavior child (`16.058 ms/exec`
  server delta vs. `12.133 ms` posture, `11.759 ms` vertical, `11.909 ms`
  depth). The same model also owns the widest dense output tensor
  (`[84,2100]`). This measurement led to the B.2.b exact server-side slice
  candidate below, which then led to the B.2.c Top-K route. Future Cycle 9b
  work should use the B.2.c accepted-with-caveat profile as the production
  baseline before attempting B.1 compact postprocessing, B.3 Step-2 engine
  variants, or B.4 batch-window changes.
- **2026-06-02 Cycle 9b B.2.b TensorRT output-slice variant NOT ACCEPTED**:
  code added a guarded `GAZE_HORIZONTAL_HEAD_VARIANT=gaze2` path where
  `gaze_horizontal_gaze2_model` gathers legacy horizontal channels
  `[0,1,2,3,8,9]` to produce `[6,2100]`, `behavior_ensemble_gaze2` fans out to
  that child, and the app remaps compact class IDs `0/1` back to legacy DB IDs
  `4/5`. Local validation passed (`160 passed`), and production built/loaded
  the plan at SHA `49932a22bfb429a74075e6952788af63eb007810`, but parity failed
  twice. The final rebuilt-engine proof
  `backend/logs/gaze_horizontal_gaze2_parity_20260601T231503_postrebuild.json`
  reported `max_abs_diff=9.5` against a `1e-6` tolerance. No full benchmark was
  run because the pre-benchmark parity gate failed. Production was rolled back to
  `GAZE_HORIZONTAL_HEAD_VARIANT=coco80`,
  `MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble`, `LPM_ENABLED=0`,
  then Triton/workers were restarted. See
  `docs/cycle_9b_output_fusion_results.md`.
- **2026-06-02 Cycle 9b B.2.b exact server-side slice ACCEPTED**:
  because the prod Triton build has no `python` backend, the accepted low-risk
  dense-byte candidate uses TensorRT only: `gaze_horizontal_model` remains the
  unchanged legacy producer, `gaze_horizontal_slice_model` gathers channels
  `[0,1,2,3,8,9]` from its dense `[84,2100]` output, and
  `behavior_ensemble_gaze_slice` returns `[6,2100]` as `gaze_h_out`.
  `gaze_horizontal_slice_adapter` provides the same 6-channel contract for
  standalone fallback. Production benchmark
  `cycle9b-exactslice-crop-frame-20260601T233211` / job
  `7933c1e5-a970-47a3-81c5-0c9bd01bd332` deployed SHA
  `ca69f02a8ceb214d7ef55cd2ae4b7ec75549c257` and completed
  `4541/4541` frames. Post-benchmark tensor parity
  `backend/logs/gaze_horizontal_slice_parity_20260601T235623_postbench.json`
  passed with `max_abs_diff=0.0`. Step 2 wall improved `858.1 s →
  573.927 s` (`-33.1%`), behavior ensemble RTT improved `107.9 ms →
  91.470 ms`, DB-completed FPS improved `4.09 → 4.307`, average GPU util
  moved `9.36% → 9.595%`, and correctness stayed within noise
  (`attention_tracking` unchanged at `11776`, detections/bboxes `-2`,
  embeddings `-2`, tracks unchanged). Production remains on
  `GAZE_HORIZONTAL_HEAD_VARIANT=slice` with `LPM_ENABLED=0`; rollback is
  `prod_enable_parallel_flow.sh --profile per-frame-signals`.
- **2026-06-02 Cycle 9b B.2.c exact slice + Top-K ACCEPTED WITH CAVEAT**:
  production keeps the accepted exact-slice route and adds FP32 TensorRT Top-K
  adapters (`posture_topk_model`, `gaze_horizontal_slice_topk_model`,
  `gaze_vertical_topk_model`, `gaze_depth_topk_model`) behind
  `TRITON_BEHAVIOR_TOP_K_ENABLED=1`,
  `TRITON_BEHAVIOR_TOP_K_VALUE=100`, and
  `MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble_gaze_slice_topk`.
  FP16 adapters failed decoded parity before benchmarking; FP32 parity passed
  exactly in `backend/logs/behavior_topk_parity_20260602T011830Z_fp32.json`
  (`failed_count=0`, `max_score_diff=0.0`, `max_box_diff=0.0`). Production
  benchmark `cycle9b-topk-crop-frame-20260602T041900` / job
  `be4ba9ee-4786-48e9-8334-28feb237a1fb` deployed SHA
  `9f879affeb4478e63a09276b10a2d64844bcbc44` and completed `4541/4541`
  frames. Versus exact slice, Step 2 frame wall improved
  `573.927 s → 540.399 s` (`-5.84 %`), behavior RTT mean improved
  `91.470 ms → 84.865 ms`, DB-completed FPS improved `4.307 → 4.429`, and
  behavior output traffic fell `~6.85 MB/frame → ~0.33 MB/frame`. Correctness
  stayed within tolerance (`attention_tracking` `11776 → 11781`,
  `person_detection` unchanged at `19162`, tracks unchanged at `53`). Caveat:
  average GPU utilization did not improve (`9.595 % → 9.3 %`) though peak moved
  `45 % → 53 %`; next cycle must target GPU occupancy / server-side execution /
  orchestration rather than response-byte trimming alone. Rollback is
  `prod_enable_gaze_horizontal_slice.sh --input-size 320 --skip-build`, then
  restart Triton and workers.
- **2026-06-02 Cycle 9b B.3 Step 1 REMEASURED against Top-K baseline (not an
  accepted optimization)**: production stats and a controlled direct gRPC RTT
  probe were re-run against the active `behavior_ensemble_gaze_slice_topk`
  graph (job `be4ba9ee-4786-48e9-8334-28feb237a1fb`, SHA `9bc53d86`). Probes:
  `tools/prod/probe_child_stats_topk.py` (`backend/logs/probe_child_stats_topk_20260602T120051.json`)
  and `tools/prod/probe_rtt_decompose_topk.py`
  (`backend/logs/probe_rtt_decompose_topk_20260602T120240.json`). Dominant
  child is still `gaze_horizontal_model` at `18.790 ms/exec` server delta vs
  `16.324 ms` posture, `16.236 ms` vertical, `16.377 ms` depth, but the gap
  shrank from `+33 %` pre-Top-K to `+15 %` (absolute gap `3.93 ms → 2.47 ms`).
  Ensemble end-to-end RTT is `63.59 ms` mean / `30.13 ms` server work — so
  server compute is only `47.4 %` of RTT and orchestration overhead is
  `~8.7 ms/call`. Per-crop GPU compute is constant at `~0.94 ms/crop` across
  bs=17 (probe) and bs=32 (prod aggregate). Per-child Step 2 ceiling is
  therefore `~4 %` Step 2 wall reduction. The remeasurement names two
  candidate Step 2 levers and ranks them in
  `docs/cycle_9b_child_critical_path_remeasure_topk_results.md`: (1) Cycle 11
  input 320 → 256 — hits all four children, projected `−10 % to −13 %`
  Step 2 wall; (2) B.3.b / B.3.d kernel-tactic tuning on `gaze_horizontal_model`
  — bounded at `~4 %` Step 2 wall but lowest risk. Per §E.6 both must be
  prod-benchmarked before one ships. No engine, route, or env flag was
  changed by this remeasurement.
- **2026-06-02 Cycle 11.A behavior input 320→256 NOT ACCEPTED by real
  production benchmark**: production built the 256 behavior engines and matching slice/Top-K
  adapters, then captured candidate outputs with
  `tools/prod/prod_behavior_input_size_parity.py`. Evidence:
  baseline capture `backend/logs/parity_capture_320_20260602T123459.npz`,
  candidate capture `backend/logs/parity_capture_256_20260602T154826.npz`, and
  parity JSON `backend/logs/parity_input_size_256_20260602T154842.json`.
  The candidate showed lower synthetic capture time but failed synthetic
  correctness gates before the full benchmark (`posture_model` class agreement `0.695`,
  `gaze_vertical_model` `0.955`, and large centroid drift on posture / vertical
  / depth). This was treated as a warning, not a final decision authority. Per
  the current operator rule, the real production benchmark was executed before
  deciding:
  `cycle11-input256-realbench-20260602T161641Z-input256`, job
  `822b0da4-fbf2-4186-a5a6-dd066f2eb571`. The benchmark completed
  `4541/4541` frames and proved the performance hypothesis, but failed
  correctness and GPU-utilization gates: Step 2 frame wall `540.399 s →
  391.673 s` (`-27.52 %`), behavior RTT mean `84.865 ms → 51.529 ms`
  (`-39.28 %`), DB-completed FPS `4.439 → 4.820` (`+8.58 %`), but detection
  rows `72,762 → 101,213` (`+39.10 %`), bbox rows `72,762 → 101,213`
  (`+39.10 %`), `attention_tracking` boxes `11,781 → 20,558`
  (`+74.50 %`), and average GPU utilization `9.344 % → 7.367 %`
  (`-21.16 %`). Additional model-agreement measurement against the accepted
  320 Top-K baseline (`F1@IoU0.5`, not human-labeled accuracy) showed
  `attention_tracking=31.195 %`, `hand_raising=38.032 %`,
  `person_detection=100.000 %`, and `sitting_standing=65.250 %`; Step 2 FPS
  improved `8.403 → 11.594` (`+37.97 %`). Evidence:
  `backend/logs/cycle11-input256-realbench-20260602T161641Z/input_256_metrics.json`
  / `.md` plus
  `backend/logs/cycle11-input256-realbench-20260602T161641Z/model_agreement_320_vs_256.json`
  / `.md`.
  Reproducibility tooling:
  `tools/prod/prod_run_behavior_input_size_matrix.sh` runs the 320/256
  production matrix, and `tools/prod/prod_collect_benchmark_metrics.py`
  captures DB correctness, telemetry RTT, GPU CSV, and inference-audit metrics
  into JSON/Markdown evidence bundles. `tools/prod/prod_compare_benchmark_accuracy.py`
  captures per-model baseline-agreement metrics. Production was rolled back with
  `prod_set_behavior_input_size.sh --input-size 320 --topk 100`, workers were
  restarted, and final verified state is the accepted Cycle 9b B.2.c profile:
  `TRITON_CROP_BEHAVIOR_INPUT_SIZE=320`,
  `MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble_gaze_slice_topk`,
  `TRITON_BEHAVIOR_TOP_K_ENABLED=1`, `LPM_ENABLED=0`, Triton health `200`, and
  all active Top-K models `READY`. Runtime guard commit
  `4bcc79a5a4ea7c4d452b6fcd3ae3a6ff064a3bb5` adds explicit Triton model
  loading for `TRITON_LOAD_MODEL` and parameterized validator checks; follow-up
  commit `d378cec7` trims the non-320 Top-K Triton load list so future 256
  matrix runs do not try to load stale 320-only compatibility ensembles.
- **2026-06-02 Cycle 9b B.4 larger batch window NOT ACCEPTED by real
  production benchmark**: production benchmark
  `cycle9b-b4-maxframes4-20260602T175820Z`, job
  `416efe8c-772c-442f-8e55-cf44c54fe261`, raised
  `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES` from the accepted `2` to `4` while
  keeping `TRITON_CROP_BEHAVIOR_INPUT_SIZE=320`,
  `GAZE_HORIZONTAL_HEAD_VARIANT=slice`, and
  `MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble_gaze_slice_topk`.
  The run completed `4541/4541` frames and bounded sampled Celery RSS at
  `1120.328 MiB`. It improved Step 2 frame wall `540.399 s → 512.445 s`
  (`-5.17 %`) and DB-completed FPS `4.439 → 4.572` (`+3.00 %`), but failed
  the acceptance gate: behavior RTT mean regressed `84.865 ms → 99.251 ms`,
  persisted StudentTrack count dropped `53 → 47`, GPU peak moved
  `53 % → 49 %`, and baseline-agreement F1@IoU0.5 failed for
  `attention_tracking=24.531 %`, `hand_raising=26.648 %`, and
  `sitting_standing=17.217 %` (`person_detection=100.000 %`). Evidence:
  `docs/production_inference_benchmark.md` §21,
  `docs/cycle_9b_batch_window_results.md`, and
  `backend/logs/cycle9b-b4-maxframes4-20260602T175820Z/`. Production was
  restored to `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES=2`; do not repeat
  `max_frames=4` unless the tracking/model-agreement instability is addressed
  first.
- **2026-06-01 Cycle 10 STAGED — Logical Path Matrix (LPM)** —
  deterministic mathematical constraint layer applied AFTER the three gaze
  models (horizontal / vertical / depth) and BEFORE persistence. Scope is
  HARD-RESTRICTED to those three models only; RTMPose, person_detector, and
  posture_model are read-only inputs (C4 uses RTMPose head-yaw as a signal
  but never modifies pose outputs) or are not touched at all. Constraints
  C1–C4 enforce within-axis exclusivity (margin-based), temporal smoothness
  with hysteresis, impossible-state alarms, and pose-coupling. The pure math
  lives at `backend/apps/pipeline/services/logical_path_matrix.py`; C.2.1 is
  now locally wired into `backend/apps/video_analysis/tasks.py` behind
  `LPM_ENABLED`, with integration coverage in
  `backend/tests/unit/video_analysis/test_lpm_crop_behaviour_hook.py`.
  C.2.2 is also locally staged: `telemetry_lpm_events` is modeled in
  `backend/apps/telemetry/models.py`, migrated by
  `backend/apps/telemetry/migrations/0002_telemetrylpmevent.py`, collected via
  `TelemetrySession.record_lpm_event(...)`, flushed through the JSON +
  PostgreSQL telemetry writer, and emitted from the crop-frame hook through
  `_tel_record_lpm_event(...)`. Local validation for C.2.1-C.2.2 plus the
  post-benchmark safety fix: focused telemetry/LPM set `65 passed`; focused
  inference-parallelization workflow pytest set `156 passed`;
  `makemigrations --check --dry-run`, compileall,
  docs diagram verification, and `git diff --check` passed.
  Production benchmark `cycle10-lpm-crop-frame-20260601T201239` / job
  `17075418-4386-4b5f-85d4-ea23bec71f66` deployed SHA
  `377fb59e66fe46327e6cbf2e6e64fd3bb70a0bde` and proved the telemetry table
  works (`4541` LPM rows), but Cycle 10 is **NOT ACCEPTED**: `C1=0`,
  `eliminated=0`, and `attention_tracking` boxes regressed `11776 → 2680`.
  Production was rolled back to `LPM_ENABLED=0` and workers were restarted.
  Safety-fix proof `cycle10-lpm-violationonly-crop-frame-20260601T221110` /
  job `21666815-f4bd-4f5f-b90e-b9101b4d899d` deployed SHA
  `31edac44c66233baadd3a26ddd57b51b1a043d66` and completed, with summary
  `backend/logs/bench_summary_20260602T011132.json`, GPU CSV
  `backend/logs/gpu_monitor_bench_20260602T011132.csv`, and telemetry session
  `be855e0e-6393-467a-9688-b723a29a56a4`. It restored most of the first-run
  attention-box loss (`2680 → 11122`) but still failed Cycle 9 parity
  (`11776`), recorded `C1=0`, `eliminated=0`, and regressed DB-completed FPS
  (`4.09 → 4.039`). Production was rolled back again to `LPM_ENABLED=0`.
  Next LPM work must capture pre-decode gaze probabilities or move the pure
  math layer into the future compact postprocessing / Triton BLS path where
  dense gaze outputs are still available.
- **Constitutional re-affirmation (2026-06-01)**: **No optimization cycle may
  be marked accepted/success/agreed without a production benchmark on the
  Linux RTX 5090 server that demonstrates both (a) the targeted metric
  improvement and (b) zero correctness regression vs. the prior accepted
  baseline.** Code review, local testing, theoretical reasoning, and
  parity probes are necessary but never sufficient. Cycle 9 is the
  reference precedent: parity passed, code reviewed, FPS improved, but the
  designated acceptance gate (Step 2 wall) failed, so the cycle is held back
  even though the change is technically working. Every future cycle's
  acceptance section must cite the production replay key and job id.
- **2026-06-01 Cycle 8 ACCEPTED** (largest single-cycle win since Cycles 1–5):
  bundled `OFFLINE_EMBEDDING_REUSE_BY_TRACK=1` + lazy `cv2.VideoCapture` reads
  (skip `.set/.read` when every detection in a frame already has a cached
  track vector; forward-skip with `.grab()` instead of keyframe-rewind
  `.set()`) + new `persist_embeddings_bulk` helper that replaces 72 k single-
  row `FrameEmbedding.objects.create()` calls with `bulk_create` batches of
  `EMBEDDING_BULK_BATCH_SIZE=500`. The embedding loop now does ~53
  `model.embed()` calls instead of 72 579 (one per StudentTrack instead of
  per Detection). Replay key `cycle8-embed-bulk-crop-frame-20260601T125627`,
  job `d2de80a0-31b7-4a47-b9f1-d2e2156ea3a8`. **Embedding stage wall
  450.7 s → ~174 s (−61.5 %); total DB-completed elapsed 26.37 min →
  21.87 min (−17.1 % vs C7); overall FPS (DB-completed) 2.87 → 3.46
  (+20.5 % vs C7; +164 % vs baseline); detection/bbox/embedding parity
  within 0.003 %.** Evidence: `docs/production_inference_benchmark.md` §14,
  `docs/runtime_sla_video_plus_5min.md`.
- **2026-06-01 Cycle 9 NEEDS FURTHER ITERATION**: implemented a guarded
  `behavior_ensemble` Triton model and app route `behavior_all` so one crop
  batch upload feeds `posture_model`, `gaze_horizontal_model`,
  `gaze_vertical_model`, and `gaze_depth_model`, then splits the four named
  outputs back into the existing `output0` decode path. Production parity probe
  passed with max abs diff `0.0`. Replay key
  `cycle9-behavior-ensemble-crop-frame-20260601T180847`, job
  `c1651663-e08a-4e29-9ee3-fd0f09884b98`, candidate SHA
  `0fa847af43186017316cc11a8c76645ff463e574`. **DB-completed elapsed improved
  1312.3 s → 1110.7 s (+18.1 % FPS), app-level model calls fell 20 348 →
  9 557, behavior crop calls fell 14 391 → 3 597, and correctness parity was
  exact; however Step 2 wall stayed flat 852.8 s → 858.1 s (+0.6 %), so the
  Cycle 9 acceptance gate failed. Do not mark accepted.** The pinned production
  Triton build originally had `TRITON_ENABLE_ENSEMBLE=OFF`; it was rebuilt in
  place with `TRITON_ENABLE_ENSEMBLE=ON` after saving backup binary
  `/home/bamby/services/triton_build_r2502/tritonserver/install/bin/tritonserver.pre_cycle9_no_ensemble_20260601T180729`.
  Rollback is `TRITON_BEHAVIOR_ENSEMBLE=0`; redesign should reduce dense output
  movement or the four-child server-side critical path, not only Python request
  count. Evidence: `docs/production_inference_benchmark.md` §15 and
  `docs/cycle_9_results.md`.
- **2026-06-01 Cycle 7 ACCEPTED (with caveat)**: cache Redis client per `REDIS_URL`
  in both `apps.tracking.embeddings.redis_client` and
  `apps.video_analysis.tasks._redis_client`. The embedding loop calls these
  helpers ~217 k times per offline job. Replay key
  `cycle7-rediscache-crop-frame-20260601T120927`, job
  `515fe118-6009-4776-916d-6473fbf31ed7`. **Total DB-completed elapsed
  27.22 min → 26.37 min (−3.1 %); overall FPS 2.78 → 2.87 (+3.2 % vs C6;
  +119 % vs baseline); embedding stage 467.6 s → 450.7 s (−3.6 %);
  detection-row parity within 0.012 %.** The hypothesis predicted −69 %
  embedding wall but actual was only −3.6 % because `redis-py` 5.x's
  `Redis.from_url()` is much cheaper than I assumed (~0.08 ms per call, not
  ~1.5 ms). The cache is still kept (cleaner code, modest measurable win,
  zero risk); next cycle attacks the *real* embedding sub-costs (cv2 seek,
  per-row create, per-row exists query, model.embed single-crop). Evidence:
  `docs/production_inference_benchmark.md` §13.
- **2026-06-01 Cycle 6 ACCEPTED**: chunk `PoseRuntime._provider_infer_batch`
  at the deployed rtmpose `max_batch_size=16` (configurable via
  `TRITON_MODEL_BATCH_SIZE_OVERRIDES["pose_estimation"]`). Cycles 1–5
  enabled true-batch packing across all paths, but the pose path bypassed
  `_infer_task_batch` and let the orchestrator stack every per-crop payload
  into one gRPC call — when `tasks.py` `dynamic_cap` exceeded 16 (which
  happens once `avg_pose_ms_per_person` decays below ~88 ms on a fast GPU),
  every such frame got the `batch-size must be <= 16` Triton error and a
  silent HTTP-fallback round-trip. Replay key
  `cycle6-posechunk-crop-frame-20260601T022240`, job
  `a1a448b9-474f-4dea-942b-3288bcae6900`. **Pose post-processing wall
  12 m 13 s → 3 m 42 s (−69.7 %); total job `run.complete` 28 m 36 s →
  19 m 26 s (−32.1 % vs cycles 1–5); overall FPS (DB-completed basis)
  2.077 → 2.78 (+33.8 % vs cycles 1–5; +112 % vs baseline); rtmpose
  batch-warnings many → 0; detection-row parity 72 743 → 72 752 (+1 vs
  baseline 72 751, statistical noise).** Evidence:
  `docs/production_inference_benchmark.md` §12 and
  `docs/crop_frame_optimization_execution.md` Cycle 6.
- Goal: saturate the RTX 5090 (today ~1% GPU util, CPU-bound) and raise offline
  throughput from single-digit fps toward 100+ fps, every phase measured by the
  telemetry layer and shipped behind a flag with fallback.
- Diagrams: README → *Triton Model Inference End to End lifecycle (Sequential Order)*
  (current) and *Triton Model Inference End to End (Parallel)* (target).
- Status: **Repo implementation complete; ROI-320 production benchmark partial,
  final GPU-utilization certification pending.** Implemented and flag-gated: P1a binary tensors
  (`TRITON_BINARY_TENSORS`), P1b concurrent model dispatch
  (`TRITON_CONCURRENT_MODELS`, VRAM-bounded), P2 pipeline overlap
  (`TRITON_OFFLINE_PIPELINE_OVERLAP`), P3 gRPC transport
  (`TRITON_PROTOCOL_PREFERENCE=grpc`, fixed request-timeout type bug), P4 progress
  throttle + batched DB writer/offload (`OFFLINE_PROGRESS_UPDATE_EVERY_N`,
  `OFFLINE_DB_BATCH_WRITES`, `OFFLINE_OFFLOAD_POST_STAGES`), P5 dynamic batching
  in tracked Triton configs plus true client-side request batching
  (`TRITON_TRUE_BATCH_REQUESTS`, model caps via
  `TRITON_MODEL_BATCH_SIZE_OVERRIDES`), P6 VRAM/process discipline, P7a first-class
  `full_frame`/`crop_frame`, and P7b optional embedding/behaviour reuse. P7c Triton
  ensemble/BLS is classified as future advanced model-repository work, not a
  blocker for this plan's application implementation.
- Workflow: `.github/workflows/inference-parallelization.yml` is the focused local
  gate for this plan. It runs transport/config/crop-frame/batch-queue/docs checks
  and shell syntax checks for production helpers.
- Optimized production defaults are now applied through
  `tools/prod/prod_enable_parallel_flow.sh` and chained by
  `tools/prod/prod_run_parallel_flow_benchmark.sh`. The active benchmark default
  is `crop_frame` on
  `/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4`
  with gRPC, binary tensors, true Triton batch requests, concurrent model dispatch,
  pipeline overlap, batched DB writes, and post-stage offload enabled in `backend/.env`. The default
  profile is now `per-frame-signals`: `TRITON_OFFLINE_FRAME_STRIDE=1`, sparse
  `person_detection` is allowed with last-box reuse
  (`OFFLINE_DETECT_EVERY_N_FRAMES=5`,
  `OFFLINE_REUSE_LAST_BOXES_TTL_FRAMES=10`), decode/preprocess queueing is bounded
  (`TRITON_OFFLINE_DECODE_QUEUE_SIZE=4`), behaviour/gaze reuse is disabled so
  current-frame crop signals are recomputed, and accepted Cycle 8 track-level
  embedding reuse remains enabled (`OFFLINE_EMBEDDING_REUSE_BY_TRACK=1`).
- Current production benchmark run (started 2026-05-31 20:28 EEST):
  replay key `parallel-crop-frame-20260531T202819`, job
  `5801ef31-050f-4e20-a58e-d98122c5e920`, log
  `backend/logs/parallel_flow_parallel-crop-frame-20260531T202819.log`.
  First probe showed status `processing`, `25/4541` frames, 30-second window
  throughput `0.800 fps`, gRPC model-call telemetry present, and max effective
  batch size `4`. This run was cancelled at `50/4541` frames because the active
  crop-frame worker reached ~100 GB RSS. `TRITON_OFFLINE_BATCH_QUEUE_MAX_FRAMES`
  is now set to `1` for optimized crop-frame production runs to bound retained
  person-crop payloads and response lists.
- Follow-up run (started 2026-05-31 20:36 EEST): replay key
  `parallel-crop-frame-20260531T203638`, job
  `17518cbe-c320-4880-8265-62df0add1ae3`, log
  `backend/logs/parallel_flow_parallel-crop-frame-20260531T203638.log`.
  Memory is bounded compared with the prior attempt (~20-26 GB RSS observed),
  but GPU utilization samples remained near 0% and progress stalled at the next
  crop batch. The run was cancelled at `25/4541`; the follow-up contract now
  permits person-box reuse only. Behaviour/gaze and embedding reuse are disabled
  in the default profile because every frame must carry fresh current-frame
  signal predictions. Follow-up optimization skips full-frame tensor
  build/serialization on non-detection frames in `crop_frame`, keeping only the
  original pixels needed for current-frame crop inference, and avoids the generic
  `FrameCropper` PNG encode path in favor of direct bounded `xyxy` slicing.
  After the crop fanout memory spike, `TRITON_NUMPY_OUTPUTS=1` was added so gRPC
  YOLO outputs are decoded from NumPy arrays instead of Python list materialization.
- Active per-frame-signal production run (started 2026-05-31 21:39 EEST):
  replay key `parallel-per-frame-signals-crop-frame-20260531T213945`, job
  `c1a9117e-7f59-4c53-9731-b528ab5e6cbd`, log
  `backend/logs/parallel_flow_parallel-per-frame-signals-crop-frame-20260531T213945.log`.
  First 60-second probe showed `25/4541` frames, window throughput `0.400 fps`,
  `frame_signal_contract=per_frame_signals_with_person_box_reuse`, GPU sample
  up to `62%`, and DB frame rows pending until Step 3 persistence.
- Active replacement run after NumPy-output fix (started 2026-05-31 21:47 EEST):
  replay key `parallel-per-frame-signals-crop-frame-20260531T214740`, job
  `e8071c5a-8fb7-4c23-882d-621f8469c097`, log
  `backend/logs/parallel_flow_parallel-per-frame-signals-crop-frame-20260531T214740.log`.
  First probe confirmed `TRITON_NUMPY_OUTPUTS=True`, `25/4541` frames,
  `window_fps=0.400`, and the active worker RSS stayed near 12 GiB instead of
  the prior >100 GiB crop-fanout spike. The run was later cancelled at
  `100/4541` frames when RSS still climbed to ~49 GiB and throughput remained
  CPU-bound; follow-up implementation adds real batched Triton requests for
  crop fanout plus scoped gRPC channel reuse/cleanup. A subsequent guarded run
  `parallel-per-frame-signals-truebatch-guarded-crop-frame-20260531T221625`
  reached `50/4541` with one active job, proving the stale-redelivery guard, but
  was cancelled because true-batch splitting still copied per-crop NumPy output
  slices and RSS climbed to ~34.5 GiB. Follow-up no-copy split keeps batch-output
  views instead of contiguous per-crop copies. A later no-copy run
  `parallel-per-frame-signals-nocopy-crop-frame-20260531T222447` still showed
  worker RSS high-water growth (~33 GiB by `50/4541`), so
  `OFFLINE_TRIM_PROCESS_MEMORY=1` now calls Python GC + Linux `malloc_trim(0)`
  after each offline frame batch to return freed NumPy/gRPC buffers to the OS.
  The trim-enabled run
  `parallel-per-frame-signals-trim-crop-frame-20260531T223309` (job
  `a6a06fd6-97e8-4a82-b1ef-2b9fe132d667`) was cancelled at `75/4541` after RSS
  still climbed to ~37 GiB. Follow-up implementation keeps one job-scoped
  async/gRPC event loop for all offline Triton calls and replaces crop-frame
  full-history tracking rescans with incremental tracking; sparse person-box
  reuse now preserves assigned track IDs while behaviour/gaze predictions remain
  fresh per frame.
  Follow-up validation
  `parallel-per-frame-signals-looptrack-crop-frame-20260531T224816` (job
  `74572801-b257-4229-b37c-1e8ba5952b4e`) reached `75/4541` with worker RSS
  near ~2 GiB, confirming the memory fix, but throughput stayed near `0.4 fps`.
  The next CPU hot-path fix adds vectorized YOLO decode and
  `TRITON_YOLO_MAX_DECODE_CANDIDATES=100` before NMS so dense low-confidence
  anchors cannot dominate one CPU core; the 100-candidate prod probe reached a
  90-second window throughput of `1.111 fps` with worker RSS near ~1.7 GiB and,
  later, `425/4541` frames with a 120-second window of `1.042 fps`, overall
  `1.000 fps`, and worker RSS near ~1.6 GiB.
- Production hardening from the failed `all_merged.mp4` subjective run is now part
  of the plan: `prod_start_triton.sh` raises `TRITON_NOFILE_LIMIT` (default
  `65535`) and truncates oversized `triton.log` using `TRITON_LOG_MAX_MIB`
  (default `1024`); `prod_start_celery_workers.sh` reads worker time limits,
  worker concurrency, and guardrails from `backend/.env`; `prod_cancel_video_jobs.sh`
  stops old active jobs, purges the video queue, and terminates waiting ingest
  commands before a new benchmark; `prod_parallel_flow_probe.sh`
  verifies env/runtime/DB/model-call telemetry; `prod_verify_per_frame_signals.sh`
  checks the per-frame signal contract and DB frame completeness;
  `prod_run_benchmark.sh` accepts `--pipeline-mode`; `prod-runtime-ingest-video.ps1` accepts `crop_frame`; and
  `prod_check_subjective_progress.sh` provides DB-backed progress/ETA.
- Agents must treat this plan as the priority for runtime/inference work and update
  its task checklist + `docs/production_inference_benchmark.md` after each phase.

## Current Active Production Runtime Plan
- **Active plan name:** `Heterogeneous Production Runtime Maturity Plan` — **COMPLETED 2026-05-29**
- Plan file: [docs/heterogeneous_production_runtime_maturity_plan.md](docs/heterogeneous_production_runtime_maturity_plan.md)
- Tasks file: [docs/heterogeneous_production_runtime_maturity/tasks.md](docs/heterogeneous_production_runtime_maturity/tasks.md)
- Evidence root: [ci_evidence/production/runtime_maturity/](ci_evidence/production/runtime_maturity/)
- Accepted job: `b1d2311c-b0af-44a4-a551-61e58200eb11` | Final SHA: `af3fce3`

## Runtime Stability Remediation (Completed)
- **Plan:** [docs/runtime_stability_remediation/plan.md](docs/runtime_stability_remediation/plan.md)
- **Tasks:** [docs/runtime_stability_remediation/tasks.md](docs/runtime_stability_remediation/tasks.md)
- **Constitution:** Section 17 (§17.1–§17.4) — Runtime Job Lifecycle and Vector Integrity
- **Status:** All RS tasks completed and merged to master (SHA `375cc84`)
- Local workflow: [.github/workflows/prod-runtime-maturity.yml](.github/workflows/prod-runtime-maturity.yml)
- Production evidence helper: [tools/prod/prod-runtime-maturity-evidence.ps1](tools/prod/prod-runtime-maturity-evidence.ps1)
- **Status:** Active, not completed.
- **Branch target:** `release/prod-runtime-stabilization`
- This is the current controlling plan for production runtime work. Agents must treat it as the source of truth when it conflicts with older optimization notes.
- **Production policy update:** Development and production are intentionally heterogeneous. Local Windows/Docker/RTX 3050 Ti may validate code behavior and contracts, but only native Linux/no-Docker/RTX 5090 production validation can certify inference execution, GPU evidence, throughput claims, and final lifecycle acceptance.
- **Triton policy:** Production must use `INFERENCE_STRATEGY=triton_only`. Triton uses **dual configured endpoint profiles** (live and offline), but production runs **one active mode at a time** selected by `backend/.env` (`TRITON_EXECUTION_MODE=live` or `TRITON_EXECUTION_MODE=offline`). The inactive endpoint profile must not receive production inference traffic, scheduler requests, Celery routing, or production-ready health status.

## Superseded Optimization Context
- Previous plan name: `Linux Production Optimization Execution Phases (RTX 5090)`
- Previous plan file: [docs/linux_production_optimization_execution_phases.md](docs/linux_production_optimization_execution_phases.md)
- The previous plan remains useful for phase history and implementation details, but it is no longer the current top-level plan.

## Current Phase Status (handover snapshot)
- M0: partially completed (sync/hygiene repeatedly applied; local untracked `tmp/*` still exists on dev laptop).
- M1: partially completed (profile/cadence env contract documented, continue hardening env examples).
- M2: largely completed (queue routing + worker scripts added).
- M3:
  - Phase 3.1: in progress/partially implemented (sparse detect + reuse present).
  - Phase 3.2: not started (bbox interpolation/clamp/continuity guards).
- M4: partially touched, not accepted (live cadence/reuse/telemetry still needs full validation).
- M5: COMPLETED — single active endpoint policy enforced; offline Triton (39100) active, live (39000) isolated.
- M6: not completed (dynamic-batch TensorRT regeneration and tuned Triton configs still pending full sign-off).
- M7:
  - Phase 7.1: started (benchmark exporter/test scaffolding added).
  - Phase 7.2: not started (full profile matrix execution/report ranking).
- M8: COMPLETED — final acceptance gates passed; evidence packaged; branch merged to master.

## Heterogeneous Production Runtime Maturity — COMPLETED (2026-05-29)
- **Accepted job:** `b1d2311c-b0af-44a4-a551-61e58200eb11`
- **Final git SHA:** `af3fce3` (all locations in parity)
- **Runtime:** native Linux, no Docker, no sudo, RTX 5090, Triton offline (39100)
- **Evidence root:** `ci_evidence/production/runtime_maturity/`
- **Package index:** `ci_evidence/production/runtime_maturity/final/evidence_package_index.md`
- **Acceptance summary:** `ci_evidence/production/runtime_maturity/final/production_acceptance_summary.md`
- **Key runtime fixes applied during this cycle:**
  - `TRITON_FORCE_DOCKER=False` in backend/.env — was selecting non-existent Docker authority
  - `runtime_used` label corrected from `"hybrid"` to `"triton"` for triton-only paths (commit 12c291c)
  - `selected_authority` renamed from `triton_docker` to `triton_native` for native Linux
  - Embedding vector dimension coerced at DB write boundary (commit 441c28e, f689a4d)
  - Stale-job reconciler registered in Celery beat_schedule — no more stuck non-terminal jobs
- **Constitution:** bumped to v2.4.0 (Sections 17 + 18 added)

## Production Server Details (Linux, no Docker, no sudo)
- Host: `0.tcp.eu.ngrok.io`
- SSH port: `27681`
- Username: `bamby`
- Repo path: `/home/bamby/grad_project`
- Runtime constraints:
  - no Docker
  - no sudo
  - CUDA runtime authority: **CUDA 12.8**
  - TensorRT runtime authority must be pinned and rebuilt against CUDA 12.8-compatible stack
  - Triton-only inference on NVIDIA GPU
  - dual Triton endpoint profiles configured, one active production mode at a time
  - backend runtime env authority is `backend/.env`; repo-root `.env` must not participate in backend startup
  - PostgreSQL-backed persistence only; SQLite-backed execution is invalid in prod and invalid for production evidence

## Production Toolchain Pinning (Mandatory)
- Triton production default binary MUST be pinned to:
  - `/home/bamby/services/triton_build_r2502/tritonserver/install/bin/tritonserver`
- Python command in production shells MUST resolve consistently to backend venv:
  - `/home/bamby/grad_project/backend/.venv/bin/python`
- Agents must ensure both `~/.bashrc` and `~/.profile` export stable user-path resolution for:
  - Python (`PATH` includes backend venv `bin`)
  - Triton (`TRITON_SERVER_BIN` and optional `PATH` prepend)
  - pip user cache (`PIP_CACHE_DIR=/home/bamby/.cache/pip`)
- Duplicate or conflicting shell-export lines in `~/.bashrc` / `~/.profile` are considered drift and must be cleaned.

### Required Prod Shell Exports
- `export PATH="/home/bamby/grad_project/backend/.venv/bin:$PATH"`
- `export TRITON_SERVER_BIN="/home/bamby/services/triton_build_r2502/tritonserver/install/bin/tritonserver"`
- `export PIP_CACHE_DIR="/home/bamby/.cache/pip"`
- Optional convenience alias:
  - `alias python="/home/bamby/grad_project/backend/.venv/bin/python"`

### TensorRT Engine Compatibility Rule
- If runtime error contains serialization/version mismatch (for example engine version tag mismatch), treat all affected `.engine`/`model.plan` artifacts as incompatible.
- Rebuild required engines with the active production TensorRT stack, then refresh Triton model repository artifacts before reloading Triton.
- Rebuild success is not enough unless Triton model load status is `READY` for required models.
- Current commonly used app ports:
  - Backend: `127.0.0.1:8011`
  - Frontend: `127.0.0.1:5174`
  - Triton (offline profile mode): `39100/39101/39102`
  - Triton (live profile mode, only when selected): `39000/39001/39002`
  - App Redis/Celery broker: `6380` (user-space Redis)
  - System Redis: `6379` may require auth and must not be assumed usable for app workers
  - PostgreSQL may run non-default in this environment (check with `ss -ltnp` and `.env`)

## Windows Connection + Tooling Workflow
- Preferred client tools on Windows:
  - OpenSSH `ssh` / `scp` (built into modern Windows)
  - PowerShell scripts in `tools/prod`
- Optional/fallback:
  - PuTTY `pscp` if OpenSSH transfer has issues

### Recommended helper scripts
- `tools/prod/prod-ssh.ps1`:
  - `.\tools\prod\prod-ssh.ps1`
  - `.\tools\prod\prod-ssh.ps1 -Cmd "hostname"`
- `tools/prod/prod-copy-to.ps1`:
  - `.\tools\prod\prod-copy-to.ps1 -LocalPath "E:\grad_project\Raw Data\Video Project 1.mp4" -RemotePath "/home/bamby/grad_project/raw_test_videos/"`
- `tools/prod/prod-copy-from.ps1`
- `tools/prod/prod-workers.ps1` (`preflight|start|status|stop`)
- `tools/prod/prod-health-snapshot.ps1`
- `tools/prod/prod-hash-parity.ps1`
- `tools/prod/prod-stash-hygiene.ps1`
- `tools/prod/prod-sync-env-keys.ps1`

## Known Open Issues (2026-05-30) — ALL RESOLVED

- **~~Celery beat not running on prod.~~ RESOLVED 2026-05-30.** Beat is now a managed
  part of the worker lifecycle. `prod_start_celery_workers.sh` starts it via
  `tools/prod/prod_start_celery_beat.sh` (skippable with `CELERY_BEAT_DISABLED=1`), and
  `prod_stop_celery_workers.sh` stops it via `tools/prod/prod_stop_celery_beat.sh`. Both
  beat scripts are idempotent and refuse to launch a second scheduler. The stale-job
  reconciler (`reconcile_stale_jobs_periodic`) and all periodic tasks now run. Manual
  control is also exposed through `tools/prod/prod-workers.ps1 -Action beat-start|beat-stop|beat-status`.
- **~~Celery soft-time-limit=300s too short for large offline videos.~~ RESOLVED 2026-05-30.**
  `VIDEO_UPLOAD_TASK_SOFT_TIME_LIMIT_SECONDS` default raised 600→**1800** and
  `VIDEO_UPLOAD_TASK_TIME_LIMIT_SECONDS` default 660→**1920** in
  `backend/config/settings/base.py` (env-overridable; `.env.example` already uses 1800/1860).
  The worker-level fallback in `prod_start_celery_workers.sh` was raised from 300/600 to
  **1800/2400** seconds. Long offline videos now complete within the soft limit.
- **~~Full benchmark run never completed end-to-end.~~ RESOLVED 2026-05-30 (unblocked + automated).**
  Both blockers above are fixed, so the benchmark can complete. A one-shot runner
  `tools/prod/prod_run_benchmark.sh` performs the full re-run checklist (pre-flight, beat
  ensure, GPU monitor, job submit with `--timeout-seconds 3600`, telemetry summary). See
  `docs/production_inference_benchmark.md` §8. Final throughput certification still requires
  executing this runner on the prod RTX 5090 host.

## Production Startup/Validation Checklist (for next agents)
1. Confirm branch and hash parity:
   - local: `git rev-parse HEAD`
   - remote: `ssh prod-grad "cd /home/bamby/grad_project && git rev-parse HEAD"`
2. Confirm single Triton endpoint policy:
   - active endpoint health returns `200`
   - inactive endpoint returns `000`/unreachable
3. Confirm backend/model-serving health:
   - `http://127.0.0.1:8011/api/v1/health/`
   - `http://127.0.0.1:8011/api/v1/health/model-serving/`
   - interpret `redis=degraded` separately from `redis=unhealthy`; auth-challenge is a degraded state, not a process-down state
   - if runtime values disagree with `backend/.env`, treat repo-root `.env` as drift to be removed from prod startup assumptions
4. Confirm ports:
   - `ss -ltnp | egrep '39100|39000|8011|5174|6379|6380|5432|55432'`
   - confirm backend DB settings point to PostgreSQL, not SQLite
5. Run focused tests before full suites:
   - `backend/tests/unit/video_analysis/test_tasks_detection_cadence.py`
   - `backend/tests/unit/scripts/test_benchmark_export_csv.py`
   - `backend/tests/unit/pipeline/test_runtime_mode_authority.py`
   - `backend/tests/contract/test_runtime_mode_contract.py`
   - `backend/tests/system/test_wave1_runtime_policy.py`
   - scaffold integration/system tests (xfail markers are expected until converted)
6. Validate toolchain pinning:
   - `which python` resolves to backend venv python
   - `echo $TRITON_SERVER_BIN` points to pinned binary
   - `python -c "import tensorrt as trt; print(trt.__version__)"` runs in venv
7. Reload Triton and verify model readiness:
   - start selected mode only (`live` or `offline`)
   - `curl` active `/v2/health/ready` returns `200`
   - inactive mode endpoint remains unreachable

## Triton Startup Guide — Prod Linux Server (RTX 5090, No Docker, No sudo)

This section documents every issue encountered getting Triton running on
`/home/bamby/grad_project` and the permanent fixes applied. Read this before
touching Triton on prod.

### Hardware & Software Constraints

| Item | Value |
|------|-------|
| GPU | NVIDIA GeForce RTX 5090, 32 GB GDDR7 |
| CPU | 32 cores |
| CUDA | 12.8 |
| Triton binary | `/home/bamby/services/triton_build_r2502/tritonserver/install/bin/tritonserver` |
| TRT backends dir | `/home/bamby/services/triton_build_r2502/tritonserver/install/backends` |
| Model repository | `/home/bamby/grad_project/backend/models/triton_repository_cuda12` |
| Active TRT version | **10.16.1.11** (backend/.venv python3.12) — must match `.plan` engine build |
| Serialization version | **240** (what the `.plan` engines were compiled with) |
| Postgres port | **55432** (non-standard — always check `.env`) |

---

### Issue 1 — `libnvinfer.so.10: cannot open shared object file`

**Symptom:**
```
load failed for model 'person_detector': unable to load shared library: libnvinfer.so.10
```

**Root cause:**
Non-interactive SSH sessions (`bash -l -s` or `ssh prod-grad "cmd"`) do **not**
automatically source `~/.bashrc`. The `LD_LIBRARY_PATH` set in `~/.bashrc` was
never exported into the process that launched Triton via `nohup`, so the
TensorRT backend plugin (`libtriton_tensorrt.so`) could not find `libnvinfer.so.10`.

The Triton binary RUNPATH is `${ORIGIN}/../lib` (= `install/lib`) and the
TRT backend RUNPATH is `${ORIGIN}` (= `backends/tensorrt/`). Neither location
contains `libnvinfer.so.10`. There is no system `ldconfig` entry for it either.
The library lives only in the Python venvs under `site-packages/tensorrt_libs/`.

**Fix applied (permanent):**

Both `~/.bashrc` and `~/.profile` now export the correct `LD_LIBRARY_PATH`:
```bash
export LD_LIBRARY_PATH="/home/bamby/grad_project/.venv/lib/python3.11/site-packages/tensorrt_libs:/home/bamby/services/triton/lib:${LD_LIBRARY_PATH:-}"
```

A dedicated start script `tools/prod/prod_start_triton.sh` was created that
`source ~/.bashrc` before calling the Triton binary, so the correct
`LD_LIBRARY_PATH` is always in scope regardless of how the script is invoked.

**Rule for future agents:**
Never start Triton with a bare `nohup tritonserver ...` in a non-login shell.
Always use:
```bash
bash -l /home/bamby/grad_project/tools/prod/prod_start_triton.sh
```

---

### Issue 2 — TRT serialization version mismatch (RESOLVED — engines rebuilt for TRT 10.16.1.11)

**Historical symptom:**
```
IRuntime::deserializeCudaEngine: Error Code 1: Serialization
(Serialization assertion stdVersionRead == kSERIALIZATION_VERSION failed.
Version tag does not match.)
```

**Root cause & resolution:**
The project was upgraded from TRT 10.8.0.43 (serial tag 239) to TRT 10.16.1.11 (serial tag 240).
All `.plan` engines were rebuilt with TRT 10.16.1.11 and deployed to `triton_repository_cuda12`.
The legacy py3.11 `.venv` is no longer used. The single authoritative venv is `backend/.venv` (Python 3.12).

**Current state:**
- `LD_LIBRARY_PATH` → `backend/.venv/lib/python3.12/site-packages/tensorrt_libs` (TRT 10.16.1.11, tag 240)
- All `.plan` engines in `triton_repository_cuda12` compiled with TRT 10.16.1.11
- Engine/TRT version locked in `backend/models/tensorrt_builds/latest_compat.json`

**Guard mechanism (prevents recurrence):**
`prod_trt_guard.sh` compares the installed TRT version against `latest_compat.json` and
blocks `prod_start_triton.sh` if they diverge. After any `uv sync`, `prod_post_sync.sh`
also warns if engines need a rebuild.

**Rule for future agents:**
- After any TRT version change run: `bash tools/prod/prod-rebuild-tensorrt-engines.sh`
- Never start Triton if `prod_trt_guard.sh` returns non-zero.
- `backend/models/tensorrt_builds/latest_compat.json` is the single source of truth for which TRT version the active engines require.

---

### Issue 3 — Duplicate Celery Workers Causing Job Stalls and Timeouts

**Symptom:**
- Video analysis jobs queue successfully but stall indefinitely or hit timeout.
- `ps aux | grep 'celery -A config'` shows 3× the expected number of processes
  (e.g., 3 sets of offline workers all consuming from the same queue).

**Root cause:**
The `prod_start_celery_workers.sh` script was run multiple times without stopping
the existing workers first. The PID file check (`kill -0 $OLD_PID`) only guards
against starting a second copy of a single named worker, but if the PID file was
stale or absent, a new set of workers launched alongside the existing ones.
Multiple workers competing on the same queue causes race conditions: one picks
up the task, the others time out waiting, creating apparent stalls.

**Fix applied (permanent):**
- `prod_stop_celery_workers.sh` must always be run before `prod_start_celery_workers.sh`.
- `prod_start_triton.sh` now also stops any existing Triton instance before
  launching a fresh one, preventing accumulation of stale processes.

**Rule for future agents:**
```bash
# Always stop before starting — never just start
bash tools/prod/prod_stop_celery_workers.sh
bash tools/prod/prod_start_celery_workers.sh
```
After starting, verify exactly one set of workers:
```bash
ps aux | grep 'celery -A config worker' | grep -v grep | wc -l
# Expected: 5 parent processes (one per queue) + their child forks
# If > 15 processes, there are duplicates — stop and restart
```

---

### Issue 4 — `PYRAMID_WORKER_COUNT=24` Fails Pydantic Validation

**Symptom:**
```
pydantic_core.ValidationError: 1 validation error for PipelineConfig
worker_count: Input should be less than or equal to 16
```
Celery workers crash immediately on startup; PID files are created but processes
exit within seconds.

**Root cause:**
`PipelineConfig` in `backend/apps/pipeline/config.py` had `le=16` on
`worker_count`. Setting `PYRAMID_WORKER_COUNT=24` (appropriate for a 32-core
machine) exceeded this limit.

**Fix applied (permanent):**
Raised the Pydantic limit from `le=16` to `le=32` in `config.py`. Committed
and pushed as `f878620`. Both prod and local now accept values up to 32.

**Rule for future agents:**
If you add `PYRAMID_WORKER_COUNT` to `.env` and workers crash immediately,
check the Celery log at `backend/logs/celery_*.log` for `ValidationError`
before assuming a runtime or import problem.

---

### Issue 5 — Triton Log Exhaustion + Low File-Descriptor Limit

**Symptom:**
- `backend/logs/triton.log` grows without bound and fills `/`.
- PostgreSQL enters recovery mode after disk exhaustion.
- Triton holds GPU memory but reports near-zero GPU utilization while CPU spins.
- Triton log contains repeated:
```
[warn] Error from accept() call: Too many open files
```

**Root cause:**
Triton inherited a low `nofile` soft limit (`1024`) and then repeatedly logged
accept failures. In the `all_merged.mp4` subjective run this produced a
~193 GiB `triton.log`, exhausted the root filesystem, and caused the job to fail
with `reconciled_stale_processing_state`.

**Fix applied (permanent repo-side):**
- `tools/prod/prod_start_triton.sh` reads `TRITON_NOFILE_LIMIT` from
  `backend/.env` and defaults to `65535`.
- `tools/prod/prod_start_triton.sh` reads `TRITON_LOG_MAX_MIB` from `backend/.env`
  and truncates an oversized `backend/logs/triton.log` to the last 5000 lines
  before launching Triton.
- `tools/prod/prod_disk_cleanup.sh --delete` remains the manual cleanup tool while
  services are running.

**Rule for future agents:**
Before long offline benchmarks, check:
```bash
df -h /home/bamby/grad_project
pid=$(cat /home/bamby/grad_project/backend/logs/triton.pid)
grep 'Max open files' /proc/$pid/limits
du -h /home/bamby/grad_project/backend/logs/triton.log
```
Do not accept production evidence from a run that occurred while disk was full,
PostgreSQL was in recovery, or Triton was at its file-descriptor limit.

---

### Canonical Triton Start Procedure (after any restart or code change)

```bash
# 1. Stop all workers and Triton
bash /home/bamby/grad_project/tools/prod/prod_stop_celery_workers.sh

# 2. Start Triton (sources .bashrc for correct LD_LIBRARY_PATH)
bash -l /home/bamby/grad_project/tools/prod/prod_start_triton.sh

# 3. Wait for all 6 models to load (~60–90 seconds)
sleep 70

# 4. Verify Triton ready
curl -sf http://127.0.0.1:39100/v2/health/ready && echo READY || echo FAILED

# 5. Start Celery workers
cd /home/bamby/grad_project && bash tools/prod/prod_start_celery_workers.sh

# 6. Full health check
curl -sf http://127.0.0.1:8011/api/v1/health/ | python3 -m json.tool
```

### Quick Diagnostic Checklist

```bash
# Is Triton ready?
curl -sf http://127.0.0.1:39100/v2/health/ready && echo READY

# Which TRT version is loaded? (must print 10.16.1.11)
/home/bamby/grad_project/backend/.venv/bin/python3 -c "import tensorrt; print(tensorrt.__version__)"

# Is LD_LIBRARY_PATH pointing to backend py3.12 venv?
echo $LD_LIBRARY_PATH | grep 'python3.12'

# Does installed TRT match compiled engines?
bash /home/bamby/grad_project/tools/prod/prod_trt_guard.sh

# How many Celery workers? (expect 5 parents + forks, no duplicates)
ps aux | grep 'celery -A config worker' | grep -v grep | wc -l

# GPU state
nvidia-smi --query-gpu=memory.used,memory.free,utilization.gpu --format=csv,noheader

# Are there duplicate workers?
ps aux | grep 'celery -A config worker' | grep -v grep | \
  awk '{print $12}' | sort | uniq -c | sort -rn | head -10
# If any queue name appears more than once → stop all workers and restart
```

### Key File Locations

| File | Purpose |
|------|---------|
| `tools/prod/prod_start_triton.sh` | Permanent Triton launcher — calls TRT guard before launch |
| `tools/prod/prod_start_celery_workers.sh` | Starts all offline/default workers with tuned limits |
| `tools/prod/prod_stop_celery_workers.sh` | Stops all workers gracefully |
| `tools/prod/prod_trt_guard.sh` | TRT version guard — blocks Triton if engines are stale |
| `tools/prod/prod-rebuild-tensorrt-engines.sh` | Full engine rebuild + Triton restart workflow |
| `tools/prod/prod_post_sync.sh` | Run after every `uv sync` — installs TRT + warns if rebuild needed |
| `tools/prod/prod_update_bashrc.sh` | Idempotently fixes `~/.bashrc` and `~/.profile` paths |
| `tools/prod/prod_disk_cleanup.sh` | Purge `__pycache__`, probe engines, stale rollbacks, old logs |
| `backend/models/tensorrt_builds/latest_compat.json` | Source of truth: TRT version the active engines require |
| `~/.bashrc` and `~/.profile` | Managed by `prod_update_bashrc.sh` — TRT 10.16.1.11 / py3.12 paths |
| `backend/.env` | Active runtime config authority for all services |
| `backend/logs/triton.log` | Triton startup and model load log |
| `backend/logs/celery_*.log` | Per-worker startup and task execution log |

---

## Notes About `xfail` Scaffold Tests
- Some tests are intentionally marked `xfail(strict=True)` as implementation scaffolds.
- `xfailed` is not a regression by itself.
- When behavior is fully implemented, remove `xfail` and convert to normal passing assertions in the same phase commit.

## Parallel Testing Policy
- Prefer framework-native parallelism first:
  - Frontend unit tests: Vitest workers.
  - Frontend E2E: Playwright workers.
  - Backend tests: `pytest-xdist` workers.
- Avoid nested parallel collisions:
  - Do not run multiple backend `pytest -n ...` invocations at the same time unless DB isolation is guaranteed per invocation.
- Use percentage-based worker tuning on shared machines.

## Standard Commands
- Frontend unit (parallel):
  - `npm run test:unit:parallel`
- Frontend E2E (parallel):
  - `npm run test:e2e:parallel`
- Frontend combined parallel suite:
  - `npm run test:all:parallel`
- Backend unit/integration/contract/system (parallel per suite):
  - `.\.venv\Scripts\python.exe -m pytest backend/tests/<suite> -n auto --dist=loadscope -q --tb=short`

## Automation Testing
- Use fast feedback first, then broaden scope:
  - PR gate: unit + API contract + smoke.
  - Pre-release: full E2E + accessibility + visual regression.
- Practical parallel commands:
  - Frontend automation (unit + E2E): `npm run test:all:parallel`
  - Backend automation by suite (run one suite at a time): `.\.venv\Scripts\python.exe -m pytest backend/tests/<suite> -n auto --dist=loadscope -q --tb=short`
  - API contract (parallel): `.\.venv\Scripts\python.exe -m pytest backend/tests/contract -n auto --dist=loadscope -q --tb=short`
  - Smoke/canary (parallel, tagged): `.\.venv\Scripts\python.exe -m pytest -m "smoke or canary" -n auto --dist=loadscope -q --tb=short`

## Load Testing
- Goal: validate throughput, latency, and error rate under concurrent traffic.
- Use staged runs:
  - Baseline: short warm-up + steady-state check.
  - Stress: ramp beyond expected peak.
  - Soak: longer duration for leak/drift detection.
- Practical parallel commands:
  - k6 local workers: `k6 run --vus 50 --duration 5m load-tests/baseline.js`
  - k6 scenario file: `k6 run load-tests/scenarios/stress.js`
  - Locust multi-process: `locust -f load-tests/locustfile.py --headless -u 200 -r 20 -t 10m --processes -1`

## Additional Test Types
- Performance regression:
  - `.\.venv\Scripts\python.exe -m pytest backend/tests/performance -n auto --dist=loadscope -q --tb=short`
- Chaos/resilience:
  - `.\.venv\Scripts\python.exe -m pytest backend/tests/resilience -n auto --dist=loadscope -q --tb=short`
- Security:
  - `npm audit --audit-level=high`
  - `.\.venv\Scripts\python.exe -m pip-audit`
- Accessibility:
  - `npx playwright test --grep @a11y --workers=50%`
- Visual regression:
  - `npx playwright test --grep @visual --workers=50%`
- API contract (consumer/provider):
  - `.\.venv\Scripts\python.exe -m pytest backend/tests/contract -n auto --dist=loadscope -q --tb=short`
- Smoke/canary:
  - `npx playwright test --grep @smoke --workers=50%`
  - `.\.venv\Scripts\python.exe -m pytest -m canary -n auto --dist=loadscope -q --tb=short`

## Best-Practice Rules
- Keep tests isolated: no shared mutable cross-test state.
- Mock external dependencies in E2E where practical to reduce flakiness.
- Keep coverage collection enabled, but enforce thresholds only through explicit configured gates.
- Do not introduce SQLite-only test shortcuts or fallback database settings; keep test and runtime behavior aligned with PostgreSQL.

---

## Telemetry & Metrics Layer

### Status: **IMPLEMENTED** — all integration points wired. Run `python manage.py migrate` on prod to apply the DB migration.

### Mandate

Every run — whether a pytest/benchmark test, an RTSP live video streaming session, or an offline video-processing job — **must** route its timing and quality signals through a single, dedicated telemetry layer. No ad-hoc `time.time()` calls, no scattered log lines. One layer, one contract, one sink.

### Scope of collection

The layer must compute and record metrics at **four granularities**:

| Granularity | Key metrics |
|---|---|
| **Session** | total wall time, total frames processed, mean/p50/p95/p99 frame latency, mean/p95 end-to-end RTT per model call, drop rate, GPU utilisation summary, peak GPU memory, Triton model load time |
| **Video** | video ID/name, source type (`rtsp` \| `offline` \| `test`), fps observed vs. target fps, total frames, processing duration, per-video latency stats, detection count, model call count |
| **Frame** | frame index, timestamp (wall + stream PTS if available), pre-process latency, inference latency (per model: detector, embedder, classifier), post-process latency, total pipeline latency, detection count |
| **Student** (if applicable) | student ID, frames tracked, average confidence, embedding distance stats, recognition latency, first-seen / last-seen timestamps within the session |

### Required metrics per model call (RTT)

For every Triton HTTP/gRPC call record:
- `model_name` — which model was called (`person_detector`, `face_embedder`, `face_classifier`, etc.)
- `request_sent_at` — monotonic timestamp before sending
- `response_received_at` — monotonic timestamp after receiving
- `rtt_ms` — round-trip time in milliseconds
- `input_shape` — shape of the tensor sent
- `status` — `ok` | `timeout` | `error`
- `error_detail` — filled only on non-ok status

### Persistence contract

Each run must produce **both** of the following outputs atomically at session end (or on graceful shutdown):

1. **PostgreSQL rows** — written to dedicated telemetry tables:
   - `telemetry_sessions` — one row per run
   - `telemetry_videos` — one row per video within the run
   - `telemetry_frames` — one row per processed frame (may be batched for efficiency)
   - `telemetry_model_calls` — one row per Triton model call
   - `telemetry_students` — one row per student tracked per session (if applicable)

2. **JSON file** — one file per run, written to `backend/logs/telemetry/` with filename:
   ```
   <session_id>_<source_type>_<YYYYMMDD_HHMMSS>.json
   ```
   Schema mirrors the DB tables above; top-level keys: `session`, `videos`, `frames`, `model_calls`, `students`.
   The file must be written even if the DB write fails — file is the guaranteed fallback record.

### Integration points

The layer must integrate at these locations without polluting business logic:

- **Triton client wrapper** (`backend/apps/pipeline/triton_client.py` or equivalent) — wrap every `infer()` call to record RTT automatically.
- **Frame pipeline entry/exit** (`backend/apps/pipeline/` frame processing loop) — inject telemetry context at frame intake and flush at frame completion.
- **Celery task boundaries** (`backend/apps/video_analysis/tasks.py` or equivalent) — record session start/end and bind session ID to the task context.
- **RTSP stream manager** — record stream open time, first frame arrival time, and any reconnect events.
- **Test harness** — benchmark and integration tests must call `TelemetrySession.start()` / `.end()` so test runs are also captured in the same schema.

### API contract (internal)

```python
# backend/apps/telemetry/session.py  (canonical module path)

class TelemetrySession:
    def __init__(self, source_type: Literal["rtsp", "offline", "test"], metadata: dict): ...
    def start(self) -> str: ...          # returns session_id (UUID)
    def record_frame(self, frame_meta: FrameMeta) -> None: ...
    def record_model_call(self, call_meta: ModelCallMeta) -> None: ...
    def record_student(self, student_meta: StudentMeta) -> None: ...
    def end(self) -> SessionSummary: ...  # flushes DB + JSON atomically
    def __enter__(self): ...
    def __exit__(self, *_): ...
```

- `TelemetrySession` must be thread-safe and safe to use from Celery async tasks.
- Frame records may be buffered in memory and flushed in batches of ≤ 500 rows to avoid DB write amplification on high-fps streams.
- The JSON file must be written **before** the DB transaction commits; if the transaction rolls back, the JSON file is retained with a `db_commit_failed: true` flag so data is never silently lost.

### Code and documentation standards

- All telemetry code lives under `backend/apps/telemetry/` — no telemetry logic outside this package.
- Public functions and classes must have a one-line docstring stating *what* they measure.
- DB migrations for telemetry tables must be added as numbered Django migration files.
- A `README.md` inside `backend/apps/telemetry/` must document: the table schema, the JSON file schema, how to query common metrics (e.g., "p95 inference latency for the last 10 sessions"), and the integration checklist.
- Unit tests for the layer must cover: session lifecycle, frame batching, model-call recording, JSON file write, DB failure fallback.

### Rules for next agents

1. **Implement this layer before continuing any other runtime work.** All M-phase tasks that produce latency or throughput numbers must route through it.
2. The layer must work for **all three run modes**: test (`pytest`), RTSP live streaming, and offline video processing. If a mode is not yet wired up, add a `TODO(telemetry): wire <mode>` comment at the integration point and note it in the session-end JSON under `unwired_sources`.
3. Never write a duplicate telemetry sink. If a metrics reporter already exists, refactor it to call this layer instead of running in parallel.
4. After wiring all integration points, update this section's status to **IMPLEMENTED** and record the canonical module paths, migration file name, and the git SHA of the implementing commit.

### Implemented (2026-05-30)

- **Package:** `backend/apps/telemetry/`
- **Models:** `models.py` — six DB tables (`telemetry_sessions`, `telemetry_videos`, `telemetry_frames`, `telemetry_model_calls`, `telemetry_lpm_events`, `telemetry_students`)
- **Public API:** `session.py` — `TelemetrySession`, `FrameMeta`, `ModelCallMeta`, `LpmEventMeta`, `StudentMeta`, `VideoSummary`, `SessionSummary`
- **Writer:** `writer.py` — atomic JSON-first + PostgreSQL dual-sink; JSON fallback with `db_commit_failed: true` on DB error
- **Metrics:** `metrics.py` — `percentile`, `mean`, `latency_summary`, `per_model_summary`
- **Migrations:** `migrations/0001_initial.py`, `migrations/0002_telemetrylpmevent.py`
- **Tests:** `backend/tests/unit/telemetry/` — `test_telemetry_layer.py` (layer unit tests) + `test_integration.py` (wiring); **41 tests, all passing** (verified 2026-05-30)
- **Django app registered:** `apps.telemetry` added to `INSTALLED_APPS` in `config/settings/base.py`
- **JSON output directory:** `backend/logs/telemetry/`

### Integration wiring — COMPLETE (2026-05-30)

| Integration point | File | How it works |
|---|---|---|
| Celery task lifecycle | `backend/apps/telemetry/celery_integration.py` | `task_prerun` / `task_postrun` signals auto-start/end sessions for `process_video_upload`, `run_live_stream_inference`, `ingest_runtime_event_task` |
| ContextVar propagation | `backend/apps/telemetry/context.py` | Same-thread access to active session without threading it through every call; mirrors `pipeline/logging_context.py` pattern |
| Triton client RTT | `backend/apps/pipeline/services/triton_client.py` | `_try_record_telemetry()` called at both `infer()` exit paths; reads ContextVar, records `ModelCallMeta` |
| Offline frame recording | `backend/apps/video_analysis/tasks.py` | `_tel_record_frame()` helper called inside `_on_frame_complete` (multi-model path) and `_on_triton_frame_inferred` (Triton-only path) |
| Offline LPM event recording | `backend/apps/video_analysis/tasks.py` | `_tel_record_lpm_event()` records `LpmEventMeta` when `LPM_ENABLED=1`; production run proved rows flush, but LPM remains disabled after Cycle 10 rejection |
| Live stream frame recording | `backend/apps/video_analysis/tasks.py` | `_tel_record_frame()` helper called inside live `_on_frame_complete` after `stage_timings` is computed |

**Migration**: Apply `backend/apps/telemetry/migrations/0001_initial.py` and `backend/apps/telemetry/migrations/0002_telemetrylpmevent.py` with `python manage.py migrate` before first production run.
**JSON output**: `backend/logs/telemetry/<session_id>_<source_type>_<YYYYMMDD_HHMMSS>.json`
