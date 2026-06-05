# Agents Execution Guide

**Last updated:** 2026-06-05

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
- Active feature plan: [specs/013-human-pose-kinematics/plan.md](specs/013-human-pose-kinematics/plan.md)
- 2026-06-05 Cycle 013 Human Pose Kinematics decision: **INITIAL PRODUCTION
  ENABLEMENT EXCEPTION for the offline-proven path only**. Offline
  `combined.mp4` matrix evidence, enabled retry evidence, and disabled-layer
  rollback proof are recorded in
  [docs/production_inference_benchmark.md](docs/production_inference_benchmark.md)
  §43 and
  [docs/entity/cycles/cycle_013_human_pose_kinematics.md](docs/entity/cycles/cycle_013_human_pose_kinematics.md).
  The production default is `POSE_KINEMATICS_ENABLED=1` by operator exception.
  This is not full scientific/live acceptance: a governed
  `reviewer_label_manifest.json` / label-agreement report for SC-003 through
  SC-006 and real-media live validation evidence for SC-007 remain open
  reminders and must be completed before final production acceptance. The open
  evidence tasks are T071, T073, and T075 in
  [specs/013-human-pose-kinematics/tasks.md](specs/013-human-pose-kinematics/tasks.md).
<!-- SPECKIT END -->

## ⭐ Benchmark Figure Evidence — BINDING

Constitution v2.11.0 extends Sections 7.1.1, 12.6, and 14.25: no optimization
cycle decision has authority unless generated figures are created from the same
raw artifacts as the benchmark decision table and embedded or linked from the
connected Markdown evidence.

Before implementation or benchmark work on any optimization cycle, the agent
MUST read:

1. this `AGENTS.md`;
2. [`.specify/memory/constitution.md`](.specify/memory/constitution.md);
3. the cycle's investigation/results Markdown file;
4. the current benchmark history in
   [`docs/production_inference_benchmark.md`](docs/production_inference_benchmark.md);
5. the current sorted queue in
   [`docs/inference_parallelization_plan.md`](docs/inference_parallelization_plan.md)
   and [`docs/cycle_9_and_10_improvements_todo.md`](docs/cycle_9_and_10_improvements_todo.md).

For every cycle state (`ACCEPTED`, `NOT ACCEPTED`, `NEEDS FURTHER ITERATION`,
`PROBE_ONLY`, `HYPOTHESIS_ONLY`, `STAGED`, or rollback), the evidence package
MUST include:

- baseline-vs-candidate-vs-best-cycle metric deltas when comparable;
- FPS, latency, throughput, per-model RTT/call-rate, per-step/per-phase wall,
  GPU/VRAM, CPU/worker memory, DB/Redis, correctness/model agreement, identity,
  reconciliation, rollback, and unavailable-metric reasons;
- generated plots plus a manifest that records input artifact paths and digests;
- Markdown embeds or links in the responsible cycle/result/benchmark docs;
- workflow/CI updates when code or evidence-generation scripts are changed.

Figures are not a substitute for raw JSON/CSV/log evidence. A missing metric
must render as `unavailable` with a reason and must not be hidden, omitted, or
plotted as zero. A cycle remains open until implementation, local tests,
workflow validation, production Linux RTX 5090 benchmark, metric deltas, figure
generation, documentation updates, and rollback proof are all recorded.

### Figure evidence role split

Every optimization cycle MUST name exactly one **Figure Planner** and exactly
one **Figure Implementer** before any benchmark-decision claim is made.

- The Figure Planner defines the plots required for the cycle, the raw
  artifacts each plot consumes, the unavailable-metric policy, and the Markdown
  embed targets.
- The Figure Implementer owns the Python/Bash generator changes, tests,
  manifests, digests, and produced images.
- The two roles SHOULD be different agents when multiple agents are active. If
  one agent must temporarily perform both roles, the cycle doc MUST explain why
  separation was unavailable and must still keep the plan and implementation
  evidence separate.
- No benchmark, metric delta, acceptance, rejection, closure, or "needs further
  iteration" decision has authority until the Figure Planner output and Figure
  Implementer output are both linked from the cycle/result Markdown.

For Cycle 18.C, Agent 19 owns the cycle kickoff and packet-budget /
association-readiness plan. The Figure Implementer lane must be explicitly
claimed in the coordination board before changing the figure generator so it
does not overlap Agent 17 or Agent 18 work.

## ⭐ Documentation Systematization Program (DSP) — BINDING

**Constitution Section 19 (added in v2.5.0; current constitution v2.11.0)
makes the following rules non-bypassable. Hallucinations are FORBIDDEN — a
documentation claim without a resolvable reference is a CI-blocking
regression, not a style nit.**

### Read these first before any doc change
1. [`docs/documentation_systematization_plan.md`](docs/documentation_systematization_plan.md) — the master multi-cycle plan
2. [`docs/per_entity_doc_template.md`](docs/per_entity_doc_template.md) — the template every entity doc MUST follow
3. [`docs/mermaid_theme_contract.md`](docs/mermaid_theme_contract.md) — the theme + palette + label rules
4. [`.specify/memory/constitution.md`](.specify/memory/constitution.md) Section 19 — the constitutional text

### The 8 non-negotiables (mirror of constitution § 19)
1. **Reading-order authority.** `README.md` § "Documentation Reading Order" wins over any individual file date.
2. **Per-entity coverage.** Every System / Module / Phase / Script / Code / API entity MUST have exactly one entity doc under `docs/entity/<kind>/`.
3. **Mermaid theme contract.** Every Mermaid diagram declares the matching theme initializer from `docs/mermaid_theme_contract.md` § 2.
3a. **Mermaid compilation gate (constitution § 19.3.1).** Every Mermaid block in every project-owned `.md` file MUST compile to SVG via `mmdc`. `.github/workflows/mermaid-diagrams.yml` runs `scripts/ci/verify_mermaid_diagrams.py` on every push / PR touching a `.md` file. Compilation failure is CI-blocking; the failure JSON identifies file + line + parser error. To check locally before pushing: `npm install -g @mermaid-js/mermaid-cli@^11.4.0 && python scripts/ci/verify_mermaid_diagrams.py --paths <doc-or-dir>`.
4. **Text-fitting.** Node labels ≤ 40 chars per logical line; multi-line uses `<br/>` or `\n`.
5. **Diagram preservation.** When a diagram is updated, the old version is PRESERVED — never delete. Add an H3 `historical` section.
6. **Source-of-truth references.** Every entity doc has a `## Source-of-truth references` table; every File / Symbol / Commit / Job / Workflow / Doc row MUST resolve in the working tree. Unsourced claims are hallucinations.
7. **Cycle discipline.** DSP Cycles 2-7 ship one entity per commit. A single commit MUST NOT advance two DSP cycles.
8. **Diagram-deletion check.** `git log` walked by the CI gate fails on any net-removed Mermaid block without a sibling `historical` block.

### The DSP cycle map
| DSP Cycle | What lands | Status |
|---|---|---|
| 0 | Plan + governance hardening (this commit) | ACTIVE |
| 1 | Inventory + reading-order refresh for ALL `.md` | PLANNED |
| 2 | System-level entity docs | PLANNED |
| 3 | Module-level entity docs (per Django app + frontend package) | PLANNED |
| 4 | Phase / optimization-cycle docs | PLANNED |
| 5 | Script + service + helper entity docs | PLANNED |
| 6 | Code-level entity docs (hot files only) | PLANNED |
| 7 | API surface entity docs | PLANNED |
| 8 | CI gate proving coverage + theme + deletion compliance | PLANNED |

### What to do if you cannot back a claim with a real reference
**Delete the claim.** Do not invent a reference. Do not soften the
claim to evade verification. Document the open question in the entity
doc's § 16 "Open questions" with an owner and a target close date.
Hallucination is the precedent that justified Section 19; do not be
the next case.

### Forbidden patterns (CI gate fails on these)
- "Approximately N lines of code" without a `<file>` citation
- "Around X ms" without an `inference_audit` / bench summary citation
- "The function does Y" without a `<file>:<line>` citation
- Quoted text from a doc that does not exist at the cited path
- A Mermaid block missing `%%{init:`
- A Mermaid block that `mmdc` cannot compile to SVG (the mermaid-diagrams workflow blocks the PR; check locally with `python scripts/ci/verify_mermaid_diagrams.py`)
- A node label > 40 chars with no `<br/>` / `\n` break
- A commit that nets-deletes a Mermaid block without a sibling `historical` version
- A commit that advances two DSP cycles at once

## ⭐ Streaming-Source Compatibility — BINDING

**Constitution Section 8.6 (added in v2.8.0) makes the following rules
non-bypassable. Shipping an offline-only optimization to the live profile
is a CI-blocking regression per § 14.25 row "Offline-only optimization
shipped to live", not a tuning oversight.**

### Why this exists

The repo supports two input source classes that share the inference plane:

| Source class | Examples | Owning entity doc |
|---|---|---|
| **Offline files** | uploaded `.mp4`, `runtime_ingest_video`, archive replays | [`docs/entity/systems/offline_inference_pipeline.md`](docs/entity/systems/offline_inference_pipeline.md) |
| **Live streams** | RTSP, RTSPS, WHEP/WebRTC (bridged via `go2rtc` + `gst-mediamtx`), HLS-as-fallback only | [`docs/entity/systems/live_streaming_pipeline.md`](docs/entity/systems/live_streaming_pipeline.md) + [`docs/entity/systems/camera_streaming_bridge.md`](docs/entity/systems/camera_streaming_bridge.md) |

Several ACCEPTED offline optimizations rely on offline-only invariants
(whole-video access, unbounded buffering, backward seek, post-decode
mutation, input-pixel reduction without retraining). Enabling them on a
continuous live stream collapses the system: unbounded memory growth,
missed frames after backward seeks, false-positive amplification an
operator cannot retroactively suppress, latency-budget violations that
trip the RTSP state machine into `degraded`.

### Forbidden on the live profile (offline-only levers)

Per constitution § 8.6.1. Each row cites the originating cycle entity doc:

