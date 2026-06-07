# MASTER CYCLE TRACKER — single source of truth

**Branch:** `014-yoloe-scene-srvl` · **Maintained:** continuously · **Owner:** agent
This is the canonical index of every cycle + sub-cycle, its status, and the
reason for each accept/refuse. Detail lives in the linked design docs; this file
is authoritative for **status**.

## Status legend
| Code | Meaning |
|---|---|
| ☐ PLANNED | designed, not started |
| ◐ IN-PROGRESS | implementation under way |
| ■ IMPLEMENTED | code written, not yet verified on prod |
| ✔ TESTED | unit/integration tests pass on prod |
| ★ ACCEPTED | passes the §12.5 full RTX-5090 benchmark + §7.1.1 figure bundle (decision authority) |
| ✘ REFUSED | rejected — reason recorded |

> **ACCEPTED requires a completed full end-to-end native-Linux RTX-5090
> benchmark at frame stride = 1** (inference on every decoded frame; stride > 1
> is profiling-only, no authority — Constitution §7.1.1 v2.12.0). TESTED ≠
> ACCEPTED. Every benchmark run (accepted or refused) is recorded with metrics +
> reason in the single ledger **[`docs/BENCHMARK_RESULTS_LEDGER.md`](BENCHMARK_RESULTS_LEDGER.md)**.

---

## Cycle 015 — YOLOE-26L-seg model upgrade
Ref: `docs/entity/cycles/cycle_014_yoloe_scene_srvl.md` §Cycle 015. Commit `66b59ebb`.

| Sub-cycle | Status | Notes / reason |
|---|---|---|
| 015.1 fetch+verify yoloe-26l-seg.pt | ✔ TESTED | sha `a612d2d5…`, torch-loadable |
| 015.2 export ONNX→TRT FP16, deploy Triton | ✔ TESTED | plan `a4357103…`, I/O contract identical |
| 015.3 Triton reload + export verify | ✔ TESTED | ready 200, manifest `7f93fd08` |
| 015.4 full benchmark 26L vs 26s | ★ ACCEPTED | +26.5% objects (88,363→111,792); infer p95 11.7→9.9ms |
| 015.5 FP16-vs-INT8 decision | ★ ACCEPTED | **stay FP16** — inference not the bottleneck; INT8 risks recall gain |
| 015.6 update defaults + docs | ✔ TESTED | settings/base.py + both .env.example |

---

## Cycle 016 — ReID embedding architecture
Ref: `docs/scene_reid_embedding_cycle_plan.md`. Verified end-to-end in full run `a4b08e1c` (77,671 embeddings: student 76,921, teacher 750).

| Sub-cycle | Status | Notes / reason |
|---|---|---|
| 016.1.1 `person_embeddings.py` module | ✔ TESTED | OSNet 512-d, fail-closed, Redis keys (commit `a2127e89`) |
| 016.1.2 wire into `run_scene_frame_lane` (frame_bgr) | ✔ TESTED | benchmark path; 23 embeddings/3-frame (`37502b93`) |
| 016.1.3 wire all-models path (preview imread) | ★ ACCEPTED | full run wrote 77,671 embeddings (`96a751c4`) |
| 016.2 role tagging (student/teacher/person) | ★ ACCEPTED | role-keyed Redis, by_role; full run student/teacher split (`f15efd39`) |
| 016.3 embedding contradiction arbiter | ✔ TESTED | `contradiction_arbiter.py`, 7 tests, wired step 9b (`6baf77db`) |
| 016.4a telemetry in bench_summary | ✔ TESTED | `reid_embeddings` section (`7d39a232`) |
| 016.4b reid figure + Redis collector | ✔ TESTED | `reid_embeddings.png`, manifest 8 figures (`61af30ef`) |
| 016.4c frontend arbitration badge | ☐ PLANNED | API serializer + WS + TS + component (larger) |

---

## Cycle 017 — Unified Telemetry Spine (the metric watcher)
Ref: `docs/unified_telemetry_spine_and_subcycles.md` Part A. **Prerequisite for accepting any 018-023 optimization** (per-sub-stage truth, §7.1.1).

