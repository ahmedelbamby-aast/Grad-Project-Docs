# Phantom Tracks & Track-ID Persistence — Research-Backed Best Practices

**Created:** 2026-06-11
**Context:** Cycle 015.17 refused on track parity (serial 138 → async 149).
Root cause: superseded person-interpolation revisions orphan `StudentTrack`
rows. This doc records the researched best practices and how each maps to our
cross-process (multi-worker) persistence pipeline.

## Two distinct problems

1. **Persistence-layer phantoms** — a concurrency/idempotency problem: a
   consumer persists an early frame revision (creating a track row), a later
   revision supersedes the frame's boxes, and the old track row is left with
   zero references. This is *not* a tracking-algorithm bug.
2. **Track-ID stability across workers** — the same physical person must keep
   the same `tracking_id` regardless of which worker/thread persists it, and
   concurrent writes must not duplicate or orphan rows.

## Best practices from the literature

### A. Idempotent consumer / ordered persistence (distributed systems)
- **Idempotency key + DB unique constraint** — record a processed-message id (or
  put it on the business entity) and rely on an atomic unique-constraint /
  `INSERT … ON CONFLICT`, so duplicate / at-least-once redelivery cannot create
  duplicate rows. The check-then-insert race is fixed by the constraint, *not*
  by pre-checking. (microservices.io idempotent-consumer; Conduktor.)
- **Monotonic version / last-writer-wins** — stamp each update with a
  monotonically increasing revision; reject or overwrite with anything whose
  revision is ≤ the persisted one, so a stale/superseded update leaves no
  residue. (Kafka producer sequence numbers; monotonic-write consistency.)
- **Single-writer-per-key ordering** — route all updates for a key to one
  consumer so revisions apply in order (partitioned consumer groups).
- **Reference-counted / cascade cleanup** — when the last reference to an entity
  is removed, GC the entity in the *same* transaction, rather than leaving
  orphans for a later sweep.

### B. MOT track lifecycle (computer vision)
- **tentative → confirmed → deleted** with `n_init` (min hits before a track is
  confirmed) and `max_age` (frames before a lost track is deleted). DeepSORT
  keeps tracks tentative for the first ~3 frames and *deletes* them if not
  re-associated — this is precisely what stops single-false-positive "ghost"
  tracks from being treated as real. (DeepSORT; ByteTrack.)
- **ByteTrack** additionally associates low-confidence detections in a second
  pass to avoid dropping a tracklet during occlusion (fewer ID switches:
  IDF1 76.9 → 79.3, ID switches 291 → 159 vs SORT).

## How our pipeline already follows the best practices

| Best practice | Our implementation | Status |
|---|---|---|
| Idempotency key | SHA-256 `scene:{job}:{frame}`; `update_or_create(Frame)`; embedding-guard skip | ✅ |
| Unique constraint | `UniqueConstraint(job, tracking_id)` on `StudentTrack` | ✅ |
| Single-writer-per-key | `db_rows` lane = **1** consumer (`consumer_count_for_lane`) → ordered revisions | ✅ |
| Monotonic revision | compact packet `packet_revision`; replace-pre-embedding supersede | ✅ |
| Deterministic IDs across workers | authoritative `tracked_ids` computed once in the inference process and passed to finalize — consumers never invent IDs | ✅ |
| Reference-counted cleanup | finalize prune (`_async_persistence_prune_phantom_tracks`) — batch reconcile | ✅ (global net) |

The single-writer + idempotency-key + monotonic-revision combination is exactly
the recommended idempotent-consumer design. The finalize prune is the standard
"reconcile orphans" sweep.

## The improvement applied (prevention, not just reconciliation)

The finalize prune is correct but *post-hoc*: phantoms accumulate during the run
(observed mid-run on r3: ~148 tracks) and are reconciled to 138 only at the end.
Best practice B + A.4 says to GC at the moment the last reference is removed.

**`OFFLINE_PERSIST_INLINE_ORPHAN_GC`** (default-off): when a revision supersedes
a frame's boxes (`replaced_pre_embedding_packet`), capture the superseded
tracks, and after the new boxes are written, delete — *in the same
transaction* — any superseded track that lost its last box and is not in the new
revision. The check is bounded to this frame's superseded tracks (no full-table
scan), so it adds no hot-path scan; the finalize prune remains the global safety
net (defense in depth). Implemented in `_persist_offline_detection_frame_packet`;
tested by `test_inline_orphan_gc_removes_superseded_track` (and the default-off
companion). Benchmarked independently after the 015.17 parity verdict.

## Track-ID persistence across threads — guarantees

1. **Deterministic assignment**: track IDs come from the tracker in the single
   inference process; the authoritative final set is computed once and passed to
   the finalize barrier. Consumers persist exactly those IDs — they never
   allocate IDs locally, so two workers cannot disagree.
2. **Ordered application**: the `db_rows` lane uses one consumer, so revisions
   for a frame apply in arrival order (monotonic), and a later revision cleanly
   supersedes an earlier one.