| Forbidden lever | Cycle |
|---|---|
| Whole-job true-batch concat memoisation | [Cycle 5 in cycles_1_to_5_bundle](docs/entity/cycles/cycles_1_to_5_bundle.md) |
| Per-job process-local `track_vector_cache` reuse | [Cycle 8 lever 1](docs/entity/cycles/cycle_8_embedding_stage.md) |
| Lazy `cv2.VideoCapture.grab()` forward-skip | [Cycle 8 lever 2](docs/entity/cycles/cycle_8_embedding_stage.md) |
| `persist_embeddings_bulk` with row-count-only flush (no time bound) | [Cycle 8 lever 3](docs/entity/cycles/cycle_8_embedding_stage.md) — live needs a bounded-time variant |
| LPM Phase 1 post-decode box suppression | [Cycle 10](docs/entity/cycles/cycle_10_lpm_phase1.md) (already NOT-ACCEPTED everywhere) |
| Behavior input-size shrink without retraining | [Cycle 11.A](docs/entity/cycles/cycle_11_input_size.md) (already NOT-ACCEPTED everywhere) |
| Larger batch windows that wait N frames | `docs/cycle_9b_batch_window_investigation.md` (Cycle 9b B.4) |
| Pose-tail batching that waits for a frame group | `docs/cycle_14a_pose_tail_decomposition_investigation.md` |
| Video sharding / parent-shard coordination | `docs/cycle_15b_video_sharding_design_proof_investigation.md` |

### Stream-safe levers (the recommended live-throughput knobs)

| Stream-safe lever | Cycle |
|---|---|
| Per-frame pose chunking to `max_batch_size=16` | [Cycle 6](docs/entity/cycles/cycle_6_pose_chunking.md) |
| Process-local cached Redis client | [Cycle 7](docs/entity/cycles/cycle_7_redis_client_cache.md) |
| Triton ensembles + server-side slice + Top-K adapters | [Cycle 9b B.2.b](docs/entity/cycles/cycle_9b_b2b_exact_slice.md) + [Cycle 9b B.2.c](docs/entity/cycles/cycle_9b_b2c_topk.md) |
| Dual-sink JSON-first telemetry writer | [`apps.telemetry`](docs/entity/modules/apps.telemetry.md) |

### What every new optimization-cycle entity doc MUST declare

Per § 8.6.3, every new file under `docs/entity/cycles/` MUST carry a
`Streaming compatibility:` field in its Section 1 with one of:

- `stream-safe` — no cross-frame / whole-job / backward-seek dependency
- `stream-safe-with-config` — safe IF a stated env knob / bound is set for live; cite the knob + value
- `offline-only` — MUST be enumerated in the offline block of `tools/prod/prod_enable_parallel_flow.sh` AND explicitly disabled in the live block
- `requires-stream-investigation` — defaults OFF in live until decided

A new cycle doc without this field will fail the (DSP Cycle 8) verifier
once the gate ships. Until then, pre-amendment cycle docs landed under
DSP Cycle 4 SHOULD be backfilled opportunistically; missing field on a
pre-amendment doc is a documentation gap, not a constitutional violation.

### Pre-commit checklist for any cycle that touches inference or persistence

1. Does the change introduce per-job state (dict, buffer, cache) keyed on
   `job_id` or `session_id`? If yes, decide its eviction policy and verify
   it survives an indefinite live stream (no unbounded growth).
2. Does the change call `cv2.VideoCapture.set(...)` or any backward seek?
   If yes, gate it off in live mode.
3. Does the change wait for N frames / N seconds before dispatch? If yes,
   declare and respect the live SLO budget; disable in live if the budget
   cannot be met.
4. Does the change mutate persisted detection / identity rows after the
   fact (LPM-style suppression, smoothing)? If yes, it MUST NOT enable on
   live; corrections MUST be append-only with lineage.
5. Does the change rely on the "whole file" being available? If yes
   (sharding, end-to-end memo), it is `offline-only`.
6. Update `tools/prod/prod_enable_parallel_flow.sh`: add the env knob to
   the offline profile block AND explicitly disable it in the live
   profile block.
7. Set the `Streaming compatibility:` field on the cycle entity doc.

### Live-profile invariants every change MUST preserve

Per § 8.6.2:

1. Frame-drop is signal, not defect (drop counters per camera per interval)
2. Bounded per-camera queue (overflow drops oldest; never blocks decoder)
3. No retroactive mutation of persisted live evidence (corrections are
   append-only)
4. Latency budget is per-frame, not per-window
5. Process-local or per-camera-bounded state — never per-job
6. Fail-stop on persistence failure (camera → `degraded` per § 8.4)
7. No reliance on file-system seekability
8. Reconnection MUST NOT fabricate continuity (report the temporal gap)

## ⭐ Identity Association And Cross-Run Evaluation — BINDING

**Constitution Section 4.6 (added in v2.9.0) makes local tracker labels
source-scoped opaque values. Raw local-ID equality is not identity proof across
independent tracker runs, shards, jobs, reconnects, replays, or runtime
activations.**

### Required evaluation contract

1. Match observations with a documented localization rule before comparing
   identity association.
2. Map independent label namespaces with a deterministic globally one-to-one
   assignment. Do not use raw string or numeric local-ID equality as the
   identity correctness gate.
3. Report detection/localization quality separately from association quality,
   fragmentation, merge collisions, unresolved associations, and eligible /
   matched / unmatched boundary tracklets.
4. Label a single-run baseline as a reproducibility proxy, not human-labeled
   identity ground truth.
5. Ground-truth-backed acceptance reports HOTA/AssA, IDF1, ID switches, and
   fragmentation. Proxy-only metrics may guide investigation but cannot alone
   accept or close identity work.

### Required cross-shard association contract

- Operate on bounded source-scoped tracklets, not one boundary box or a local
  label number.
- Carry documented temporal/motion, governed appearance, lifecycle, quality,
  feature/model provenance, candidate cost, threshold, and ambiguity evidence.
- Make globally one-to-one decisions and keep ambiguous, conflicting, missing,
  stale, or below-threshold cases `unresolved`; never force a canonical merge.
- Keep sharding and boundary-state transport offline-only. The live profile
  explicitly disables them.

Cycle 18 applies this doctrine through
`docs/cycle_18_identity_association_root_cause_investigation.md`,
`tools/prod/prod_probe_cycle18_label_invariant_tracking.py`, and
`docs/architecture/cycle18_label_invariant_tracking_probe.production_summary.json`.
The production read-only probe confirmed that raw-label comparison understated
agreement, but it also measured a real residual association gap. The later
Agent 20 override production benchmark rejected the bounded
`OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=one_to_one` candidate. The follow-up
default-off boundary packet producer is production-validated as evidence-only;
Agent 19 then staged governed appearance evidence plus a parent-side
`appearance_packet` association consumer locally. That candidate still needs a
full production benchmark and rollback proof before it can be accepted,
rejected, or used to advance sharding.

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
  (`-5.17 %`) and DB-completed FPS `4.439 → 4.471` (`+0.71 %`), but failed
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
- **2026-06-02 Cycle 9b B.1 compact postprocessing investigation OPEN**:
  `docs/cycle_9b_compact_postproc_investigation.md` records that the accepted
  exact-slice + Top-K baseline has already reduced behavior output traffic from
  the old `~17.1 MB/frame` dense-output problem to `~0.33 MB/frame`. B.1 is
  therefore still a valid candidate, but it must be measured as a remaining
  output-transfer plus Python decode/NMS lever, not as the original dense-byte
  removal. Pinned production Triton backends currently list
  `distributed_addsub`, `dyna_sequence`, `implicit_state`, `iterative_sequence`,
  `query`, `sequence`, and `tensorrt`; there is no `python` backend in the
  pinned runtime, so B.1.a Python BLS is blocked until a controlled Triton
  rebuild/switch is benchmarked. The first B.1 artifact is measurement-only:
  `tools/prod/prod_probe_behavior_decode_cost.py` samples accepted Top-K
  `person_detection` crops, calls `behavior_ensemble_gaze_slice_topk`, and
  records gRPC parse plus Python `_decode_yolo_output0`/NMS cost. Production
  probe `backend/logs/cycle9b_b1_decode_cost_topk_20260602T185559Z.json` sampled
  `340` real crops: mean RTT with parse `62.082 ms`, mean gRPC/Triton wait
  `59.651 ms`, mean `as_numpy` parse `0.114 ms`, and mean Python decode/NMS
  `3.125 ms` per 17-crop batch (`0.183823 ms/crop`). With `3597` accepted
  baseline behavior calls, decode/NMS is about `11.24 s` or `~2.08 %` of the
  accepted `540.399 s` Step 2 wall. The dominant measured time is still
  gRPC/Triton wait plus server execution (`59.651 ms` of the `62.082 ms`
  RTT-with-parse sample), so the next B.1 implementation must reduce that
  remaining wait or model-side work, not only move Python Top-K decode/NMS.
  No compact backend is implemented, rejected, skipped, or accepted; only a
  full production benchmark can make that decision. The required end-to-end
  repeat was run with the reproducible wrapper
  `tools/prod/prod_run_b1_decode_cost_full_benchmark.sh`: replay
  `cycle9b-b1-fullbench-20260602T192344Z`, job
  `00e0e1da-44b6-4198-ad39-39fd853e4e18`, deployed SHA `7556e84`, video
  `combined.mp4`, status `completed`. This run kept the accepted 320
  exact-slice + Top-K route and implemented no compact backend, so it records
  `NO_DECISION_BENCHMARK_RECORDED`, not acceptance or non-acceptance. Final
  comparison vs accepted Top-K: Step 2 wall `540.399 s → 540.748 s`
  (`+0.06 %`), DB-completed FPS `4.439 → 4.346` (`-2.11 %`), behavior RTT mean
  `84.865 ms → 85.201 ms` (`+0.40 %`), GPU avg `9.344 % → 11.962 %`, DB rows
  within `0.02 %`, tracks unchanged `53`, and model-agreement F1@IoU0.5 stayed
  high (`attention_tracking=99.724 %`, `hand_raising=99.790 %`,
  `person_detection=100.000 %`, `sitting_standing=99.979 %`). The fresh
  all-crop decode probe sampled `19,146` crops / `1,127` batches from this
  completed job: RTT `45.160 ms`, infer wait `42.704 ms`, decode/NMS
  `2.040 ms/batch` (`0.120 ms/crop`), output `367,603,200` bytes vs estimated
  compact `222,936` bytes. Evidence lives under
  `backend/logs/cycle9b-b1-fullbench-20260602T192344Z/`.
  The required second repeat also completed:
  `cycle9b-b1-fullbench-repeat2-20260602T195517Z`, job
  `df7f832f-de81-4f92-89c7-fd213bdba7fa`, status `completed`, wrapper status
  `NO_DECISION_BENCHMARK_RECORDED`. It again kept the accepted route. Repeat-2
  Step 2 wall was `546.702 s`, DB FPS `4.411`, behavior RTT mean
  `84.360 ms`, GPU avg `9.660 %`, rows stayed within `0.01 %`, tracks stayed
  `53`, and model-agreement F1@IoU0.5 stayed `>=99.707 %`. Repeat-2 decode
  evidence measured `47.154 ms` RTT, `44.162 ms` infer wait, and
  `4.243 ms/batch` decode/NMS over `19,146` crops. Watcher script
  `tools/prod/prod_watch_benchmark_metrics.sh` now renders bounded,
  wrapping tables and includes metrics, model agreement, decode cost,
  artifact presence, DB counters, GPU, audit, RTT, and wrapper-log tail.
