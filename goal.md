# MISSION: Serve OSNet-AIN person-ReID in Triton (TensorRT), wire it into the
# Cycle 18.D offline cross-shard boundary association, keep it strictly separated
# from the live/RTSP path, then run the governed production benchmark and record
# an honest §12.6 decision. Local tests and proxy metrics MUST NOT be presented
# as acceptance.

You are continuing Cycle 18.D (offline video sharding identity association). Every
mechanically-fixable blocker is already fixed (packet validity 2/2, shard-0
merge-ready, StudentTrack inflation 64→55, mapping 10/36→18/36, honest provenance,
a fused geometry+motion+appearance one-to-one association). The cycle is still
NOT ACCEPTED for ONE root cause: the production detector is exported to
OpenVINO/TensorRT, so Ultralytics `model.embed()` is incompatible and appearance
falls back to a weak hand-crafted descriptor — there is NO learned ReID embedding
in the runtime. Your job: deploy a real ReID network as a Triton TensorRT model
and feed its embeddings to the offline boundary appearance producer, with a clean,
governed separation from the live RTSP path.

================================================================================
## 0. MANDATORY READING (in order — do not skip; treat every claim as verifiable)
================================================================================
1. `AGENTS.md`: "Production Server Details", "Production Toolchain Pinning",
   "Windows Connection + Tooling Workflow", "Triton Startup Guide (RTX 5090)",
   the Streaming-Source Compatibility section, the Identity-Association section,
   and the Benchmark Figure Evidence section.
2. `.specify/memory/constitution.md`: §1.7 (deployment authority), §2 (Triton
   authority + dual endpoint profiles + startup gates), §2.5/§2.6 (mode ownership,
   fault isolation), §3 (temporal), §4.1/§4.3/§4.6 (identity + independent-run/
   cross-shard doctrine), §5 (pose, for context), §7.1.1 + §12.5/§12.6 (benchmark
   decision authority + figure evidence), §8.4 + §8.6 (streaming collapse-prevention,
   live invariants, offline-only levers), §17 (vector dimension contract), §18 (CI
   file visibility), §19 (anti-hallucination).
3. `docs/cycle_18d_combined_cost_boundary_association_investigation.md` (full
   history incl. the OpenVINO `model.embed()` finding and all three benchmark
   results), `docs/cycle_18_identity_association_root_cause_investigation.md`,
   `docs/cycle_18_redis_boundary_state_cache_investigation.md`.
4. `docs/production_inference_benchmark.md`, `docs/inference_parallelization_plan.md`.
5. Boundary code: `backend/apps/video_analysis/services/offline_sharding.py`
   (`_descriptor_mode`, `_region_hsv_descriptor`, `_track_appearance_reference`,
   `_write_boundary_appearance_feature`, `_combined_cost_assignment`,
   `_cosine_similarity_score`, `_parent_appearance_vectors`,
   `build_cycle18_boundary_packet`, `_apply_packet_association`, `_build_track_map`).
6. `backend/apps/tracking/embeddings.py` (`extract_crop_embedding`,
   `normalize_vector`, `_coerce_embedding_dimension`, `EMBEDDING_DIMENSION=768`)
   and the embedding stage + boundary call in `backend/apps/video_analysis/tasks.py`.
7. Existing Triton serving: the detector and RTMPose gRPC clients under
   `backend/apps/pipeline/`, and a WORKING `config.pbtxt` in the production model
   repo `backend/models/triton_repository_cuda12/`. Mirror these patterns exactly.
