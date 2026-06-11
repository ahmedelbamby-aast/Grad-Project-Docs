# BENCHMARK RESULTS LEDGER — single source of truth for every benchmark

**Last updated:** 2026-06-12
**Branch:** `015-xai-anomaly-score` · **Authority:** Constitution §7.1.1 / §12.5
This is the **one place** that records every benchmark run, accepted or refused,
with its config, metrics, and the decision reason. No optimization decision is
valid unless its run appears here.

## Binding rules
- **Frame stride = 1 on EVERY benchmark** (inference on every decoded frame).
  Stride > 1 is profiling-only and carries no decision authority; a recorded
  stride ≠ 1 invalidates acceptance (Constitution §7.1.1, v2.12.0).
- **ACCEPTED requires a completed full end-to-end native-Linux RTX 5090
  benchmark** with §7.1.1 precision metrics captured and saved.
- Hardware (all runs): RTX 5090, CUDA 12.8, TensorRT 10.16.1.11, Triton offline
  profile (`:39100/1/2`), PostgreSQL, `combined.mp4`
  (`Raw Data/Diverse Classroom Enviroments/combined.mp4`, 4541 frames @ 30 fps).

## Status legend
★ ACCEPTED · ✘ REFUSED · ◷ BASELINE (reference, not a decision) ·
◆ OPERATIONAL DEFAULT (not benchmark acceptance) · ☐ PENDING

---

## Ledger

### B001 — YOLOE-26s-seg scene benchmark (baseline) · ◷ BASELINE
| Field | Value |
|---|---|
| Run | `cycle014-yoloe-scene-20260607T121352Z` · scene-only lane |
| Stride | **1** (4541/4541 frames) |
| Objects | 88,363 |
| Throughput | 15.29 FPS |
| YOLOE inference | p50 7.16 ms · p95 11.66 ms · p99 16.43 ms |
| Frame latency | p50 65.9 ms · p95 96.3 ms · p99 120.7 ms |
| Decision | ◷ baseline for the 26L comparison |

### B002 — YOLOE-26L-seg scene benchmark (model swap) · ★ ACCEPTED
| Field | Value |
|---|---|
| Run | `cycle014-yoloe-scene-20260607T162021Z` · scene-only lane · commit `66b59ebb` |
| Stride | **1** (4541/4541) |
| Objects | 111,792 (**+26.5%** vs B001) |
| Throughput | 8.51 FPS |
| YOLOE inference | p50 8.13 ms · p95 **9.93 ms** · p99 10.80 ms |
| Frame latency | p50 120.0 ms · p95 157.7 ms · p99 184.6 ms |
| **Decision** | ★ **ACCEPTED** |
| **Reason** | +26.5% recall for ~par inference (p95 *lower* than 26s: 9.93 vs 11.66 ms; tails tighter). FPS drop is postprocess-bound, not model. FP16 kept over INT8 because inference is not the bottleneck and INT8 would risk the recall gain. |

### B003 — Full all-models pipeline + ReID embeddings (end-to-end) · ★ ACCEPTED + ◷ OPTIMIZATION BASELINE
| Field | Value |
|---|---|
| Run | job `a4b08e1c-e824-4a63-babf-ca739a981919` · `crop_frame` all-models · 2026-06-07 |
| Stride | **1** (4541/4541) |
| Wall time | **1912.3 s (~31.9 min)** |
| Throughput | **2.37 FPS** (end-to-end, all models) |
| Scene objects | 111,792 (26L) · detected objects 72,750 |
| **ReID embeddings** | **77,671 in Redis** — student 76,921, teacher 750 (OSNet 512-d, role-keyed) |
| Aggregate compute | postprocess 4,829 s · inference 2,114 s · preprocess 105 s · decode 0.08 s |
| GPU utilization | **2–9 %** (36 % spike), power ~90–100 W / 575 W → **GPU idle, CPU/orchestration-bound** |
| **Decision** | ★ **ACCEPTED** (Cycle 016 embedding architecture — items 4/8/9 proven live) · ◷ **baseline for Cycles 018–023** |
| **Reason** | Real OSNet student/teacher embeddings extracted + cached end-to-end in the production pipeline, embedding-arbiter wired. Accepted as functional verification of the embedding architecture. Simultaneously the frozen latency baseline: the optimization program must beat **1912 s → ≤600 s** with correctness parity. |