- **2026-06-02 Cycle 12 Persistent Async Dispatcher STAGED / Phase A active**:
  the cycle map is restaged so the next inference-wall work measures
  single-process orchestration before implementing another compact-output or
  render/persistence change. Source doc:
  `docs/cycle_12_persistent_dispatcher_investigation.md`. Local instrumentation
  is staged: `TRITON_ASYNC_DISPATCH_PROFILING` records
  `async_runner.run(...)` boundary wall in
  `backend/apps/video_analysis/tasks.py`, and
  `tools/prod/prod_collect_benchmark_metrics.py` plus
  `tools/prod/prod_watch_benchmark_metrics.sh` surface it. Reproducible wrapper:
  `tools/prod/prod_run_async_dispatch_profile_benchmark.sh`. Clean production
  replay `cycle12-async-dispatch-profile-clean-20260602T213441Z` / job
  `dfa1f138-7086-418a-ba17-9999cd12b9ac` completed at deployed SHA `3d2c8e8a`.
  It measured `349.643 s` async-dispatch blocking wall; `behavior_all` owned
  `338.779 s`. No optimization candidate was deployed, so this remains
  `HYPOTHESIS_ONLY`: no Cycle 12 optimization is accepted, rejected, skipped,
  or closed yet. The next candidate must overlap behavior wait/server execution;
  replacing only the sync async-bridge is probably below the `>=10 %` Step 2
  acceptance gate. 2026-06-03 metric decision: the bridge-only dispatcher was
  not selected for implementation; Cycle 12.B is the bounded behavior-wait
  overlap candidate recorded in
  `docs/cycle_12_overlap_dispatcher_investigation.md`. It is implemented only
  behind `TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=1`, with reproducible benchmark
  wrapper `tools/prod/prod_run_behavior_overlap_benchmark.sh`. This is still
  staged work only. No acceptance/rejection/skip/closure decision exists until
  a completed `combined.mp4` production Linux RTX 5090 benchmark writes the
  before/after table, model-agreement rows, DB parity, RTT/GPU/memory evidence,
  and rollback proof.
- **2026-06-03 Cycle 12.B behavior-wait overlap NEEDS FURTHER ITERATION**:
  production benchmark `cycle12-behavior-overlap-20260602T223350Z`, job
  `46ba8b2a-3c61-4d89-b7b6-63ec72159428`, deployed SHA `d1f5e9b7`, completed
  `4541/4541` frames on `combined.mp4`. It improved Step 2 wall
  `540.399 s → 395.495 s` (`-26.81 %`), DB FPS `4.439 → 5.195`
  (`+17.03 %`), DB elapsed `1022.952 s → 874.104 s` (`-14.55 %`), and GPU avg
  `9.344 % → 11.274 %`. Correctness stayed within gate: tracks `53 → 53`, DB
  rows within `0.02 %`, model-agreement F1 `>=99.716 %`. It is **not
  accepted** because behavior RTT regressed: mean `84.865 ms → 115.420 ms`
  (`+36.00 %`) and p95 `128.056 ms → 224.661 ms` (`+75.44 %`). Root cause to
  address next: the current overlap submits the new `behavior_all` request
  before finalizing the pending one, creating brief two-behavior-job Triton
  contention. Production rollback proof: `TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=0`
  and `TRITON_ASYNC_DISPATCH_PROFILING=0`, accepted 320/slice/Top-K route
  remains active. Next task is Cycle 12.C single-inflight behavior overlap.
- **2026-06-03 Cycle 12.C single-inflight behavior overlap STAGED**:
  investigation doc `docs/cycle_12_single_inflight_overlap_investigation.md`
  exists before code. The implementation keeps the same rollback flag
  `TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=1` but changes ordering so the current
  crop payloads are built first, the previous pending behavior job is finalized,
  and only then the current `behavior_all` request is submitted. This preserves
  overlap of previous behavior execution with current person/crop preparation
  while avoiding two behavior jobs in flight. No decision exists until a real
  production `combined.mp4` benchmark records FPS, Step 2 wall, RTT, GPU,
  memory, DB parity, model agreement, and rollback proof.
- **2026-06-03 Cycle 12.C single-inflight behavior overlap ACCEPTED**:
  production benchmark `cycle12-single-inflight-overlap-20260602T225821Z`, job
  `069a217f-fa43-48cc-bf18-c946d53bb3ee`, deployed SHA `f31ff39b`, completed
  `4541/4541` frames. It improved Step 2 wall `540.399 s → 459.461 s`
  (`-14.98 %`), DB FPS `4.439 → 4.854` (`+9.35 %`), DB elapsed
  `1022.952 s → 935.516 s` (`-8.55 %`), and GPU avg `9.344 % → 10.332 %`.
  Behavior RTT gate passed: mean `84.865 ms → 83.936 ms` (`-1.09 %`), p95
  `128.056 ms → 130.200 ms` (`+1.67 %`). Correctness passed: tracks
  `53 → 53`, rows within `0.02 %`, model-agreement F1 `>=99.716 %`.
  `TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=1` is now part of the accepted optimized
  production profile; rollback is setting it to `0` and restarting Celery
  workers. Evidence: `docs/production_inference_benchmark.md` §26 and
  `docs/cycle_12_single_inflight_overlap_results.md`.
- **2026-06-03 Redis broader optimization investigation STAGED**:
  `docs/redis_broader_optimization_opportunities.md` records broader Redis
  strategies that could improve benchmark metrics after the current cycle
  sequence: command-cost instrumentation, embedding/tracking pipeline
  coalescing, Redis Streams for non-authoritative progress/benchmark sampling,
  boundary-state cache for future video sharding, and conditional Redis
  scripts for measured read/compute/write hotspots. This is roadmap evidence
  only. Cycle 7 remains the cautionary precedent: Redis client caching helped
  only modestly, so Cycle 16.A must measure Redis command count, Redis wall,
  bytes, memory, and errors on production before any Redis implementation is
  accepted, rejected, skipped, or closed. PostgreSQL remains the authoritative
  store for job status, rows, and benchmark evidence.
- **2026-06-03 Cycle 13.B prefetch-aware embedding track lookup ACCEPTED**:
  `docs/cycle_13_persistence_render_investigation.md` first recorded the
  accepted Cycle 12.C post-stage decomposition from production evidence:
  Step 3 persistence `39.820 s`, render `25.692 s`, post-run-complete tail
  `189.323 s`, and PostgreSQL `FrameEmbedding.created_at` span `187.139 s`
  across `72,578` rows. Measurement-only profiling then ran on production:
  replay `cycle13-embedding-profile-20260603T003853Z`, job
  `aa2fe7a9-b3fb-49d7-92a3-eca41c894dcd`, status `completed`,
  `4541/4541` frames. Evidence:
  `backend/logs/cycle13-embedding-profile-20260603T003853Z/embedding_profile_metrics.json`
  and `docs/cycle_13_embedding_profile_results.md`. The run preserved DB rows
  and model-agreement exactly (`100.000 %` F1@IoU0.5 for all persisted
  models) but was measurement-only: DB FPS `4.854005 -> 4.802395` and elapsed
  `935.516 s -> 945.570 s`. The embedding profile measured total wall
  `188.620 s`, with track lookup `66.223 s`, Redis flush `59.304 s`, DB flush
  `38.467 s`, and existing-embedding checks `14.527 s`. The Cycle 13.B
  production benchmark then ran: replay `cycle13-track-lookup-20260603T011324Z`,
  job `c9f75d55-6043-4f27-bf9e-b2826d299459`, deployed SHA `9b872a3`,
  status `completed`, `4541/4541` frames. Evidence:
  `backend/logs/cycle13-track-lookup-20260603T011324Z/track_lookup_metrics.json`,
  `backend/logs/cycle13-track-lookup-20260603T011324Z/model_agreement_cycle12c_vs_track_lookup.json`,
  and `docs/cycle_13_embedding_track_lookup_results.md`. Results versus Cycle
  12.C: DB elapsed `935.516 s -> 872.317 s` (`-6.756 %`), DB FPS
  `4.854005 -> 5.205675` (`+7.245 %`), embedding profile wall
  `188.620 s -> 121.681 s` (`-35.489 %` vs Cycle 13.A), track lookup
  `66.223 s -> 0.447 s` (`-99.325 %`), GPU avg `10.332 % -> 11.003 %`,
  DB rows unchanged, StudentTracks `53 -> 53`, and model-agreement F1
  `100.000 %` for all persisted models. Profile proof recorded
  `track_lookup_prefetch_hits=144785`, `track_lookup_prefetch_misses=0`, and
  `track_lookup_orm_fallbacks=0`. `EMBEDDING_PREFETCH_TRACK_LOOKUP=1` is now
  part of the optimized production profile; rollback is setting it to `0` and
  restarting Celery. Next bottleneck: Redis flush `59.874 s` and DB flush
  `38.773 s`; because Cycle 7 overestimated Redis savings, the next
  Redis-related step must first measure command count/wall before changing
  Redis semantics. That next step completed as Cycle 13.C / 16.A measurement:
  replay `cycle13c-redis-command-profile-20260603T020723Z`, job
  `aa246a4e-e0f9-471a-9ce3-74f343bbd1fb`, deployed SHA `bea98cb9a0bc`,
  status `completed`, exact DB/model parity. Results: DB FPS
  `5.205675 -> 5.024795` (`-3.47 %` because profiling only), Step 2 wall
  `458.696 s -> 458.532 s`, behavior RTT mean `86.545 ms -> 86.203 ms`,
  embedding wall `121.681 s -> 152.771 s`, Redis flush
  `59.874 s -> 92.397 s`, DB flush `38.773 s -> 37.348 s`.
  Redis decomposition recorded `870936` estimated helper commands, `72578`
  estimated pipeline executes, `1017733` Redis server calls, but only
  `530.485 ms` Redis server command wall. Conclusion:
  `docs/cycle_13c_redis_db_side_effect_measurement_results.md` is
  **MEASUREMENT COMPLETE / HYPOTHESIS_ONLY**; Redis server tuning is not the
  first fix. Cycle 16.B then implemented guarded Redis side-effect coalescing:
  production replay `cycle16b-redis-coalescing-20260603T025823Z`, job
  `b2dfa987-afc5-4b96-ab12-6799b149ac25`, deployed SHA `c458c443`, status
  `completed`, `4541/4541` frames. Results versus Cycle 13.B: DB elapsed
  `872.317 s -> 849.136 s` (`-2.66 %`), DB FPS `5.205675 -> 5.347791`
  (`+2.73 %`), embedding profile wall `121.681 s -> 97.505 s` (`-19.87 %`),
  Redis flush `59.874 s -> 35.970 s` (`-39.92 %`), behavior RTT mean
  `86.545 ms -> 86.532 ms`, exact DB rows and `100.000 %` model agreement for
  all persisted models. Redis pipeline executes fell from the Cycle 13.C
  measured `72578` shape to `146`; coalesced Redis errors and unavailable count
  were `0`. `EMBEDDING_REDIS_SIDE_EFFECT_COALESCING=1` is now part of the
  accepted optimized production profile; rollback is setting it to `0` and
  restarting Celery. Evidence:
  `docs/cycle_16b_redis_side_effect_coalescing_results.md` and
  `docs/production_inference_benchmark.md` §31. Next measured bottlenecks are
  ordered by measured wall: Step 2 through-pose tail over frame wall
  `221.777 s`, embedding DB flush `37.737 s`, and Redis payload serialization
  `31.242 s`.