### 017.1 Span primitive — `core/telemetry/spine.py`
| Sub-cycle (atomic) | Status | Notes |
|---|---|---|
| 017.1.1 `Span`+`SpanStatus` dataclasses | ✔ TESTED | commit `148ab788` |
| 017.1.2 contextvar current-span stack | ✔ TESTED | |
| 017.1.3 `span()` context manager | ✔ TESTED | |
| 017.1.4 `@timed` decorator | ✔ TESTED | |
| 017.1.5 unit tests (nesting/error/reset) | ✔ TESTED | 8 tests pass |

### 017.2 Mode-agnostic run context
| Sub-cycle (atomic) | Status | Notes |
|---|---|---|
| 017.2.1 `TelemetryContext` + contextvar | ✔ TESTED | `148ab788` |
| 017.2.2 `frame_scope()` root-span helper | ✔ TESTED | |
| 017.2.3 child scope-inheritance test | ✔ TESTED | |

### 017.3 Decompose postprocess into sub-stage spans (forbid single block)
| Sub-cycle (atomic) | Status | Notes |
|---|---|---|
| 017.3.1 `postprocess.mask_decode` span | ☐ PLANNED | in yoloe_triton decode |
| 017.3.2 `postprocess.nms` span | ☐ PLANNED | |
| 017.3.3 `postprocess.box_decode` span | ☐ PLANNED | |
| 017.3.4 `postprocess.gaze_decode` span | ☐ PLANNED | all-models decode |
| 017.3.5 `postprocess.posture_decode` span | ☐ PLANNED | |
| 017.3.6 `postprocess.scene_normalize` span | ✔ TESTED | scene_lane step 4; fires through real lane (0.01ms) — commit pending |
| 017.3.7 `postprocess.srvl_compute` span | ☐ PLANNED | scene_lane SRVL (in `_run_srvl_lane`) |
| 017.3.8 `postprocess.person_embedding` span (pre/infer/post) | ✔ TESTED | scene_lane step 6b wrapped |
| 017.3.9 `postprocess.contradiction_arbiter` span | ✔ TESTED | scene_lane step 9b wrapped |
| 017.3.10 `persistence.scene_frame` span | ✔ TESTED | fires through real lane (11.7ms) |
| 017.3.10b `persistence.{detections,poses}` spans (all-models) | ☐ PLANNED | in tasks.py persistence |
| 017.3.1-5 mask_decode/nms/box_decode/gaze/posture spans | ☐ PLANNED | in yoloe_triton decode + tasks.py all-models decode |
| 017.3.11 wrap `decode`/`preprocess`/`infer.<model>` + residual "unattributed" | ☐ PLANNED | full-frame accounting |
| 017.3.12 sub-stage instrumentation verified through real lane | ✔ TESTED | prod shell: frame→{scene_normalize,persistence} tree, real durations |

### 017.4 Sinks
| Sub-cycle (atomic) | Status | Notes |
|---|---|---|
| 017.4.1 RingBufferSink (bounded in-proc) | ✔ TESTED | `148ab788` |
| 017.4.2 Redis live sink (`XADD telemetry:{mode}:{job}:{cam}`, capped) | ✔ TESTED | `RedisSpanSink`, lazy redis, fail-isolated, 5 tests |
| 017.4.3 Postgres durable sink (`SpanRecord`, batched flush off critical path) | ✔ TESTED | `SpanRecord` model + migration `0007_spanrecord` APPLIED on prod; `PostgresSpanSink` batched bulk_create, fail-isolated, 2 django_db tests |
| 017.4.4 Rollup (count, p50/p95/p99 per span/scope/window; measured-zero vs unavailable) | ✔ TESTED | `rollup_spans`, nearest-rank pct, no-numpy, 5 tests |
| 017.4.5 sink unit/integration tests | ✔ TESTED | 18 telemetry tests pass (spine+redis+rollup) |

### 017.5 Frontend endpoints (WebGL/WebGPU-ready)
| Sub-cycle (atomic) | Status | Notes |
|---|---|---|
| 017.5.1 REST `GET /api/telemetry/spans` (windowed aggregates) | ✔ TESTED | `TelemetrySpanRollupView` (DRF, IsAuthenticated); filters job/mode/camera/name/since/limit; rollup_spans over SpanRecord; unavailable when empty; 4 tests |
| 017.5.2 SSE `GET /api/telemetry/stream` | ✔ TESTED | `TelemetrySpanStreamView` + `span_stream_events` (Redis XREAD→rollup SSE, heartbeat, idle-break, error-clean); 5 tests, decoupled generator |
| 017.5.3 WS `ws://…/ws/telemetry/{scope}` | ☐ PLANNED | channels consumer |
| 017.5.4 binary metric-frame schema (ArrayBuffer/Float32, column-major) | ☐ PLANNED | GPU-upload-ready |
| 017.5.5 OpenAPI contract version for schema | ☐ PLANNED | |
| 017.5.6 endpoint tests | ☐ PLANNED | |