### B010 — Cycle 015.17 attempt 1 serial baseline · ◷ BASELINE
| Field | Value |
|---|---|
| Run | `cycle015-17-prod-20260611-baseline` · job `449f1540-f586-4049-829b-a4dd46bf166f` · commit `a1551966` |
| Config | Native Linux RTX 5090 · canonical `combined.mp4` · stride **1** · async persistence off |
| Completion | `4541/4541` · `2729.178 s` · **1.663871 DB FPS** |
| Correctness | frames `4541` · detections/bboxes `127117` · embeddings `126519` · tracks `138` |
| GPU | average `4.672%` |
| Decision | ◷ baseline for R003 |
| Evidence | `docs/figures/benchmark_artifacts/cycle015-17-prod-20260611/` |

### R003 — Cycle 015.17 attempt 1 cross-process persistence · ✘ REFUSED
| Field | Value |
|---|---|
| Run | `cycle015-17-prod-20260611-candidate` · job `349010db-51c9-4b1a-8806-4a22a8afc541` · commit `a1551966` |
| Config | Native Linux RTX 5090 · canonical `combined.mp4` · stride **1** · async persistence on |
| Completion | `4541/4541` · `2869.358 s` · **1.582584 DB FPS** (`-4.88%`) |
| Correctness | frames/detections/bboxes/embeddings exact · tracks `138 -> 140` |
| Lane | `8185` produced · `1031` applied · `7139` failed · not drained · `3968` serially reconciled |
| **Decision** | ✘ **REFUSED** |
| **Reason** | Consumer color ownership reset between packets, causing apply failures, drain timeout, serial repair, throughput regression, and track-row divergence. |
| Rollback | Async persistence disabled and workers restarted |
| Evidence | `docs/figures/benchmark_artifacts/cycle015-17-prod-20260611/` |

### B011 — Cycle 015.17 r2 serial baseline · ◷ BASELINE
| Field | Value |
|---|---|
| Run | `cycle015-17-prod-r2-20260611-baseline` · job `e60e678a-8862-4f87-8177-54b979aaf884` · commit `6f0a4805` |
| Config | Native Linux RTX 5090 · canonical `combined.mp4` · stride **1** · async persistence off |
| Completion | `4541/4541` · `2742.376 s` · **1.655863 DB FPS** |
| Correctness | frames `4541` · detections/bboxes `127117` · embeddings `126519` · tracks `138` |
| Step 2 / GPU | `1626.406689 s` · average `4.666%` · peak `51%` |
| Decision | ◷ baseline for R004 |
| Evidence | `docs/figures/benchmark_artifacts/cycle015-17-prod-r2-20260611/` |

### R004 — Cycle 015.17 r2 cross-process persistence · ✘ REFUSED
| Field | Value |
|---|---|
| Run | `cycle015-17-prod-r2-20260611-candidate` · job `f50e8e0e-d997-4008-ac8f-b5a9a65fbdcd` · commit `6f0a4805` |
| Config | Native Linux RTX 5090 · canonical `combined.mp4` · stride **1** · async persistence on |
| Completion | `4541/4541` · `2763.274 s` · **1.643340 DB FPS** (`-0.76%`) |
| Step 2 | `1629.966177 s` (`+0.22%`) |
| Correctness | frames/detections/bboxes/embeddings exact · tracks `138 -> 149` |
| Lane | `8185` produced/applied · zero failed · drained · zero serial reconciliation |
| GPU | average `5.032%` · peak `48%`; embedding tail not captured |
| **Decision** | ✘ **REFUSED** |
| **Reason** | DB FPS regressed and stayed far below `15 FPS`; track rows diverged; runner rolled back at `03:40:31Z` before actual terminal completion at `03:43:32Z`, invalidating clean terminal GPU/rollback authority. |
| Rollback | Flag off and workers restarted; serial setting verified |
| Evidence | `docs/xai_anomaly/cycle_015_17_results.md`; `docs/figures/benchmark_artifacts/cycle015-17-prod-r2-20260611/` |

### B012 — Cycle 015.17 r3 serial baseline · ◷ BASELINE
| Field | Value |
|---|---|
| Run | `cycle015-17-prod-r3-20260611-baseline` · job `83ca0e27-16b5-4fbb-ac5e-203168a4a088` · commit `31d34d8b` |
| Config | Native Linux RTX 5090 · canonical `combined.mp4` · stride **1** · async persistence off |
| Completion | `4541/4541` · `2703.859 s` · **1.679452 DB FPS** |
| Correctness | frames `4541` · detections/bboxes `127117` · embeddings `126519` · tracks `138` |
| Decision | ◷ baseline for R005 and B013 determinism control |
| Evidence | `docs/xai_anomaly/cycle_015_17_results.md`; `docs/figures/benchmark_artifacts/cycle015-17-prod-r3-20260611/` |