- **2026-06-03 Cycle 14.A pose-tail decomposition MEASUREMENT COMPLETE /
  HYPOTHESIS_ONLY**: production replay
  `cycle14a-pose-tail-profile-20260603T135129Z`, job
  `862a13db-a2ae-408f-a737-ee9aeca45f5c`, completed `4541/4541` frames on
  `combined.mp4`. This was profiling only, so no optimization is accepted,
  rejected, skipped, or closed. Evidence:
  `docs/cycle_14a_pose_tail_decomposition_results.md` and
  `docs/production_inference_benchmark.md` §32. The tail remained material:
  Step 2 through-pose upload `690.164 s`, Step 2 frame wall `465.856 s`,
  tail `224.308 s`, and post-frame-loop drain `200.778 s`. The drain is
  dominated by RTMPose runtime/provider work: runtime wall `190.176 s`,
  provider batch wall `186.090 s`, provider async batch wall `154.171 s`, and
  provider payload build `17.327 s`. DB/model parity was exact and all
  model-agreement F1 rows were `100.000 %`. Next cycle must target RTMPose
  provider overlap or cross-frame RTMPose batching behind a rollback flag before
  compact BLS/TRT postprocessing, Redis Streams/scripts, streaming persistence,
  video sharding, or worker-count increases. Rollback proof from the profiling
  run: `POSE_TAIL_PROFILING=0`, `EMBEDDING_STAGE_PROFILING=0`, accepted Cycle
  16.B runtime flags remain enabled.
- **2026-06-03 Cycle 14.B split into scenario sub-cycles STAGED**:
  `docs/cycle_14b1_rtmpose_single_inflight_overlap_investigation.md` and
  `docs/cycle_14b2_rtmpose_cross_frame_batching_investigation.md` exist before
  acceptance code/benchmark decisions. Cycle 14.B1 uses
  `POSE_TAIL_OPTIMIZATION_MODE=overlap` to benchmark one-inflight RTMPose
  overlap. Cycle 14.B2 uses
  `POSE_TAIL_OPTIMIZATION_MODE=cross_frame_batch` and
  `POSE_CROSS_FRAME_BATCH_SIZE=16` to benchmark ordered cross-frame RTMPose
  batching. Both must run separate production Linux RTX 5090 `combined.mp4`
  benchmarks through `tools/prod/prod_run_cycle14b_pose_tail_scenario_benchmark.sh`,
  then be compared directly against each other and against the Cycle 16.B /
  Cycle 14.A evidence. No acceptance/rejection/skip/closure decision exists
  until those benchmark tables, DB/model parity, RTT/GPU/memory evidence, and
  rollback proof are written.
- **2026-06-03 Cycle 14.B decision recorded**:
  `docs/cycle_14b_rtmpose_scenario_results.md` and
  `docs/production_inference_benchmark.md` §33 record the completed production
  scenario matrix. B1 overlap replay
  `cycle14b-overlap-20260603T143000Z` / job
  `b366807a-5e14-4a37-aade-a555ae85cdf0` is **NOT ACCEPTED** because FPS and
  pose tail were flat. B2 first run
  `cycle14b-cross-frame-batch16-20260603T144500Z` / job
  `2dfc1470-e01e-4d81-96cf-5f493fdf898f` is **REJECTED** because pose records
  dropped `19157 -> 15075`. B2 fixed rerun
  `cycle14b-cross-frame-batch16-r2-20260603T150000Z` / job
  `6b42a557-b954-4954-a2f8-de54634229eb` is **ACCEPTED**: DB FPS
  `5.347791 -> 5.680314`, DB elapsed `849.136 s -> 799.428 s`, pose tail
  `221.777 s -> 171.751 s`, RTMPose calls `5047 -> 1199`, GPU avg
  `11.030 % -> 12.168 %`, exact DB/model parity, and pose records
  `19157 -> 19180`. Production accepted profile now includes
  `POSE_TAIL_OPTIMIZATION_MODE=cross_frame_batch` and
  `POSE_CROSS_FRAME_BATCH_SIZE=16`; rollback is
  `POSE_TAIL_OPTIMIZATION_MODE=off` plus Celery worker restart.
- **2026-06-03 Cycle 14.C RTMPose batch-size matrix STAGED**:
  `docs/cycle_14c_pose_batch_size_matrix_investigation.md`,
  `docs/cycle_14c1_pose_batch_size_8_investigation.md`, and
  `docs/cycle_14c2_pose_batch_size_32_investigation.md` exist before any
  decision. The accepted Cycle 14.B2 batch `16` replay
  `cycle14b-cross-frame-batch16-r2-20260603T150000Z` is the baseline. Cycle
  14.C1 benchmarks batch `8`; Cycle 14.C2 benchmarks batch `32`. The helper
  `tools/prod/prod_run_cycle14c_pose_batch_size_matrix.sh` must run each
  scenario independently against the baseline and then compare the two
  scenarios directly. No acceptance/rejection/skip/closure decision exists
  until the completed production benchmark table, model agreement, DB parity,
  GPU/RTT/memory evidence, and rollback proof are written.
- **2026-06-03 Cycle 14.C decision recorded**:
  `docs/cycle_14c_pose_batch_size_matrix_results.md` and
  `docs/production_inference_benchmark.md` §34 record the completed production
  matrix. Cycle 14.C1 batch `8` replay
  `cycle14c-pose-batch-matrix-20260603T154945Z-batch8` / job
  `d3cc149f-9ad3-4dd6-8c14-beed80f7c0d2` is **NOT ACCEPTED**: DB FPS
  `5.680314 -> 5.518671`, Step 2 through-pose `633.939 s -> 655.592 s`,
  and provider chunks `1199 -> 2398`. Cycle 14.C2 batch `32` replay
  `cycle14c-pose-batch-matrix-20260603T154945Z-batch32` / job
  `7a0b5819-9af4-46b4-a5ee-c46cdd7c1d9c` is **NOT ACCEPTED**: GPU avg
  improved `12.168 % -> 14.288 %`, but DB FPS regressed
  `5.680314 -> 5.624745`, Step 2 through-pose regressed
  `633.939 s -> 643.184 s`, and RTMPose p95 regressed
  `46.952 ms -> 100.563 ms`. DB/model parity and pose records were preserved
  in both scenarios. Production remains on accepted batch `16`.
- **2026-06-03 Cycle 14.C3 batch-32 provider chunk parallelism STAGED**:
  `docs/cycle_14c3_batch32_parallel_chunks_investigation.md` records the
  batch-32-specific follow-up requested after Cycle 14.C2. The candidate keeps
  `POSE_CROSS_FRAME_BATCH_SIZE=32` but sets
  `POSE_PROVIDER_CHUNK_PARALLELISM=2` so the two internal 16-crop RTMPose
  provider chunks can run concurrently. Default `1` preserves the accepted
  batch-16 profile. Reproducible wrapper:
  `tools/prod/prod_run_cycle14c3_batch32_parallel_chunks_benchmark.sh`. No
  acceptance/rejection/skip/closure decision exists until the wrapper completes
  a real production `combined.mp4` benchmark and records FPS, Step 2
  through-pose, RTMPose p95, GPU, memory, DB parity, model agreement, and
  rollback proof against both accepted batch `16` and prior rejected batch `32`.
- **2026-06-03 Cycle 14.C3 decision recorded**:
  production replay `cycle14c3-batch32-parallel2-20260603T170117Z`, job
  `d71802b2-4c20-4f31-9c06-9854ebbc4eed`, deployed SHA `6642046`, completed
  `4541/4541` frames. Correctness was exact against both accepted batch `16`
  and prior rejected batch `32`, but performance failed the repair gate:
  DB FPS `5.680 -> 5.465` versus batch `16` (`-3.79 %`) and `5.625 -> 5.465`
  versus prior batch `32` (`-2.84 %`), Step 2 through-pose `633.939 s ->
  665.406 s` (`+4.96 %`) versus batch `16`, RTMPose p95 `100.563 ms ->
  188.211 ms` (`+87.16 %`) versus prior batch `32`, and GPU average
  `14.288 % -> 12.092 %` versus prior batch `32`. Cycle 14.C3 is **NOT
  ACCEPTED**. The wrapper restored production to accepted
  `POSE_CROSS_FRAME_BATCH_SIZE=16`, `POSE_PROVIDER_CHUNK_PARALLELISM=1`,
  `POSE_TAIL_PROFILING=0`, and `EMBEDDING_STAGE_PROFILING=0`. Evidence:
  `docs/cycle_14c3_batch32_parallel_chunks_results.md`.
- **2026-06-03 Cycle 14.D server-side compact postprocessing PHASE A
  COMPLETE / NO IMPLEMENTATION SELECTED**:
  `docs/cycle_14d_server_side_compact_postproc_results.md` records production
  measurement evidence from
  `/home/bamby/grad_project/backend/logs/cycle14d-phase-a-20260603T175051Z`.
  The wrapper made no runtime mutation. D1 Python BLS is blocked for code
  because the pinned production Triton runtime log did not expose Python backend
  support. D2 TensorRT/plugin compacting and D3 fused output were not selected
  because output compaction can reduce bytes but current parse/decode cost is
  only `3.357 ms/batch` (`0.197497 ms/crop`) after Top-K; infer wait and server
  work remain dominant. This is not a rejection, skip, closure, or acceptance of
  an implemented candidate. The next sorted cycle is Cycle 15.
- **2026-06-03 Cycle 15 CUDA shared memory vs video sharding PHASE A STARTED**:
  `docs/cycle_15_cuda_shared_memory_vs_sharding_investigation.md` splits the
  next architecture decision before code into 15.A CUDA shared memory and 15.B
  video sharding. No shared-memory or sharding implementation may start until
  Phase A measurements prove whether copy/serialization or insufficient
  independent GPU work in flight is the dominant lever. Any concurrency or
  worker/process increase remains governed by constitution §8.1.1 and requires
  full production `combined.mp4` benchmark evidence before acceptance.