### 017.6 Frontend realtime renderer
| Sub-cycle (atomic) | Status | Notes |
|---|---|---|
| 017.6.1 TS types `frontend/src/types/telemetry.ts` | ☐ PLANNED | |
| 017.6.2 `useTelemetryStream` hook (WS/SSE→GPU buffer) | ☐ PLANNED | |
| 017.6.3 `TelemetryGpuRenderer` (WebGPU, WebGL2 fallback) | ☐ PLANNED | per-sub-stage lanes/flame |
| 017.6.4 stale/disconnect handling (no fake liveness) | ☐ PLANNED | |
| 017.6.5 vitest + render tests | ☐ PLANNED | |

### 017.7 Always-on wiring (every mode)
| Sub-cycle (atomic) | Status | Notes |
|---|---|---|
| 017.7.1 open `frame_scope` in offline/all-models frame loop (`tasks.py`) | ☐ PLANNED | |
| 017.7.2 open `frame_scope` in benchmark command | ☐ PLANNED | |
| 017.7.3 open `frame_scope` in upload pipeline | ☐ PLANNED | |
| 017.7.4 open `frame_scope` in live consumer | ☐ PLANNED | |
| 017.7.5 `TELEMETRY_SPINE_ENABLED` + `…_SAMPLE_RATE` flags | ☐ PLANNED | bounded live cost |
| 017.7.6 acceptance: identical span tree in all 4 modes; `postprocess` never a leaf | ☐ PLANNED | |

### 017.8 Generated bounding-box throughput (per frame/ms/second/call/call_ms)
| Sub-cycle (atomic) | Status | Notes |
|---|---|---|
| 017.8.1 `compute_box_throughput` (5 dims, None≠0 §1.6) | ✔ TESTED | per frame/ms/second/call/call_ms |
| 017.8.2 `box_throughput_from_spans` (boxes attr) | ✔ TESTED | sums boxes/calls/call_ms, distinct frames |
| 017.8.3 scene_normalize span carries `boxes` count | ✔ TESTED | tagged in run_scene_frame_lane |
| 017.8.4 REST endpoint emits `box_throughput` | ✔ TESTED | in `/api/telemetry/spans` |
| 017.8.5 tag infer.* (person_detector/gaze/posture) box counts | ☐ PLANNED | all-models decode spans (with 017.3.1-5) |

**Cycle 017 acceptance (★):** all modes emit the full sub-stage tree; live WS/WebGPU view shows per-sub-stage latency realtime; figures render per-sub-stage p50/p95 + box throughput (per frame/ms/second/call/call_ms); no bespoke timing at call sites.

---

## Cycle 018 — Async / overlapped Triton dispatch (feed the idle GPU)
Ref: `docs/unified_telemetry_spine_and_subcycles.md` Part B. **Highest impact** (GPU measured idle 2-9%).

| Sub-cycle (atomic) | Status | Notes |
|---|---|---|
| 018.1.1 enable `async_dispatch` flag+config | ☐ PLANNED | |
| 018.1.2 spine: confirm `infer.*` overlap | ☐ PLANNED | |
| 018.2.1 person_detector batched request | ☐ PLANNED | |
| 018.2.2a gaze_horizontal batched | ☐ PLANNED | |
| 018.2.2b gaze_vertical batched | ☐ PLANNED | |
| 018.2.2c gaze_depth batched | ☐ PLANNED | |
| 018.2.3 posture batched | ☐ PLANNED | |
| 018.2.4 OSNet batch verify (≤64) | ☐ PLANNED | already dynamic-batched |
| 018.2.5 batch→(frame,identity) attribution + test (§5.4) | ☐ PLANNED | |
| 018.3.1 bounded queue infer↔postprocess (§8.6) | ☐ PLANNED | |
| 018.3.2 postprocess worker thread | ☐ PLANNED | |
| 018.3.3 backpressure + ordering test | ☐ PLANNED | |
| 018.4.x dynamic_batching tune per model | ☐ PLANNED | one config at a time |
| 018.5 full benchmark accept (GPU↑, infer-wall↓, parity) | ☐ PLANNED | |