### R005 — Cycle 015.17 r3 async persistence · ✘ REFUSED
| Field | Value |
|---|---|
| Run | `cycle015-17-prod-r3-20260611-candidate` · job `4144eef0-5fea-4508-af7c-3e0aaf461273` · commit `31d34d8b` |
| Config | Native Linux RTX 5090 · canonical `combined.mp4` · stride **1** · async persistence on + zero-reference prune |
| Completion | `4541/4541` · `2678.369 s` · **1.695435 DB FPS** |
| Aggregate rows | frames/detections/bboxes/embeddings exact · tracks `138 -> 139` |
| Exact comparison | Content delta `0`; six assignments changed from serial track `0` to async track `43`; six reused embedding vectors also differ |
| **Decision** | ✘ **REFUSED** |
| **Reason** | The embedding-existence guard blocked the final stale-revision correction. Aggregate row parity hid a deterministic track-assignment and reused-vector fidelity defect. |
| Evidence | `docs/xai_anomaly/cycle_015_17_results.md`; `docs/figures/benchmark_artifacts/cycle015-17-prod-r3-20260611/`; `docs/figures/benchmark_artifacts/cycle015-17-prod-r3det-20260611/determinism_control_comparison.json` |

### B013 — Cycle 015.17 serial determinism control · ◷ BASELINE CONTROL
| Field | Value |
|---|---|
| Run | `cycle015-17-prod-r3det-20260611-baseline` · job `7082dc07-e149-4a65-9ef9-730aa71e26fc` · commit `31d34d8b` |
| Config | Native Linux RTX 5090 · canonical `combined.mp4` · stride **1** · async persistence off |
| Completion | `4541/4541` · `2771.757 s` · **1.638311 DB FPS** |
| Correctness | frames `4541` · detections/bboxes `127117` · embeddings `126519` · tracks `138` |
| Determinism | Full content and track-assignment multisets exactly match B012; both assign the six disputed rows and all 18 related embeddings to track `0` with the same vector digest |
| Decision | ◷ reproducibility control; no optimization acceptance claim |
| Rollback | Serial setting verified; workers restarted |
| Evidence | `docs/figures/benchmark_artifacts/cycle015-17-prod-r3det-20260611/` |

### B014 — Cycle 015.17 r4 serial baseline · ◷ BASELINE
| Field | Value |
|---|---|
| Run | `cycle015-17-prod-r4-20260611-baseline` · job `b3c9241e-9d94-4e9a-9cbf-de0cd56d1e8b` · commit `770f47d8` |
| Config | Native Linux RTX 5090 · canonical `combined.mp4` · stride **1** · async persistence off |
| Completion | `4541/4541` · `2744.926 s` · **1.654325 DB FPS** |
| Correctness | frames `4541` · detections/bboxes `127117` · embeddings `126519` · tracks `138` |
| Decision | ◷ frozen baseline for R006 and R007 |
| Evidence | `docs/figures/benchmark_artifacts/cycle015-17-prod-r4-20260611/`; copied into the r5 package for the corrected comparison |

### R006 — Cycle 015.17 r4 embedded-revision repair · ✘ REFUSED
| Field | Value |
|---|---|
| Run | `cycle015-17-prod-r4-20260611-candidate` · job `0d045329-6283-4cab-99c8-451cb200a71c` · commit `770f47d8` |
| Config | Native Linux RTX 5090 · canonical `combined.mp4` · stride **1** · async persistence on |
| Completion | `4541/4541` · `2659.457 s` · **1.707491 DB FPS** (`+3.21%`) |
| Correctness | Aggregate rows exact · tracks `138 -> 139`; content delta `0`; six assignment rows still `0 -> 43` |
| Lane | `8185` produced/applied · drained · zero unresolved · zero embedded-frame reconciliations |
| **Decision** | ✘ **REFUSED** |
| **Reason** | Secondary-model public labels omit track IDs, so the packet signature treated `attention_tracking 43 -> 0` as unchanged and never enqueued the authoritative revision. |
| Rollback | Async persistence disabled and workers restarted |
| Evidence | `docs/figures/benchmark_artifacts/cycle015-17-prod-r4-20260611/` |