- **2026-06-03 Cycle 15 Phase A MEASURED / 15.B DESIGN-PROOF NEXT**:
  `docs/cycle_15_cuda_shared_memory_vs_sharding_results.md` records read-only
  production evidence from
  `/home/bamby/grad_project/backend/logs/cycle15-phase-a-20260603T180125Z-r2`
  at deployed SHA `a69b431`. CUDA shared memory is not selected for immediate
  implementation: behavior serialize `2.766 ms`, deserialize `2.296 ms`, and
  `as_numpy` `0.115 ms` are small against infer wait `49.950 ms` and Step 2
  through-pose `633.939294 s`. Video sharding is the next design-proof
  sub-cycle, not an implementation yet; it must prove deterministic shard
  overlap, track stitching, duplicate suppression, PostgreSQL idempotency,
  terminal-state coordination, and rollback before code.
- **2026-06-03 Cycle 15.B video sharding design proof PHASE A STARTED**:
  `docs/cycle_15b_video_sharding_design_proof_investigation.md` opens the
  sharding sub-cycle without authorizing implementation. The first allowed
  scenario is 15.B1 two-shard offline inference because Phase A estimated
  `0.704691 %` duplicate overlap frames. Four-shard or larger variants remain
  blocked until two-shard stitching, duplicate suppression, DB idempotency, and
  parent/shard terminal-state coordination are proven.
- **2026-06-03 Cycle 15.B1/B2 read-only design helper STAGED**:
  `tools/prod/prod_plan_video_shards.py` and
  `tools/prod/prod_run_cycle15b_shard_design_probe.sh` compute deterministic
  shard intervals for 15.B1 two-shard and 15.B2 four-shard design proof. They
  do not submit jobs, change env, restart services, or benchmark runtime speed.
  Both scenarios are `offline-only`; per constitution §8.6 they must not be
  enabled on RTSP, RTSPS, WHEP/WebRTC, HLS fallback, or any live stream profile.
- **2026-06-03 Cycle 15.B design proof PASSED / runtime still blocked**:
  `docs/cycle_15b_shard_design_probe_results.md` records production dry-run
  evidence from
  `/home/bamby/grad_project/backend/logs/cycle15b-shard-design-20260603T191500Z`
  at deployed SHA `f55fb85`. 15.B1 two-shard and 15.B2 four-shard both have
  exact authoritative frame coverage. 15.B1 has `32` context-only frames
  (`0.704691 %` overhead); 15.B2 has `96` (`2.114072 %`). Runtime sharding is
  still blocked until stitching, duplicate suppression, PostgreSQL idempotency,
  parent/shard terminal-state coordination, and rollback are implemented.
- **2026-06-03 Cycle 15.B pre-shard baseline benchmark RECORDED**:
  production replay `cycle15b-pre-shard-baseline-20260603T193531Z` / job
  `74561b05-105f-4ca8-aeaf-f510f4f802de` completed `4541/4541` frames on the
  accepted non-sharded `crop_frame` profile at runtime deployed SHA `74afc9d`.
  Evidence is stored in
  `/home/bamby/grad_project/backend/logs/cycle15b-pre-shard-baseline-20260603T193531Z/metrics.json`
  and `.md`. This is `NO_DECISION_BASELINE_ONLY`, not a sharding acceptance:
  DB FPS `5.620`, Step 2 frame wall `467.450 s`, Step 2 through-pose wall
  `641.154 s`, behavior RTT mean `83.530 ms`, GPU average `11.846 %`, GPU
  peak `57.000 %`, detection rows `72744`, bbox rows `72744`, embeddings
  `72578`, tracks `53`. Future 15.B1 runtime must beat these metrics while
  preserving DB/model parity.
- **2026-06-03 Cycle 15.B1 two-shard runtime PHASE A STARTED / BLOCKED**:
  `docs/cycle_15b1_two_shard_runtime_investigation.md` defines the parent/shard
  runtime contract and `tools/prod/prod_check_cycle15b1_runtime_readiness.py`
  provides a read-only production blocker audit. Production evidence:
  `/home/bamby/grad_project/backend/logs/cycle15b1-runtime-readiness-20260603T200611Z`.
  Result: `blocked_no_runtime_candidate`, `8` critical blockers, `0` warnings.
  The env is safe because `OFFLINE_VIDEO_SHARDING_ENABLED` is absent/off. A
  valid runtime benchmark is blocked until env/profile management,
  authoritative frame-window support, parent/shard lineage, a runtime wrapper,
  a parent merge helper, and detection/bbox/embedding merge idempotency exist.
- **2026-06-03 Cycle 15.B1 safe-default slice STAGED**:
  disabled sharding knobs are now declared in settings/profile management,
  `tools/prod/prod_set_cycle15b1_sharding_defaults.sh` sets only those keys,
  and `process_video_upload` fails closed with
  `cycle15b1_sharding_runtime_not_implemented` if a sharding flag or shard
  metadata appears before the runtime candidate exists. Focused unit tests
  passed and the local readiness audit now reports
  `blocked_no_runtime_candidate` with `5` critical blockers. Production
  deployed SHA `74631e6`, ran the focused guard/planner tests (`5 passed`), set
  `OFFLINE_VIDEO_SHARDING_ENABLED=0`, restarted Celery workers, preserved the
  accepted Top-K/overlap profile, and wrote readiness evidence to
  `/home/bamby/grad_project/backend/logs/cycle15b1-runtime-readiness-safe-default-20260603T202308Z`.
  This is safety and reproducibility work only: no two-shard runtime benchmark
  has started, and no acceptance/rejection/skip/closure decision exists.
- **2026-06-03 Cycle 15.B1 runtime candidate IMPLEMENTED / BENCHMARKED /
  NOT ACCEPTED**: `backend/apps/video_analysis/services/offline_sharding.py`
  now defines the offline-only shard contract, pre-roll-only two-shard planner,
  authoritative-frame filter, boundary summary, and idempotent parent merge.
  `process_video_upload` now accepts only valid explicit child-shard metadata,
  decodes that 0-based half-open window via `decode_frame_window`, filters
  context-only frames before Step 3, writes child shard provenance, and stops
  child jobs before render/embedding. Parent orchestration lives in
  `backend/apps/video_analysis/management/commands/cycle15b1_sharded_ingest.py`;
  reproducible helpers are `tools/prod/prod_merge_cycle15b1_shards.py` and
  `tools/prod/prod_run_cycle15b1_two_shard_runtime_benchmark.sh`. Schema
  migration `0014_cycle15b1_shard_provenance.py` adds detection/bbox/embedding
  provenance. Local evidence: Python compile passed, `bash -n` passed, Django
  `manage.py check` passed, `makemigrations --check --dry-run` reported no
  changes, targeted tests passed (`4 passed`), and
  `backend/logs/cycle15b1_readiness_local.json` reports
  `ready_for_runtime_benchmark=True` with `0` critical blockers. Production
  replay `cycle15b1-two-shard-runtime-repeat-20260603T211319Z` / parent job
  `e602a0ca-6efc-4cb0-8d30-9466fe76287b` ran on `combined.mp4` with child jobs
  `f74de22a-0a37-4546-8029-a88264d55bad` and
  `8d49bd97-072b-4ebf-bf17-510eb820b6a6`. The performance hypothesis was
  proven: DB FPS `5.620 → 7.867` (`+39.98 %`), DB elapsed
  `808.038 s → 577.232 s` (`-28.56 %`), sharded Step 2 critical-path frame
  wall `467.450 s → 233.038 s` (`-50.15 %`), through-pose wall
  `641.154 s → 324.763 s` (`-49.35 %`), GPU average `11.846 % → 17.495 %`,
  and GPU peak `57 % → 89 %`. The candidate is **NOT ACCEPTED** because
  correctness failed: StudentTracks changed `53 → 52`, behavior RTT regressed
  `83.530 ms → 89.718 ms`, and baseline-agreement F1@IoU0.5 collapsed to
  `59.473 %` (`attention_tracking`), `60.700 %` (`hand_raising`),
  `61.387 %` (`person_detection`), and `53.455 %` (`sitting_standing`).
  Evidence:
  `/home/bamby/grad_project/backend/logs/cycle15b1-two-shard-runtime-repeat-20260603T211319Z/metrics.json`,
  `.md`, `model_agreement.json`, and `model_agreement.md`. Production rollback
  proof: `OFFLINE_VIDEO_SHARDING_ENABLED=0`, `OFFLINE_VIDEO_SHARD_COUNT=1`, and
  `OFFLINE_VIDEO_SHARD_CONTEXT_FRAMES=32`. Next work, if sharding remains in
  the queue, must target shard-boundary identity/track continuity before any
  four-shard or larger benchmark.
- **2026-06-04 Cycle 15.B1.C boundary identity stitching PROBE_ONLY / STARTED**:
  `tools/prod/prod_analyze_cycle15b1_stitching.py` now separates geometry-only
  agreement from track-sensitive agreement for the rejected two-shard parent.
  Production probe evidence:
  `/home/bamby/grad_project/backend/logs/cycle15b1c-stitching-probe-20260603T220206Z/stitching_probe.json`
  and `.md`, comparing baseline job `74561b05-105f-4ca8-aeaf-f510f4f802de`
  to sharded parent `e602a0ca-6efc-4cb0-8d30-9466fe76287b`. Result: shard 0
  stays aligned (`100 %` track F1 for attention/hand/posture boxes and
  `99.613 %` for person_detection), but shard 1 has near-perfect geometry
  (`100 %` geometry F1 for all four models) and broken track labels
  (`4.043 %` attention, `2.974 %` hand, `21.308 %` person, `4.124 %`
  sitting/standing track F1). This proves the next candidate must target
  canonical identity stitching, not box geometry. This probe has no decision
  authority; sharding remains disabled and 15.B2 remains blocked.
- **2026-06-04 Cycle 15.B1.C deep stitching probe PROBE_ONLY**: helper commit
  `3baa4cdc` adds oracle relabeling and candidate ranking to the same
  read-only helper. Production evidence:
  `/home/bamby/grad_project/backend/logs/cycle15b1c-deep-stitching-20260603T221605Z/deep_stitching_probe.json`
  and `.md`. Shard-1 oracle relabeling improves track F1 but does not restore
  parity by itself: attention `4.043 % -> 75.458 %`, hand `2.974 % ->
  72.531 %`, person `21.308 % -> 56.733 %`, and sitting/standing
  `4.124 % -> 84.021 %`. Source-parent majority match is only `3.030 %` -
  `28.125 %`, so the current 32-frame IoU-only map is not a valid acceptance
  path. **Next executable subcycle:** run 15.B1.C1 as a full production
  `combined.mp4` context-window benchmark with `--context-frames 256`. If that
  fails model agreement, implement 15.B1.C2 as a guarded canonicalizer with
  ambiguity gates; do not advance to 15.B2 or unrelated cycles first.