---

## Cycle 019 — GPU / vectorized postprocess (kill the 66%)
| Sub-cycle (atomic) | Status | Notes |
|---|---|---|
| 019.1.1 batch `process_mask` on CUDA | ☐ PLANNED | |
| 019.1.2 mask numerical parity diff | ☐ PLANNED | |
| 019.2.1 NMS via torchvision GPU | ☐ PLANNED | |
| 019.2.2 NMS parity | ☐ PLANNED | |
| 019.3.1 gaze decode vectorized | ☐ PLANNED | |
| 019.3.2 gaze parity | ☐ PLANNED | |
| 019.4.1 posture decode vectorized | ☐ PLANNED | |
| 019.4.2 posture parity | ☐ PLANNED | |
| 019.5.1 scene normalize de-loop (vectorize) | ☐ PLANNED | |
| 019.5.2 scene normalize parity | ☐ PLANNED | |
| 019.6 SRVL no-pair-loop verify/extend | ☐ PLANNED | |
| 019.7 aggregate correctness-agreement table | ☐ PLANNED | vs frozen baseline |
| 019.8 full benchmark accept | ☐ PLANNED | postprocess 739→≤200 ms/f |

---

## Cycle 020 — Pose kinematics off the CPU critical path
| Sub-cycle (atomic) | Status | Notes |
|---|---|---|
| 020.1.1 port pose-kinematics to GPU/vectorized | ☐ PLANNED | currently device=cpu |
| 020.1.2 kinematics parity | ☐ PLANNED | |
| 020.2 side-lane compute, join at persistence | ☐ PLANNED | |
| 020.3 spine: rtmpose+kinematics no longer serialize | ☐ PLANNED | |
| 020.4 full benchmark accept | ☐ PLANNED | 11.5→≤5 ms/f off serial path |

---

## Cycle 021 — Decode + preprocess pipelining
| Sub-cycle (atomic) | Status | Notes |
|---|---|---|
| 021.1 producer thread decode+letterbox+tensor | ☐ PLANNED | |
| 021.2 pinned host buffers/reuse | ☐ PLANNED | |
| 021.3 bounded frame queue + backpressure (§8.6) | ☐ PLANNED | |
| 021.4.1 NVDEC GPU-decode spike (CUDA 12.8) | ☐ PLANNED | optional |
| 021.4.2 adopt-or-drop NVDEC | ☐ PLANNED | decide from spike |
| 021.5 full benchmark accept | ☐ PLANNED | decode/preprocess hidden |

---

## Cycle 022 — Governed concurrency scaling (§8.1.1)
| Sub-cycle (atomic) | Status | Notes |
|---|---|---|
| 022.1 declare queue topology + ceilings + dup-worker check | ☐ PLANNED | |
| 022.2 raise offline concurrency one step | ☐ PLANNED | |
| 022.3 spine: measure GPU/DB/Redis contention | ☐ PLANNED | |
| 022.4 repeat until GPU-util plateaus | ☐ PLANNED | atomic loop |
| 022.5 full benchmark accept (decision table) | ☐ PLANNED | |

---

## Cycle 023 — Persistence / Redis / DB batching
| Sub-cycle (atomic) | Status | Notes |
|---|---|---|
| 023.1 bulk Postgres insert detections | ☐ PLANNED | |
| 023.2 bulk insert poses | ☐ PLANNED | |
| 023.3 bulk insert embeddings + Redis pipeline | ☐ PLANNED | |
| 023.4 persistence off per-frame critical path | ☐ PLANNED | |
| 023.5 full benchmark accept | ☐ PLANNED | |

---

## Refused / superseded log
| Item | Status | Reason |
|---|---|---|
| YOLOE INT8 engine | ✘ REFUSED | 015.5 — FP16 inference already not the bottleneck (GPU idle); INT8 would risk the 26L recall gain for negligible latency benefit |
| Carrying frames in inference queue for all-models embeddings | ✘ REFUSED | superseded by preview-imread (016.1.3) — avoids queue/memory growth (§8.6) |

## Program exit criterion
Full `combined.mp4` all-models RTX-5090 run **≤ 600 s** (≥7.6 FPS sustained, from ~2.2–3.3 now) with correctness parity vs the frozen `a4b08e1c` baseline, and the live WebGPU view proving the GPU is actually working.
