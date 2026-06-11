# BENCHMARK RESULTS LEDGER — single source of truth for every benchmark

**Last updated:** 2026-06-11
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
★ ACCEPTED · ✘ REFUSED · ◷ BASELINE (reference, not a decision) · ☐ PENDING

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