- **2026-06-04 Cycle 15.B1.C1 context-256 NOT ACCEPTED / Cycle 15.B1.C2
  STAGED**: production benchmark
  `cycle15b1c1-context256-20260603T222123Z`, parent job
  `401498f1-d5e4-4b95-8a46-ad3fcbbc2c25`, completed `4541/4541` frames on
  `combined.mp4`. Context `256` improved DB FPS `5.620 -> 7.905`, Step 2 frame
  wall `467.450 s -> 242.392 s`, and GPU avg `11.846 % -> 17.636 %`, but it
  failed correctness: `StudentTracks` stayed `52` vs baseline `53`, model
  agreement F1@IoU0.5 was only attention `58.997 %`, hand `61.109 %`, person
  `61.767 %`, and sitting/standing `53.730 %`, and behavior RTT regressed
  `83.530 ms -> 89.717 ms`. Decision: **NOT ACCEPTED**. The next current-cycle
  candidate is 15.B1.C2, a guarded majority-vote parent track canonicalizer
  behind `OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=majority_vote`; default remains
  `best_iou` and sharding remains disabled after wrapper cleanup. No C2
  acceptance/rejection exists until a full production benchmark completes and
  writes model-agreement evidence.
- **2026-06-04 Cycle 15.B1.C2 majority-vote canonicalizer NOT ACCEPTED**:
  production benchmark `cycle15b1c2-majority-vote-20260603T223932Z`, parent job
  `78388c2c-d7f5-42b7-afa4-321216d23b11`, completed `4541/4541` frames on
  `combined.mp4` at deployed SHA `ed3d766`. It kept context `256` and enabled
  `OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=majority_vote` with IoU `0.50`, min
  matches `3`, and purity `0.85`. Performance stayed improved versus the
  accepted pre-shard baseline (DB FPS `5.620 -> 7.833`, Step 2 frame wall
  `467.450 s -> 243.909 s`, GPU avg `11.846 % -> 17.891 %`), but correctness
  failed harder: shard 1 mapped only `10/36` tracks to existing parent IDs,
  `26/36` fell back to offset IDs, `StudentTracks` rose `53 -> 64`, and
  model-agreement F1@IoU0.5 stayed low (attention `58.997 %`, hand `61.109 %`,
  person `60.933 %`, sitting/standing `53.730 %`). Behavior RTT regressed
  `83.530 ms -> 90.372 ms`. Decision: **NOT ACCEPTED**. Keep majority-vote
  default-off for reproducibility; do not enable it in production. Cycle 15.B2
  remains blocked. Cycle 17 Redis Streams later completed and was accepted as
  observability-only; further sharding work needs a new identity-state design
  proof.
- **2026-06-04 Cycle 17 Redis Streams progress sampling ACCEPTED
  OBSERVABILITY-ONLY / TURN FREE**: dedicated investigation file
  `docs/cycle_17_redis_streams_progress_sampling_investigation.md` records the
  bounded, non-authoritative progress/evidence mirror. Agent 18 completed the
  production Linux RTX 5090 `combined.mp4` benchmark
  `cycle17-redis-streams-20260604T025328Z` and released the lock. PostgreSQL
  remains terminal-state authority; Redis Streams are capped and default-off.
  The cycle is accepted only as observability/evidence infrastructure, not as a
  throughput or Step 2 inference-wall improvement.
- **2026-06-04 Four-agent cycle coordination ACTIVE**:
  `docs/four_agent_cycle_coordination_board.md` is the shared board for the
  coordinated agent sessions. Current ownership: Agent 17 = historical Cycle 17
  handoff only (`FREE`), Agent 18 = Cycle 17 Redis Streams (`FREE`, accepted
  observability-only), Agent 19 = Cycle 18 measured contract/evidence audit
  (`BLOCKED`, no runtime authority; recovered per-track evidence remains
  blocked), Agent 20 override = Cycle 18 one-to-one candidate plus boundary
  packet producer (`FREE`, `ACCEPTED_EVIDENCE_ONLY_NOT_IDENTITY_MERGE_READY`,
  benchmark lock released), and Agent 20 = Cycle 20 streaming
  persistence/embedding overlap readiness plus Cycle 21 concurrency/benchmark
  governance (`TAKEN`, docs/readiness only). Shared
  roadmap files (`AGENTS.md`,
  `README.md`, `docs/INDEX.md`, `docs/inference_parallelization_plan.md`,
  `docs/production_inference_benchmark.md`,
  `docs/crop_frame_optimization_execution.md`, and
  `docs/cycle_9_and_10_improvements_todo.md`) are orchestrator-owned unless an
  explicit edit window is granted. 2026-06-04 override update: the user
  reassigned Cycle 18 to Agent 20 for a bounded runtime-candidate attempt.
  Agent 20 benchmarked `OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=one_to_one` on
  production replay `cycle18-one-to-one-trackmap-20260604T174231Z`; the
  candidate is `NOT_ACCEPTED`. Agent 20 then production-validated
  `OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_ENABLED=1` on replay
  `cycle18-boundary-packet-producer-20260604T181738Z`; the producer is accepted
  only as default-off evidence infrastructure, rollback is verified, and the
  lock is released.
  Only one production benchmark may hold the benchmark lock at a time, and
  every future decision still requires constitution §12.5/§12.6 evidence.
- **2026-06-04 Agent turn ledger rule ACTIVE**:
  every cycle turn must be explicitly claimed and released in this file. A turn
  claim must include the agent name, cycle, state (`TAKEN`, `FREE`, `BLOCKED`,
  or `BENCHMARK_LOCK_HELD`), owned files, benchmark-lock state, validations run,
  and next action. A future agent must not assume a turn is free while this file
  says `TAKEN`; it must either see `FREE` or explicitly record a user-authorized
  reassignment here before starting. **Current turn:** Agent 18 has released
  the Cycle 17 Redis Streams turn as `FREE` after the 2026-06-04 production
  benchmark. The
  default-off progress mirror, watcher/collector metrics, focused tests, and
  benchmark wrapper are implemented locally. Responsibility file:
  `docs/agent_18_cycle_17_turn.md`. Production benchmark lock is `NOT HELD`;
  replay `cycle17-redis-streams-20260604T025328Z` completed and rollback was
  verified.
  Validation state: focused Cycle 17 unit tests pass (`9 passed`), edited
  Python files compile, edited production shell scripts pass `bash -n`, the
  reading-order/date gate passes, the touched
  coordination Mermaid block renders, and `git diff --check` reports no
  whitespace errors. The Cycle 17 wrapper dry-run now ends with
  `DRY_RUN_NO_BENCHMARK_RECORDED`, so dry-run output cannot be confused with
  production evidence. Decision state is
  `ACCEPTED_OBSERVABILITY_ONLY_NOT_THROUGHPUT`: DB/model parity was exact and
  Redis Stream evidence was bounded, but DB FPS was neutral (`5.620 -> 5.611`),
  so no throughput gain is claimed. Next action: open a new governed turn before
  advancing another cycle. Later on 2026-06-04, Agent 20's Cycle 18 override
  benchmark also released its lock and rejected the `one_to_one` candidate; see
  the dedicated Agent 20 paragraph below for the current Cycle 18 runtime
  decision.

  BENCHMARK_RELEASE
  agent: Agent 18
  cycle: Cycle 17 Redis Streams progress sampling
  replay_key: cycle17-redis-streams-20260604T025328Z
  job_id: a7cf6fc2-23fb-4e17-beac-42343ba8d634
  status: completed
  metrics_json: /home/bamby/grad_project/backend/logs/cycle17-redis-streams-20260604T025328Z/redis_streams_metrics.json
  metrics_md: /home/bamby/grad_project/backend/logs/cycle17-redis-streams-20260604T025328Z/redis_streams_metrics.md
  model_agreement_json: /home/bamby/grad_project/backend/logs/cycle17-redis-streams-20260604T025328Z/model_agreement_baseline_vs_redis_streams.json
  model_agreement_md: /home/bamby/grad_project/backend/logs/cycle17-redis-streams-20260604T025328Z/model_agreement_baseline_vs_redis_streams.md
  rollback_verified: yes; BENCHMARK_REDIS_STREAM_EVENTS=0, workers restarted,
  Django setting False, Triton ready
  released_at_utc: 2026-06-04T03:13:00Z