3. **Idempotent redelivery**: at-least-once delivery is absorbed by the frame
   `update_or_create` + embedding-existence guard + the `(job, tracking_id)`
   unique constraint — redelivery never duplicates a track or its boxes
   (covered by `test_db_rows_writer_applies_packet_and_is_idempotent`).
4. **Reference integrity**: a track row exists iff it is referenced — enforced
   eagerly by the inline GC (when enabled) and globally by the finalize prune.

The fourth guarantee needs one qualification: an embedding can make a stale
frame revision referenced before finalization. That case requires the exact
embedded-revision repair described below; zero-reference GC alone cannot repair
it.

## Measured on prod (r3 + determinism control, 2026-06-11)

Evidence:
`docs/figures/benchmark_artifacts/cycle015-17-prod-r3det-20260611/`.

The second serial run produced the same `138` tracks and the same complete
127117-row track-assignment multiset as the first serial run. The async run
produced `139` tracks. Its detection-content multiset was identical, but six
rows were assigned to track `43` while both serial runs assigned those exact
rows to track `0`.

This proves the residual was an async persistence fidelity defect, not an
inherent tracker-variance band. The stale revision survived because linked
embeddings caused `_persist_offline_detection_frame_packet` to return
`skipped_existing_embeddings`.

Track-level embedding reuse widened the defect: both serial runs stored all 18
track-0 embeddings under vector digest
`91a5f29504d77d65387bfdfdf177e9bef317c33310adb1126c0ccbedc64be899`,
while the async run stored the six stale track-43 rows under
`92e8fd3f9b4431f0515cd672331cfe719cee4db850fdc64b79d04d844f5834b7`.
An exact repair must therefore reconcile the track reference and the reused
vector.

The implemented finalization repair:

- requires exact model/class/confidence/coordinate content equality;
- reassigns bounding-box and embedding track references atomically;
- restores the target track's earliest same-model canonical vector when
  track-level reuse is enabled;
- clears deleted-track Redis embedding keys after the PostgreSQL transaction
  commits;
- refuses missing target vectors, ambiguous duplicates, content mismatch, or
  non-packet evidence references;
- rechecks the authoritative packet signature and fails closed.

## Why appearance auto-merge remains unsafe

We scored the actual candidate job to test whether embeddings can deterministically
resolve track 43 by appearance:

- **Within-track cohesion is trivially 1.0 for every track** — embeddings are
  *reused* per track (126 382 / 126 519 reused), so the same vector repeats
  within a track. Within-track cohesion cannot detect identity problems here.
- **Cross-track discrimination is weak.** Track 43's nearest neighbours by
  appearance are 0.93 (track 90), 0.93 (track 102), 0.92 (track 17/64/100/68) —
  i.e. *many genuinely distinct students* sit at 0.92–0.93 cosine, **above** the
  0.85 re-ID threshold. In a classroom (similar uniforms / pose / resolution),
  OSNet embeddings do not separate students reliably.

**Therefore a destructive appearance-merge is unsafe in this setting** and is
also forbidden by §4.3/§4.6 ("never force a merge"): merging track 43 into its
top match would risk fusing two different students. The data validates the
doctrine. The embedding tooling's safe role here is **non-destructive scoring /
flagging** (`reconcile_job_track_identities`, `reid_decision='rejected'` for
review), not auto-merge. Deterministic ID stability, if ever required, must come
from a stronger discriminator (classroom-fine-tuned ReID) or tracker-level re-ID
at birth — never from thresholded merging on weakly-separated embeddings.

## r4/r5 Production Resolution

r4 proved that the embedded-row repair was not reached. The packet signature
used the public overlay label, and secondary-model labels intentionally omit
the track ID. Therefore `attention_tracking 43 -> 0` looked unchanged and the
authoritative revision was not enqueued.

Commit `bec9f0f4` added explicit track ID, class, confidence, coordinates, and
model to the signature. The r5 production candidate then matched the serial
baseline exactly:

- box content multiset delta `0`;
- track-assignment multiset delta `0`;
- embedding assignment/vector multiset delta `0`;
- `138` tracks in both jobs;
- the same 18 canonical track-0 vectors and no track `43`.

The four extra r5 revisions match frames `512` through `515`, so the correction
now occurs before embedding generation. The embedded-row reconciler remains a
fail-closed fallback for other orderings.

**Decision:** do not auto-merge by appearance. The stale-revision fidelity bug
is resolved by exact ordered persistence, not ReID similarity. Keep Cycle
015.17 default-off because its corrected production gain was only `1.47%`
(`1.678566 FPS`), not because of a remaining correctness defect. Evidence is
under `docs/figures/benchmark_artifacts/cycle015-17-prod-r5-20260611/`.

## Optional future hardening (not yet needed)

- Tie `StudentTrack` lifetime to box references with a DB `ON DELETE` trigger or
  a periodic reconcile, so orphans are structurally impossible.
- Apply the MOT `n_init` idea at persistence time: defer creating a track row
  until a track has ≥ N confirmed boxes, so transient single-frame tracks never
  get rows (only relevant if the upstream tracker emits 1-frame tracks).