### R007 — Cycle 015.17 r5 corrected async persistence · ✘ NOT ACCEPTED
| Field | Value |
|---|---|
| Run | `cycle015-17-prod-r5-20260611-candidate` · job `3095ec8a-ce49-4057-a7b3-2c7ea4d2cc64` · commit `bec9f0f4` |
| Baseline | B014; no serial behavior-affecting code changed |
| Config | Native Linux RTX 5090 · canonical `combined.mp4` · stride **1** · async persistence on |
| Completion | `4541/4541` · `2705.285 s` · **1.678566 DB FPS** (`+1.47%`) |
| Exact correctness | content delta `0`; assignment delta `0`; embedding assignment/vector delta `0`; tracks `138 = 138` |
| Lane | `8189` produced/applied · `3648` revisions · `11` phantoms pruned · drained · zero unresolved |
| **Decision** | ✘ **NOT ACCEPTED as an optimization; fidelity defect resolved** |
| **Reason** | Exact PostgreSQL fidelity passed, but the end-to-end gain is not material and remains far below the binding `15 FPS` target. |
| Rollback | `OFFLINE_ASYNC_PERSISTENCE_ENABLED=0`; workers restarted; serial setting verified |
| Evidence | `docs/figures/benchmark_artifacts/cycle015-17-prod-r5-20260611/` |

### O001 — Cycle 015.17 r5 production offline default · ◆ OPERATIONAL DEFAULT
| Field | Value |
|---|---|
| Source | R007 · job `3095ec8a-ce49-4057-a7b3-2c7ea4d2cc64` |
| Deployed commit | `d64c2c91` |
| Production config | stride `1`; async enabled; lanes `db_rows,embedding`; queue `pipeline.offline.persistence`; idle exit `10000000 ms` |
| Scope | Offline production jobs only; live jobs remain excluded and require async disabled |
| **Decision** | ◆ Use the fidelity-correct r5 configuration as the production offline default |
| Benchmark authority | R007 remains **NOT ACCEPTED** as a throughput optimization; no new benchmark decision |
| Rollback | Set `OFFLINE_ASYNC_PERSISTENCE_ENABLED=0` and restart all Celery workers |
| Evidence | `docs/figures/benchmark_artifacts/cycle015-17-prod-default-20260612/runtime_activation.json` |

---

## Pending acceptance (Cycles 017–023)
Each will append a row here with stride=1, full §7.1.1 metrics, and decision+reason.

| ID | Cycle | Target gate | Status |
|---|---|---|---|
| B004 | 018 async/overlapped dispatch | GPU util ↑ (≥50%), inference wall ↓, parity | ☐ PENDING |
| B005 | 019 GPU/vectorized postprocess | postprocess 739→≤200 ms/f, parity | ☐ PENDING |
| B006 | 020 pose off CPU critical path | kinematics off serial path | ☐ PENDING |
| B007 | 021 decode/preprocess pipelining | decode/preproc hidden | ☐ PENDING |
| B008 | 022 concurrency scaling | GPU-util plateau, no contention regress | ☐ PENDING |
| B009 | 023 persistence/Redis/DB batching | persistence off critical path | ☐ PENDING |
| B0xx | **PROGRAM EXIT** | full all-models run **≤600 s** + parity vs B003 | ☐ PENDING |

## Refused log (cross-referenced in MASTER_CYCLE_TRACKER.md)
| ID | Item | Decision | Reason |
|---|---|---|---|
| R001 | YOLOE INT8 engine | ✘ REFUSED | Inference already not the bottleneck (GPU idle, B003); INT8 risks the 26L recall gain for negligible latency benefit. |
| R002 | Carry frames in inference queue for all-models embeddings | ✘ REFUSED | Superseded by preview-imread — avoids bounded-queue memory growth (§8.6). |
| R003 | Cycle 015.17 attempt 1 cross-process persistence | ✘ REFUSED | Apply failures, drain timeout, serial repair, throughput regression, and track-row divergence. |
| R004 | Cycle 015.17 r2 cross-process persistence | ✘ REFUSED | Clean DB lane, but DB FPS and track rows regressed; terminal lifecycle contamination invalidated clean GPU/rollback authority. |
| R005 | Cycle 015.17 r3 cross-process persistence | ✘ REFUSED | Determinism control proved six stale embedded assignments and their reused vectors diverged from both serial runs. |
| R006 | Cycle 015.17 r4 embedded-revision repair | ✘ REFUSED | The packet signature omitted explicit track IDs for secondary models, so the final `43 -> 0` revision was not enqueued. |
| R007 | Cycle 015.17 r5 corrected async persistence | ✘ NOT ACCEPTED | Exact content, assignment, and embedding-vector parity passed; throughput improved only `1.47%` and remained `1.678566 FPS`. |