- **2026-06-04 Agent 19 Cycle 18 measured evidence turn `BLOCKED`**:
  the user explicitly re-opened Cycle 18 and requested real measured benchmark
  values and comparisons. Agent 19 owns only Cycle 18 contract/evidence work.
  The requested comparisons and production read-only identity-candidate probe
  are complete. The final Agent 19 pre-override governed state is
  `NO_RUNTIME_CANDIDATE_SELECTED` /
  `NO_DECISION_PRODUCTION_BENCHMARK_REQUIRED`; constitutional closure remained
  blocked at that point because no Cycle 18 runtime optimization or §12.6
  production decision table existed. The Agent 19 benchmark lock was not held,
  no Agent 19 Cycle 18 runtime candidate was deployed, and the shared
  coordination board later recorded the user-authorized Agent 20 override.
  Responsibility file: `docs/agent_19_cycle_18_turn.md`. Owned files are that
  ledger, `docs/cycle_18_redis_boundary_state_cache_investigation.md`,
  `docs/architecture/cycle18_boundary_packet_v0.schema.json`,
  `docs/architecture/cycle18_boundary_packet_v0.example.json`,
  `docs/architecture/cycle18_boundary_packet_v0.historical_cycle15b1c2.json`,
  `docs/architecture/cycle18_boundary_packet_v0.historical_cycle15b1c2.source.json`,
  `tools/prod/prod_validate_cycle18_boundary_packet.py`,
  `tools/prod/prod_project_cycle18_boundary_packet.py`,
  `tools/prod/prod_audit_cycle18_recovered_evidence.py`,
  `tools/prod/prod_check_cycle18_boundary_contract.py`,
  `tools/prod/prod_check_cycle18_schema_validator_drift.py`,
  `tools/prod/prod_check_cycle18_doc_references.py`,
  `backend/tests/unit/pipeline/test_prod_validate_cycle18_boundary_packet.py`,
  `backend/tests/unit/pipeline/test_prod_project_cycle18_boundary_packet.py`,
  `backend/tests/unit/pipeline/test_prod_audit_cycle18_recovered_evidence.py`,
  `backend/tests/unit/pipeline/test_prod_check_cycle18_boundary_contract.py`,
  `backend/tests/unit/pipeline/test_prod_check_cycle18_schema_validator_drift.py`,
  `backend/tests/unit/pipeline/test_prod_check_cycle18_doc_references.py`,
  `tools/prod/prod_check_cycle19_redis_script_eligibility.py`,
  `backend/tests/unit/pipeline/test_prod_check_cycle19_redis_script_eligibility.py`,
  `tools/prod/prod_check_cycle18_measured_benchmark_values.py`,
  `backend/tests/unit/pipeline/test_prod_check_cycle18_measured_benchmark_values.py`,
  `tools/prod/prod_probe_cycle18_identity_candidates.py`,
  `backend/tests/unit/pipeline/test_prod_probe_cycle18_identity_candidates.py`,
  `docs/architecture/cycle18_identity_candidate_probe.production_summary.json`,
  and `.gitignore` only for the explicit CI visibility exceptions.
  Shared coordination and roadmap files remain orchestrator-owned unless an
  explicit edit window is granted. Production benchmark lock is `NOT HELD`;
  runtime code, production env changes, sharding enablement, 15.B2, and all
  Agent 17 or Agent 18 Cycle 17 files are out of scope. Validation
  state: reading-order verifier passed, the coordination
  board Mermaid rendered `1/1`, cited local paths resolved, and `git diff
  --check` reported no whitespace errors. Contract slice result: the failed
  Cycle 15 boundary audit, bounded V0 schema, synthetic unresolved example,
  standalone read-only validator, reproducible historical projector,
  recovered-evidence intake auditor, and one-command local contract gate are
  recorded without runtime changes. The focused Cycle 18/19 suites passed `39`
  tests. The optional external JSON Schema validator is unavailable in the
  current Python environments; manual validation and the structural drift gate
  pass. The
  example is contract-valid and correctly not identity-merge-ready. Constitution
  §18 CI visibility is preserved by explicit `.gitignore` exceptions for all
  six JSON contract/projection/evidence files. The retained-evidence Cycle 15.B1.C2 projection
  is now recorded and fails closed because raw per-track observations,
  lifecycle, appearance, candidates, collision counters, and the historical
  tracker fingerprint are unavailable locally; it also preserves the documented
  `26` offset fallbacks. The historical packet is reproducible from a strict
  local source manifest, and the one-command gate verifies contract results,
  tracked projection drift, source references, expected historical failures,
  recovered-evidence fail-closed behavior, and external schema conformance.
  The recovered-evidence auditor keeps `full_projection_ready` separate from
  `identity_merge_ready` and rejects invalid, partial, wrong-scope, or
  wrong-range packets. The structural drift audit aligns `12` object shapes
  and rejects all `92` deterministic missing/unknown-field mutations. The
  recovered-evidence CLI now supports `--require-full-projection`, which fails
  the aggregate-only historical path without implying identity-merge
  readiness. The one-command gate verifies all `28` Agent 19 inventory entries
  resolve and all six Cycle 18 JSON visibility exceptions are present. The
  historical audit emits machine-readable recovery requirements and currently
  has exactly two unmet projection checks: `packet_contract_valid` and
  `packet_track_count_match`. The reference checker verifies `157` references
  across the three owned docs plus only Agent 19's bounded `AGENTS.md` paragraph,
  and all `25` line citations resolve. Cycle 19's conditional audit is backed
  by `docs/cycle_16b_redis_side_effect_coalescing_results.md` and
  `docs/redis_broader_optimization_opportunities.md`: Redis server command wall
  is `194.039 ms`, while remaining DB flush and payload serialization are
  seconds-scale, and the source recommendation says not to tune Redis server
  execution next. Current Cycle 19 state is
  `CONDITIONAL_GATE_NOT_MET / HYPOTHESIS_ONLY_NO_RUNTIME_DECISION`.
  Lua/EVAL/EVALSHA, Redis runtime changes, production mutation, benchmark-lock
  acquisition, and optimization decision claims remain forbidden. The local
  eligibility audit passed all `11` source/evidence checks, the combined Cycle
  18/19 focused suite passed `39` tests, the Cycle 18 contract gate still
  passed, and the bounded reference checker resolved `157` references plus
  `25` line citations. Next action is to wait for later production evidence
  proving a measured read/compute/write hotspot that pipelining cannot remove.
  The re-opened Cycle 18 evidence slice now carries real completed upstream
  production measurements from the accepted pre-shard comparator plus 15.B1
  context-32, 15.B1.C1 context-256, and 15.B1.C2 majority vote. Exact values
  include DB FPS `5.619787 -> 7.866851 / 7.905 / 7.833`, StudentTracks
  `53 -> 52 / 52 / 64`, and the C2 boundary diagnostics `10/36` mapped with
  `26` offset fallbacks. The measured-value checker verifies exact parity with
  `docs/production_inference_benchmark.md` and explicitly reports
  `cycle18_runtime_benchmark_present=false`; it is integrated into the
  one-command Cycle 18 contract gate. Derived comparisons are also
  machine-verified: best carried DB FPS is `7.905` (`+40.66 %`), but that
  candidate's model-agreement F1 range is only `53.730 %` to `61.767 %`; C2
  mapping coverage is `10/36` (`27.78 %`) with `26/36` (`72.22 %`) offset
  fallback. The closure gate reports `closure_eligible=false`, state
  `CLOSURE_BLOCKED_NO_CYCLE18_RUNTIME_BENCHMARK`, and decision authority
  `NO_DECISION_PRODUCTION_BENCHMARK_REQUIRED`. The final production read-only
  identity-candidate probe found zero child embeddings, aliases, lifecycle
  events, canonical IDs, ReID scores, and ReID decisions. Its best
  implementable policy reached only `62.021 %` minimum all-model F1 and
  `19.978 %` minimum shard-1 F1; even the baseline-label oracle upper bound
  reached only `73.086 %` and `49.445 %`. The hashed evidence summary is
  `docs/architecture/cycle18_identity_candidate_probe.production_summary.json`.
  Next action requires a separately reviewed identity-feature/state producer
  before any further runtime candidate, implementation, production benchmark,
  or constitutional Cycle 18 closure decision can exist.
  The user-authorized deep root-cause continuation is complete in
  `docs/cycle_18_identity_association_root_cause_investigation.md`. Primary
  sources were applied through constitution §4.6, the binding identity
  association guidance above, and
  `tools/prod/prod_probe_cycle18_label_invariant_tracking.py`. The read-only
  production probe reused completed jobs and wrote hashed evidence summarized
  in
  `docs/architecture/cycle18_label_invariant_tracking_probe.production_summary.json`.
  It confirmed two separate root causes: shard-1 minimum raw-label F1 was
  `2.917 %`, while deterministic one-to-one global assignment recovered it to
  `79.876 %`; geometry remained `100.000 %`, but a real residual association
  gap of up to `20.124 pp` remained. The one-command Cycle 18 gate now resolves
  `28` artifacts and `6` visible JSON files, all `39` focused Cycle 18/19
  tests pass, the documentation reading-order gate passes, and `5/5` Mermaid
  blocks across the changed governed docs render. Agent 19's final state
  remains
  `NO_RUNTIME_CANDIDATE_SELECTED` /
  `NO_DECISION_PRODUCTION_BENCHMARK_REQUIRED` /
  `CLOSURE_BLOCKED_NO_CYCLE18_RUNTIME_BENCHMARK`; Agent 19 made no runtime
  mutation or production benchmark decision.
- **2026-06-04 Agent 20 remaining lanes `TAKEN`**:
  the user identified this session as the last agent and requested a ledger plus
  remaining task ownership. Agent 20 maps to the coordination board's Agent C
  and Agent D lanes: Cycle 20 streaming persistence/embedding overlap readiness
  and Cycle 21 Celery concurrency governance. Responsibility file:
  `docs/agent_20_remaining_lanes_turn.md`. Primary owned files are that ledger,
  `docs/cycle_20_streaming_persistence_embedding_overlap_investigation.md`, and
  `docs/cycle_21_celery_concurrency_scaling_investigation.md`; coordination-only
  edits may integrate the claim into the board and reading order. Production
  benchmark lock is `NOT HELD`; runtime code, lifecycle/persistence/embedding
  changes, worker-count changes, queue topology changes, GPU-cap changes,
  `backend/.env` edits, and production env changes are out of scope. Validation
  state is recorded in the Agent 20 ledger. Next action: keep Cycle 20/21 as
  readiness/governance-only until a separate implementation or benchmark-lock
  turn is explicitly opened under constitution §12.5/§12.6.
- **2026-06-04 Agent 20 Cycle 18 override `FREE / NOT_ACCEPTED`**:
  the user explicitly overrode the prior Cycle 18 blocked handoff and assigned
  this session to take Cycle 18 forward. Responsibility file:
  `docs/agent_20_cycle_18_override_turn.md`. Agent 20 staged and production
  benchmarked a bounded, disabled-by-default runtime candidate in
  `backend/apps/video_analysis/services/offline_sharding.py`:
  `OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=one_to_one`. The candidate uses existing
  child/parent boundary votes and a deterministic maximum-vote one-to-one
  assignment before offset fallback; focused coverage is in
  `backend/tests/unit/video_analysis/test_cycle15b1_shard_merge.py`. This is
  not Redis identity authority and not live-profile behavior. Production
  replay `cycle18-one-to-one-trackmap-20260604T174231Z`, parent job
  `aa5f5328-ae81-49da-a44b-ae308859035b`, completed `4541/4541` frames. The
  candidate improved DB FPS `5.619787 -> 7.913961` and Step 2 frame wall
  `467.449833 s -> 241.830808 s`, but failed identity/model agreement:
  minimum raw F1 was `53.788 %`, minimum label-invariant all-model F1 was
  `71.744 %`, shard-1 global-assignment F1 stayed `79.876 %`, and behavior RTT
  regressed. Decision: `NOT_ACCEPTED`. Production rollback is verified with
  sharding disabled and track-map mode restored to `best_iou`; benchmark lock
  is `NOT_HELD`. Evidence is summarized in
  `docs/architecture/cycle18_one_to_one_trackmap.production_summary.json` and
  `docs/production_inference_benchmark.md` section 41. Next action: do not
  rerun or enable `one_to_one`; future Cycle 18 work needs governed appearance
  evidence and a parent-side association consumer before another merge-ready
  runtime candidate.
