# Cycle 016 — Real Person/Teacher ReID Embeddings → Redis → Contradiction Arbiter

**Branch:** `014-yoloe-scene-srvl` · **Status:** planned · **Author:** 2026-06-07
**Priority:** lowest latency · highest accuracy · highest throughput (per user)

## Problem

The production run records `embedding_backend: sha256_stub` (see
`backend/apps/pipeline/audit_schema.py:89`, written at
`backend/apps/video_analysis/tasks.py:720`). Person identity embeddings are a
hash stub — they carry no appearance information, so scene contradictions
(people-count mismatch / missing person) cannot be arbitrated by appearance.

## Existing components (already built — this cycle WIRES them, not builds)

| Component | Location | State |
|---|---|---|
| OSNet-AIN ReID engine | Triton `osnet_ain_x1_0` (`input[3,256,128]`→`embedding[512]`, dynamic-batched, warmup) | **deployed + ready** |
| OSNet ReID Triton client | `backend/apps/pipeline/services/reid_triton_client.py` (`TritonReidClient.embed_crops`, fail-closed, L2-norm, provenance) | **complete** |
| Conservative ReID match | `backend/apps/tracking/reid.py` (`cosine_similarity`, `is_reid_match`@0.85, `find_best_reid_match`, `evaluate_conservative_reid`) | **complete** (constitution §4.3) |
| Embedding persistence + Redis cache | `backend/apps/tracking/embeddings.py` (768-dim path, BoT-SORT→cv2→stub) | exists; identity path uses stub |
| Scene contradiction recorder | `backend/apps/video_analysis/scene/contradictions.py` (`record_contradiction_events`) | exists; **no embedding arbiter** |
| Scene recovery candidates | `backend/apps/video_analysis/scene/recovery_*.py` | exists; geometry-only |

## Design decision: OSNet (512-dim) is the identity embedding

Use `osnet_ain_x1_0` (purpose-built ReID, already deployed, batched ≤64,
~8 ms queue) as the **person-identity** embedding for arbitration — not the
768-dim BoT-SORT/cv2 descriptor (which is a tracking-association feature and is
falling back to the hash stub). Identity arbitration needs ReID-grade features.

## Stages (each independently testable + benchmarked)

### Stage 1 — Real embeddings for scene persons → Redis (replace stub)
- In the scene person path (person-class detections from YOLOE-26L), collect
  `(frame_bgr, xyxy)` per person and call
  `get_offline_reid_client().embed_crops(...)` (batched per frame).
- Persist vectors to Postgres (`FrameEmbedding`/`SceneObjectObservation` link)
  and cache in Redis with constitution §3.4-scoped keys:
  `reid:{mode}:{job_id}:{frame}:{local_track_id}` → 512-d L2-normalized vector,
  rebuildable from durable rows.
- Record `embedding_backend: "triton_osnet_ain_x1_0"` in the audit (replaces
  `sha256_stub`); fail-closed → `unavailable` (never measured-zero).
- **Verify:** unit test (mock Triton) + 3-frame prod run shows non-stub backend
  and populated Redis keys.

### Stage 2 — Student/teacher embeddings via person_detector classes
- `person_detector` (`output0[14,8400]`) emits class incl. student/teacher
  (confirm class map). For each, embed the crop (OSNet) and tag role in the
  Redis key: `reid:{mode}:{job}:role:{student|teacher}:{track}`.
- **Verify:** role tagging test + prod run populates both roles.

### Stage 3 — Embedding-based contradiction arbiter
- In `scene/contradictions.py` recovery, when a person is missing/contradicted,
  embed the candidate region and run `reid.find_best_reid_match` /
  `evaluate_conservative_reid` against stored role/track embeddings.
- Append-only decision: similarity ≥ threshold → "already detected" (resolved
  with provenance + score); ambiguous/below-threshold → `unresolved` (constitution
  §4.3/§4.6 — never force a merge).
- **Verify:** integration test (two crops same/different identity) + prod run
  showing arbitrated contradictions with scores.

### Stage 4 — Telemetry + figures + frontend wiring
- Telemetry: per-frame OSNet RTT, embed count, arbiter decisions
  (resolved/unresolved/ambiguous), Redis hit/miss.
- Figures: embedding-coverage + arbiter-decision charts in the benchmark bundle.
- Frontend: surface arbitration state on contradiction rows (truth-state badge).
- **Verify:** full RTX-5090 benchmark on combined.mp4; metrics + figures + DB +
  Redis + telemetry all consistent (no `unavailable` where data exists).

## Constraints (constitution)
- Triton-only inference; fail-closed; PostgreSQL authority; Redis is a
  rebuildable accelerator (§3.4); append-only identity decisions, one-to-one,
  conservative-under-uncertainty (§4.3/§4.6); measured-zero ≠ unavailable.
- Each stage carries `streaming_compatibility` declaration (§8.6).
- Full benchmark on native-Linux RTX 5090 gives decision authority (§12.5).

## Open questions to confirm
1. `person_detector` class map — which indices are student vs teacher?
2. ReID similarity threshold for classroom (default 0.85 — validate on data).
3. Embedding retention window in Redis (TTL) + Postgres model for scene ReID.