8. Live path (so you don't break it): the RTSP/live entity docs
   `docs/entity/systems/live_streaming_pipeline.md`,
   `docs/entity/systems/camera_streaming_bridge.md`, and the live Triton profile
   (ports `39000/39001/39002`).

================================================================================
## 1. NON-NEGOTIABLE PRODUCTION CONSTRAINTS (verify on the host; never assume)
================================================================================
- Native Linux, NO Docker, NO sudo. CUDA 12.8. Triton binary pinned. TensorRT
  runtime **10.16.1.11, serialization tag 240** — every engine MUST match or
  Triton load fails. Source of truth: `tools/prod/prod_trt_guard.sh` +
  `backend/models/tensorrt_builds/latest_compat.json`; rebuild via
  `tools/prod/prod-rebuild-tensorrt-engines.sh`.
- Triton has **no `python` backend** in the pinned build (verify the loaded backend
  list at startup). The ReID model MUST be the **tensorrt** backend (ONNX→TRT
  engine). Do NOT design a Python/BLS ensemble unless you first prove a python
  backend exists; if absent, fail closed to TensorRT.
- Reach prod via `tools/prod/prod-ssh.ps1` (alias `prod-grad`),
  `prod-copy-to.ps1`, `prod-copy-from.ps1`, `prod-workers.ps1`. Repo
  `/home/bamby/grad_project`; venv `/home/bamby/grad_project/backend/.venv/bin/python`;
  offline Triton `39100/39101/39102`; live Triton `39000/39001/39002`; backend
  `127.0.0.1:8011`. PostgreSQL only (port in `backend/.env`); SQLite forbidden
  everywhere.
- Start Triton only via `bash -l /home/bamby/grad_project/tools/prod/prod_start_triton.sh`.
  Confirm every required model `READY` before benchmarking. Exactly ONE endpoint
  profile is active at a time (§2.2) — never start both.

================================================================================
## 2. MODEL CHOICE — USE OSNet-AIN x1.0 (justified; do not substitute lightly)
================================================================================
Primary: **`osnet_ain_x1_0`** from Torchreid (KaiyangZhou/deep-person-reid, MIT
license — VERIFY the exact checkpoint's license/source and record digest).
Rationale for THIS architecture:
- Tiny (~2.2M params, ~9 MB engine) and fast → fits both the offline throughput
  budget AND a future live per-frame budget on the RTX 5090; negligible VRAM next
  to detector+RTMPose on 32 GB.
- 512-d embedding, clean ONNX export, proven ONNX→TensorRT path.
- The **AIN (domain-generalization)** variant is chosen deliberately: our weights
  are pretrained on public ReID datasets and applied to a classroom domain
  (uniforms, fixed camera). AIN generalizes to unseen domains far better than a
  vanilla Market1501-only model — critical because we will have little/no in-domain
  training data initially.
Fallback only if AIN weights are unavailable/incompatible: `osnet_x1_0`. If you
believe a stronger model is warranted (e.g. FastReID SBS-R50, 2048-d), you MAY
propose it in your plan WITH a latency/VRAM/parity justification, but default to
OSNet-AIN and do not switch without recording why.

### EXACT preprocessing contract (a single mistake here silently destroys ReID)
- Input crop from the person bbox, clamped to frame bounds; **skip degenerate or
  tiny crops** (e.g. < 16 px on a side) and mark `quality_state` accordingly —
  never embed a 5 px crop.
- OpenCV reads **BGR**; OSNet expects **RGB** → convert BGR→RGB. (Color-order bugs
  are the #1 silent ReID failure — assert this in a test.)
- Resize to **256×128 (H×W)** (OSNet standard), bilinear.
- Scale to [0,1], then normalize with ImageNet stats: mean `[0.485,0.456,0.406]`,
  std `[0.229,0.224,0.225]`, channel order RGB. Layout **NCHW**.
- Output 512-d float embedding → **L2-normalize** → compare with cosine. Keep the
  native 512-d in the boundary packet (schema allows `dimension ≤ 4096`); do NOT
  bucket-average it to 768 (that destroys it). Only the persisted `FrameEmbedding`
  table enforces 768 (§17) — see §4 OPTIONAL routing caveat.
- TensorRT: dynamic batch (min=1, opt=16, max=64 — tune to crop counts). Build
  **FP32 first**; enable FP16 ONLY if a decoded cosine-parity check vs FP32 passes
  (prior cycles had FP16 parity failures — verify, don't assume). Add a Triton
  **warmup** sample in `config.pbtxt` so first call isn't a cold-start spike.

================================================================================
## 3. OFFLINE vs ONLINE (RTSP) — STRICT SEPARATION (this is mandatory, §8.6)
================================================================================
The Triton ReID model is shared infrastructure, but its USE differs by profile.
Do NOT conflate them. Do NOT enable any offline-only lever on live.

### 3a. OFFLINE video processing (THIS cycle's target; where acceptance happens)
- Whole-file access; sharding ON; boundary packets ON; cross-shard `combined_cost`
  association. ReID embeddings build the boundary appearance prototype and (option)
  the per-track ReID feature.
- Throughput-first: batch many crops per Triton call; dynamic batching with a queue
  delay is fine. Unbounded buffering / backward access is acceptable here.
- This path is `offline-only` and MUST stay disabled in every live profile.
- Acceptance is the two-shard `combined.mp4` benchmark (see §5/§6).

### 3b. ONLINE RTSP / live streaming (DO NOT BREAK; do not enable sharding here)
- Continuous stream, NO "whole file" → **sharding, boundary packets, and
  cross-shard merge are meaningless and FORBIDDEN on live** (§8.6.1).
- If (and only if) you add live ReID, it is a SEPARATE, separately-governed,
  default-OFF feature for *within-stream* continuity (occlusion/re-entry recovery),
  and it MUST honor the live invariants (§8.6.2):
  1. frame-drop is signal, not defect; 2. bounded per-camera queue (overflow drops
  oldest, never blocks the decoder); 3. NO retroactive mutation of persisted live
  evidence (corrections append-only with lineage); 4. latency budget is **per
  frame**, not per window — ReID must not push the RTSP state machine to `degraded`;
  5. **process-local / per-camera-bounded state, never per-job**; 6. fail-stop on
  persistence failure; 7. no reliance on file seekability; 8. reconnection must not
  fabricate continuity (report the temporal gap).
- Triton profile placement: the offline profile (`39100…`) and live profile
  (`39000…`) load from their OWN model configurations. For THIS cycle, add the ReID
  model to the **offline** profile only. If you also want it on live LATER, it needs
  its own entry in the live profile with **latency-tuned** dynamic batching (small/no
  queue delay, smaller max batch) — that is a distinct future change with its own
  live-profile benchmark, NOT part of this acceptance.
- In code: gate the ReID-boundary descriptor behind the existing offline sharding
  flags so it can never activate on the live inference path. Update
  `tools/prod/prod_enable_parallel_flow.sh` so the offline block enables the ReID
  descriptor knob and the **live block explicitly disables** it.
- Add at least one test asserting the live profile/path does NOT construct boundary
  packets or call the offline ReID boundary descriptor.

================================================================================
## 4. DELIVERABLES (all required)
================================================================================
1. ReID artifacts: pinned ONNX export recipe + reproducible TRT engine build script
   (records weights digest, ONNX opset, TRT version), `config.pbtxt`, and a
   `latest_compat.json` update. Engine in the OFFLINE Triton repo only.
2. Backend ReID Triton gRPC client (mirror detector/RTMPose clients): preprocess per
   §2 contract, batch, call Triton, L2-normalize, **fail closed** (no local-model
   fallback in prod, §2.1) → on Triton error return explicit "unavailable" so the
   producer writes `appearance_reference.state="unavailable"` (never a zero vector).
3. New descriptor mode `triton_reid` in
   `OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_APPEARANCE_DESCRIPTOR`
   (`backend/config/settings/base.py`), consumed in `_track_appearance_reference`.
   Keep `region_hsv`/`cv2`/`backbone` intact; default unchanged until the benchmark
   decides. Provenance: `model_name="triton_osnet_ain_x1_0"`, real `model_version`,
   native `dimension=512`, engine digest; honor packet `max_bytes`/`dimension`.
   OPTIONAL, SEPARATE decision: route the main embedding stage through the same
   client — but persisted `FrameEmbedding` is the 768-d §17 contract, and a 512-d
   ReID vector must NOT be bucket-averaged to 768 (degrades it). If you route it,
   store ReID at native dim in a governed separate field/contract, not by coercion.
   Do not bundle this into the boundary-association acceptance.
4. TEST SCRIPTS (mandatory):
   - ReID client unit tests with a mocked Triton stub: preprocess shape/order
     (assert BGR→RGB + 256×128 + ImageNet norm), L2-norm output, fail-closed on
     error, min-crop-size gate.
   - A PyTorch-vs-Triton **parity probe** under `tools/prod/`
     (`prod_probe_reid_triton.py`): runs identical crops through the reference
     PyTorch model and the Triton engine, asserts cosine ≈ 1.0 (FP32) within
     tolerance; records latency, determinism (fixed seed), digests; JSON+MD; no DB
     writes; plus a focused unit test. Keep CI-read JSON visible (§18 `.gitignore`).
   - Extend `backend/tests/unit/video_analysis/test_cycle15b1_shard_merge.py`:
     `triton_reid` mode yields an `available` reference with correct provenance/dim
     when the client returns a vector, and `unavailable` (not zero) on error.
   - Live-isolation test (per §3b).
   - Run the AGENTS.md "Production Startup/Validation Checklist" focused suites green.
5. Workflow/CI updates IF needed (engine-build or probe steps): update
   `.github/workflows/*.yml`; ensure new Mermaid compiles
   (`scripts/ci/verify_mermaid_diagrams.py`); ensure doc-date + reading-order +
   one-command Cycle 18 contract gate (`tools/prod/prod_check_cycle18_boundary_contract.py`)
   pass. Never add a blanket gitignore that hides a CI-required file (§18).
6. Benchmark + figure evidence: name a Figure Planner AND a Figure Implementer
   (v2.11.0). Run `tools/prod/prod_run_cycle15b1_two_shard_runtime_benchmark.sh`
   with `--track-map-mode combined_cost --boundary-packet-enabled 1
   --boundary-packet-appearance-enabled 1 --context-frames 256 --baseline-metrics
   /home/bamby/grad_project/backend/logs/cycle15b-pre-shard-baseline-20260603T193531Z/metrics.json`
   plus env selecting `triton_reid` and `--figure-*` args. Generate figures via
   `tools/prod/prod_generate_cycle_figures.py` (use its stats/distribution/gate-matrix
   support). Baseline job is `74561b05-105f-4ca8-aeaf-f510f4f802de`.
7. Rollback proof: one env/profile change restores `best_iou` + sharding off +
   packet off; ReID model may stay loaded but unused; `rollback_status.json` = true.
8. Docs: a per-entity doc for the ReID Triton model under `docs/entity/` (follow
   `docs/per_entity_doc_template.md`; all source-of-truth references resolve), and a
   benchmark-result + §12.6 decision section appended to the Cycle 18.D doc, plus the
   AGENTS turn record and coordination-board update.

================================================================================
## 5. ACCEPTANCE CRITERIA (§12.5/§12.6 — only a completed RTX 5090 run decides)
================================================================================
Versus the accepted pre-shard baseline:
- Packets `valid 2/2`, `merge_ready 2/2`; shard-1 unresolved → near 0; StudentTracks
  back toward 53.
- Min model-agreement F1@IoU0.5 recovered to threshold; min shard-1 global-assignment
  F1 materially above the current `79.876 %`.
- DB parity, FPS/Step-2/RTT/GPU/memory deltas, figure bundle+manifest, rollback verified.
- HONEST §4.6 CAVEAT (do not skip): label-invariant global-assignment F1 is a PROXY,
  not HOTA/IDF1, and CANNOT alone authorize scientific identity acceptance. The best
  state you may claim without human-labeled ground truth is
  `ACCEPTED_PROXY_PENDING_GROUND_TRUTH`. For a true accept, stand up a TrackEval
  HOTA/AssA + IDF1 + ID-switch + fragmentation evaluation on a small labeled set
  (offer to scope it; do not fabricate labels).

================================================================================
## 6. GUARDRAILS (any violation is a CI-blocking regression)
================================================================================
- ReID inference runs in Triton; no local/mock inference presented as production (§2.1).
- Strict offline/live separation (§3, §8.6). Offline-only levers never ship to live.
- Never invent a number, path, digest, latency, or "READY" status; every doc claim
  resolves to a real artifact (§19). Missing metric → `unavailable` + reason, never 0.
- Do not rerun a previously failed profile as new evidence; `triton_reid` is distinct,
  so its first completed benchmark is the only valid decision point.
- PostgreSQL only; honor the §17 fixed-dim contract at the DB write boundary.
- Branch from the active feature branch; commit only what you changed; end commits
  with the required Co-Authored-By line; push only after the focused suites and the
  Cycle 18 contract gate pass.

================================================================================
## 7. EXECUTION ORDER (post a bounded plan BEFORE writing code)
================================================================================
1. §0/§1 verification: SSH OK; offline Triton `ready=200`; TRT guard green; print the
   loaded Triton backend list (prove tensorrt-only); confirm baseline metrics + video.
2. Acquire OSNet-AIN weights (verify license/digest) → ONNX → FP32 TRT engine pinned
   → add Triton model dir + `config.pbtxt` (+ warmup) → load → confirm READY → run the
   PyTorch-vs-Triton parity probe (cosine ≈ 1.0). Gate FP16 on a passing parity check.
3. Build the ReID Triton client + unit tests (mocked + parity).
4. Add `triton_reid` descriptor + producer integration + live-isolation test; run the
   contract gate + shard suite green locally.
5. Commit/push; deploy reviewed SHA to prod (fast-forward); restart workers; reconfirm
   model READY.
6. Run the governed two-shard benchmark; collect metrics/packet-validation/
   model-agreement/label-invariant/rollback/figures.
7. Record the honest §12.6 decision (accept / proxy-pending-GT / not-accepted), update
   the Cycle 18.D doc + entity doc + AGENTS turn record + board, verify rollback,
   release the lock.

================================================================================
## 8. FILE MAP
================================================================================
- `backend/models/triton_repository_cuda12/osnet_ain_x1_0/{config.pbtxt,1/model.plan}`
- `backend/models/tensorrt_builds/latest_compat.json` (engine-set change)
- `backend/apps/pipeline/<reid_triton_client>.py` (new) + tests
- `backend/apps/video_analysis/services/offline_sharding.py` (`triton_reid` mode)
- `backend/apps/video_analysis/tasks.py` / `backend/apps/tracking/embeddings.py`
  (OPTIONAL main-stage routing — separate decision, native-dim, no 768 coercion)
- `backend/config/settings/base.py` (flags) + `tools/prod/prod_enable_parallel_flow.sh`
  (offline enables, live disables)
- `tools/prod/prod_probe_reid_triton.py` + reproducible engine-build script + tests
- `backend/tests/unit/...` (client, producer, parity, live-isolation)
- `.github/workflows/*.yml` (only if new CI steps are required)
- `docs/entity/<kind>/osnet_ain_x1_0.md`, `docs/cycle_18d_*` (result + decision)

Begin with §0/§1 verification and post your bounded plan. Treat every READY/number/
path/latency as something you must PROVE before you write it down.

  download the required model , export it into Tensort (one the production server ) , write a full docs for this new model
  do not duplicate code or logic , any logic or code this model can replace use it to remove the deuplicated functionality.
  write all the steps you do in markdown file so any agent can track your progress and complete beyond ur last step