- **2026-06-04 Agent 20 Cycle 18 boundary packet producer
  `FREE / ACCEPTED_EVIDENCE_ONLY_NOT_IDENTITY_MERGE_READY`**: continuation
  after the `one_to_one` rejection stages
  a default-off boundary packet producer behind
  `OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_ENABLED=0`. Owned implementation files
  are `backend/apps/video_analysis/services/offline_sharding.py`,
  `backend/apps/video_analysis/tasks.py`,
  `backend/tests/unit/video_analysis/test_cycle15b1_shard_merge.py`,
  `backend/config/settings/base.py`, and
  `tools/prod/prod_run_cycle15b1_two_shard_runtime_benchmark.sh`. Production
  replay `cycle18-boundary-packet-producer-20260604T181738Z`, parent job
  `2cf0b4b3-5e81-41c4-9eda-9e7faa97c224`, completed `4541/4541` frames with
  rollback verified. The producer emitted `2/2` contract-valid packets, but
  `0/2` were identity-merge-ready; raw F1 minimum was `53.730 %`, label-
  invariant all-model minimum was `69.163 %`, and shard-1 label-invariant
  minimum was `71.884 %`. This accepts only the default-off packet evidence
  producer, not sharding, not 15.B2, and not identity merge. Benchmark lock is
  `NOT_HELD`. Evidence is summarized in
  `docs/architecture/cycle18_boundary_packet_producer.production_summary.json`
  and `docs/production_inference_benchmark.md` section 42.
- **2026-06-05 Agent 19 Cycle 18 appearance-packet continuation
  `STAGED_LOCAL_ONLY / NO_PRODUCTION_DECISION`**: after the Agent 20 packet
  producer left `appearance_reference=unavailable` and `0/2` packets
  merge-ready, Agent 19 staged a default-off blocker-resolution candidate.
  `backend/apps/pipeline/multi_model.py` carries the preview frame path,
  `backend/apps/video_analysis/tasks.py` passes an appearance artifact root
  only when `OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_APPEARANCE_ENABLED=1`, and
  `backend/apps/video_analysis/services/offline_sharding.py` writes
  digest-backed crop-descriptor references plus a separate
  `OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=appearance_packet` consumer. The
  consumer verifies referenced feature files, combines boundary geometry with
  appearance score/margin gates, writes candidate decisions back into the
  packet, and leaves missing or ambiguous evidence unresolved. Defaults remain
  disabled in `backend/config/settings/base.py` and
  `tools/prod/prod_enable_parallel_flow.sh`; the benchmark wrapper
  `tools/prod/prod_run_cycle15b1_two_shard_runtime_benchmark.sh` now exposes
  explicit appearance flags and cleanup restores them to disabled defaults.
  Local validation passed: shard/Cycle 18 suite `9 passed`, focused Cycle 18
  slice `12 passed`, and the measured-value checker still reports
  `cycle18_runtime_benchmark_present=false`. This is not a §12.6 decision,
  not sharding acceptance, and not Cycle 18 closure. Next action is a governed
  full `combined.mp4` Linux RTX 5090 benchmark with packet validation,
  model-agreement and label-invariant association metrics, DB/GPU/RTT metrics,
  and rollback proof before any acceptance or rejection claim.
- **2026-06-05 open latency-cycle sort updated**: the active optimization
  queue is now dependency-sorted in
  `docs/inference_parallelization_plan.md` and
  `docs/cycle_9_and_10_improvements_todo.md`. Cycle 18.B has now completed
  and is **NOT ACCEPTED**, so the next start point is **Cycle 18.C packet-
  budget and association-readiness redesign**. The speed path remains valuable
  (`DB FPS 5.620 -> 7.410`, Step 2 wall `467.450 s -> 248.324 s` in 18.B),
  but no gain is accepted until packet validity, merge readiness,
  `StudentTracks`, model agreement, label-invariant identity, figures, and
  rollback all pass in a new production Linux RTX 5090 `combined.mp4`
  benchmark.
- **2026-06-05 Cycle 18.B benchmark lock `RELEASED / NOT ACCEPTED`**: current Codex runtime
  session took the production benchmark lock for replay
  `cycle18b-appearance-packet-20260605T151057Z`. Candidate env delta:
  `OFFLINE_VIDEO_SHARDING_ENABLED=1`,
  `OFFLINE_VIDEO_SHARD_COUNT=2`,
  `OFFLINE_VIDEO_SHARD_CONTEXT_FRAMES=256`,
  `OFFLINE_VIDEO_SHARD_TRACK_MAP_MODE=appearance_packet`,
  `OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_ENABLED=1`,
  `OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_APPEARANCE_ENABLED=1`,
  `TRITON_CROP_FRAME_BEHAVIOR_OVERLAP=1`,
  `EMBEDDING_PREFETCH_TRACK_LOOKUP=1`, and
  `EMBEDDING_REDIS_SIDE_EFFECT_COALESCING=1`. Parent job
  `85be8348-dc0b-4319-9974-f1a206203884` completed `4541/4541` frames, figures
  were generated under
  `docs/figures/benchmark_artifacts/cycle18b-appearance-packet-20260605T151057Z/`,
  and rollback restored sharding defaults. Decision: **NOT ACCEPTED** because
  valid boundary packets were `0/2`, merge-ready packets were `0/2`,
  `StudentTracks` regressed `53 -> 65`, minimum model-agreement F1@IoU0.5 was
  `53.730 %`, and minimum all-model global-assignment F1 was `69.752 %`.
  Throughput improved (`DB FPS 5.620 -> 7.410`, Step 2 wall
  `467.450 s -> 248.324 s`), but correctness/identity gates failed. The
  production benchmark lock is now released; do not rerun the same
  `appearance_packet` profile until packet byte-budget validity and shard-1
  association readiness are redesigned.
- **2026-06-05 Agent 19 Cycle 18.C local redesign
  `STAGED_LOCAL_ONLY / NO_PRODUCTION_DECISION`**: Agent 19 resumed the sorted
  Cycle 18.C lane. Local code now samples non-critical packet observations so
  a boundary packet can fit `OFFLINE_VIDEO_SHARD_BOUNDARY_PACKET_MAX_BYTES`
  while keeping track rows, tracker hints, and appearance references; shard
  metadata now carries `shard_count`; boundary summaries and packets now
  include only actual inter-shard edges instead of terminal video edges. The
  Figure Planner lane was completed by sub-agent `Huygens`, and the Figure
  Implementer lane was completed by sub-agent `Archimedes` in
  `tools/prod/prod_generate_cycle_figures.py` with focused tests. The
  production benchmark wrapper now accepts figure slug/status/role metadata so
  a Cycle 18.C run cannot emit mislabeled Cycle 18.B plots. Local validation
  passed: shard/Cycle 18, figure-generator, and shard-planning focused set
  `18 passed`, wrapper `bash -n`, plus py_compile for the modified
  runtime/generator files. This
  is not a §12.6 benchmark decision and does not enable sharding or 15.B2.
  Benchmark lock remains `NOT_HELD`.
- **2026-06-03 Cycle 20 streaming persistence and embedding overlap STAGED**:
  `docs/cycle_20_streaming_persistence_embedding_overlap_investigation.md`
  answers the current architecture question. Current offline `crop_frame`
  behavior writes authoritative `Frame`/`Detection`/`BoundingBox` rows in Step 3
  after the in-memory frame inference aggregation, then queues follow-up
  tracking confirmation and embedding generation after render/audit
  finalization. `OFFLINE_OFFLOAD_POST_STAGES=1` offloads the follow-up chain
  after finalization; it does not stream DB rows or embeddings while the same
  job's inference is still running. Cycle 20 is acceptable only as a future
  governed lifecycle cycle with a disabled-by-default rollback flag, bounded
  queue/backpressure, idempotent PostgreSQL writes, terminal-state coordination,
  and a full production `combined.mp4` before/after benchmark. It is sorted last
  after Cycle 19 unless completed production evidence proves post-stage overlap
  is the next dominant limiter.
- **2026-06-03 Cycle 21 Celery worker/thread/concurrency scaling STAGED**:
  `docs/cycle_21_celery_concurrency_scaling_investigation.md` records the
  operator permission to increase Celery workers/threads only as a governed
  candidate. Current metrics do **not** justify a blind production increase:
  one `process_video_upload` task owns the single-video frame loop, one
  `generate_embeddings` task owns the embedding loop, and Cycle 16.B proved the
  remaining post-stage bottlenecks are embedding DB flush, Redis payload
  serialization, and through-pose tail rather than worker-count starvation.
  Extra workers become useful only when there are multiple independent jobs,
  video shards, split persistence/render/embedding tasks, or a completed
  production matrix proves hidden queue parallelism. Constitution
  §8.1.1 is binding: any worker/thread/queue/GPU-cap increase must document
  baseline/candidate topology, duplicate-worker checks, CPU/RSS/VRAM/DB/Redis
  budgets, full `combined.mp4` benchmark evidence, correctness/model agreement,
  and rollback by env reset plus clean worker restart.
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
- **Production benchmark decision authority gate (2026-06-02)**: only a
  completed end-to-end production benchmark on the native Linux RTX 5090 server
  can create an optimization decision. Component probes, parity probes, local
  tests, direct Triton microbenchmarks, code review, theoretical upper bounds,
  and partial measurements have **NO DECISION AUTHORITY**. They may be recorded
  only as `PROBE_ONLY` / `HYPOTHESIS_ONLY` evidence and must not mark a cycle
  or candidate `ACCEPTED`, `NOT ACCEPTED`, `REJECTED`, `SKIPPED`, `NEGLECTED`,
  `DEPRIORITIZED`, `CLOSED`, or `COMPLETE`.
- **Benchmark decision explanation table (2026-06-02)**: after the required
  production Linux benchmark exists, every optimization decision must include a
  comparison table with baseline replay/job, candidate replay/job, exact video,
  deployed SHA, env/config delta, target gate, FPS, Step 2/frame wall, total
  wall, RTT, GPU utilization, memory, DB parity, model-agreement/correctness,
  decision status, reason for the status, remaining bottleneck, and durable
  evidence paths. A result may not be described as accepted, skipped, rejected,
  or neglected merely because one metric improved or regressed; the
  documentation must explain why the metrics moved and what unresolved
  bottleneck would have to be removed for better results. Component probes must
  report upper-bound wall-time calculations only as benchmark hypotheses.
- **Precision benchmark measurement contract (2026-06-03)**: optimization
  decisions must include enough measurement detail to locate the bottleneck,
  not only a final FPS number. Every production benchmark watcher, collector,
  result doc, and decision table must expose FPS, latency, throughput,
  per-model RTT/call-rate/status/input shape, per-step wall time, per-phase
  stage timing, queue/dispatch timing, pose-tail timing, persistence/render
  timing, embedding/DB/Redis timing, GPU/VRAM/power, worker memory, DB/model
  correctness, model agreement, missing/unavailable metric state, and raw
  evidence paths when the runtime provides them. Missing metrics must be shown
  as unavailable with a reason, never silently treated as zero. A candidate
  cannot be accepted because one metric improved if the precise breakdown shows
  the dominant wall moved elsewhere, latency regressed beyond gate, resource
  contention increased, or correctness/model agreement regressed.
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
