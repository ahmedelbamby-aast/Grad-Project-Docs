# `cycle_8_embedding_stage`

**Last updated:** 2026-06-03
**Entity kind:** `cycle`
**Status:** `accepted`

> Three-prong attack on the embedding stage (the 450 s tail that
> Cycle 7's hypothesis falsification identified): (1) enable
> `OFFLINE_EMBEDDING_REUSE_BY_TRACK=1` so 53 unique tracks compute
> 53 embeddings instead of 72 579; (2) lazy `cv2.VideoCapture.grab`
> for forward-skip, only `.read()` on first-detection-per-track
> frames; (3) `persist_embeddings_bulk` with batch size 500 instead
> of per-row `objects.create()`. Accepted by production job
> `d2de80a0-31b7-4a47-b9f1-d2e2156ea3a8`: embedding wall 450.7 s →
> ~174 s (−61.5 %), total wall 1582 s → 1312 s (−17.1 %), overall
> FPS 2.87 → 3.46 (+20.5 %).

## Source-of-truth references

| Kind | Reference |
|---|---|
| Doc | `docs/crop_frame_optimization_execution.md` § Cycle 8 (lines 343-428) |
| Doc | `docs/production_inference_benchmark.md` (Cycle 8 row) |
| Doc | `docs/inference_parallelization_plan.md` (parent plan) |
| Doc | `docs/cycle_9_and_10_improvements_todo.md` § Z |
| Job | `d2de80a0-31b7-4a47-b9f1-d2e2156ea3a8` (accepted production benchmark) |
| Job | `515fe118-6009-4776-916d-6473fbf31ed7` (Cycle 7 reference baseline) |
| File | `backend/apps/tracking/embeddings.py` (new `persist_embeddings_bulk` helper) |
| File | `backend/apps/video_analysis/tasks.py` (lazy cv2 + track cache + pending-rows buffer in `generate_embeddings`) |
| File | `backend/config/settings/base.py` (new `EMBEDDING_BULK_BATCH_SIZE` + existing `OFFLINE_EMBEDDING_REUSE_BY_TRACK`) |
| File | `tools/prod/prod_enable_parallel_flow.sh` (sets both env knobs in prod) |
| File | `backend/tests/unit/tracking/test_persist_embeddings_bulk.py` (7 regression tests) |
| Workflow | `.github/workflows/inference-parallelization.yml` (gate updated to require the new test file) |
| Commit | `6b755f90` (DSP Cycle 4 prior entry — `cycle_7_redis_client_cache`) |
| Symbol | `apps.tracking.embeddings.extract_crop_embedding` (embeddings.py:83) |
| Symbol | `apps.tracking.embeddings.get_cached_job_track_embedding` (embeddings.py:257) |
| Symbol | `apps.tracking.embeddings.persist_embedding` (embeddings.py:282) |
| Symbol | `apps.tracking.embeddings.persist_embeddings_bulk` (embeddings.py:312) |
| Symbol | `apps.video_analysis.tasks.generate_embeddings` (tasks.py:5703) |

## 1. Purpose and scope

This cycle is the **real** embedding-stage attack the Cycle 7
falsification redirected toward. Three independent levers fold into
one commit:

1. **Track-level reuse.** `OFFLINE_EMBEDDING_REUSE_BY_TRACK=1` plus a
   process-local `track_vector_cache` dict means `model.embed()`
   runs once per unique `StudentTrack` (53 times on `combined.mp4`)
   instead of once per detection (72 579 times). The
   `FrameEmbedding`-per-`Detection` row contract is preserved
   exactly; only the *content* of the vector is now track-scoped,
   which is the semantic the rest of the system already assumes.
2. **Lazy cv2 frame read.** `cv2_cap.grab()` forward-skips between
   needed frames instead of `cv2_cap.set(...).read()` (which forces
   a keyframe re-decode). On the benchmark video, sequential read =
   0.32 ms/frame vs seek-read = 16.69 ms/frame.
3. **Bulk insert.** `persist_embeddings_bulk(rows, batch_size=500)`
   replaces ~72 k single-row `objects.create()` calls with ~145
   `bulk_create()` calls.

It does NOT touch Triton, the behavior models, the pose pipeline,
or the SLA gap toward 7.5 minutes that remains. Those become the
domain of Cycle 9 (behavior ensemble) and Cycle 10 (LPM / pose
parallelization).

## 2. Position in the system

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'primaryTextColor': '#EDE9FE', 'primaryBorderColor': '#A78BFA', 'lineColor': '#A78BFA', 'secondaryColor': '#1E3A8A', 'tertiaryColor': '#5B21B6', 'fontSize': '14px'}}}%%
flowchart LR
    C7["cycle_7_redis_client_cache<br/>(prior accepted)"] -->|"baseline 515fe118<br/>2.87 FPS"| E["this cycle<br/>(Cycle 8)"]
    L1["lever 1: track reuse<br/>(72k to 53 embeds)"] --> E
    L2["lever 2: lazy cv2.grab<br/>(no per-frame seek)"] --> E
    L3["lever 3: bulk_create<br/>(batch 500)"] --> E
    E -->|"prod bench<br/>combined.mp4"| Job["Job d2de80a0<br/>1312 s wall<br/>3.46 FPS"]
    E -->|"next attack:<br/>frame inference"| Next["Cycle 9<br/>(behavior ensemble)"]
    classDef prod fill:#5B21B6,stroke:#A78BFA,color:#EDE9FE;
    classDef ok fill:#16A34A,stroke:#86EFAC,color:#053B17;
    class E prod;
    class Job ok;
```

## 3. Internal structure (the three levers)

| Lever | File | Change |
|---|---|---|
| Track reuse | `embeddings.py:257` `get_cached_job_track_embedding` | Already existed; cycle defaults `OFFLINE_EMBEDDING_REUSE_BY_TRACK=1` |
| Track reuse | `tasks.py:5703` `generate_embeddings` | New process-local `track_vector_cache` dict; first detection per track computes via `extract_crop_embedding` (embeddings.py:83), all subsequent reuse |
| Lazy cv2 | `tasks.py:5703` `generate_embeddings` | `cv2_cap.grab()` for forward-skip; `.read()` only on needed frames |
| Bulk insert | `embeddings.py:312` `persist_embeddings_bulk` | New helper batching `FrameEmbedding.objects.bulk_create(rows, batch_size=500)`; mirrors `persist_embedding`'s dimension-coercion contract |
| Env wiring | `backend/config/settings/base.py:415,419` | `OFFLINE_EMBEDDING_REUSE_BY_TRACK` + `EMBEDDING_BULK_BATCH_SIZE` |
| Prod wiring | `tools/prod/prod_enable_parallel_flow.sh:63,73,155,156` | Both knobs set in offline + live profile blocks |
| Tests | `test_persist_embeddings_bulk.py` | 7 tests: batching at 500/1200/7/0/2, field contract, dimension coercion, batch_size kwarg pass-through |

## 4. Call graph (the new embedding loop)

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'primaryTextColor': '#EDE9FE', 'primaryBorderColor': '#A78BFA', 'lineColor': '#A78BFA', 'fontSize': '14px'}}}%%
flowchart TD
    GE["generate_embeddings<br/>(tasks.py:5703)"] -->|"per frame"| Need["frame has new<br/>track?"]
    Need -->|"no"| Grab["cv2_cap.grab()<br/>fast forward"]
    Need -->|"yes (first detection)"| Read["cv2_cap.read()<br/>+ extract_crop"]
    Read --> Cache["track_vector_<br/>cache miss"]
    Cache --> Embed["model.embed<br/>(once per track)"]
    Embed --> Store["track_vector_<br/>cache[track]=v"]
    Store --> Append["pending_rows.append"]
    Grab --> ReuseCheck["track in cache?"]
    ReuseCheck -->|"yes"| Reuse["copy cached v"]
    Reuse --> Append
    Append --> Flush["batch hits<br/>EMBEDDING_BULK_<br/>BATCH_SIZE"]
    Flush --> Bulk["persist_embeddings_<br/>bulk (embeddings.py:312)"]
    Bulk --> DB[("FrameEmbedding<br/>bulk_create batch=500")]
    classDef ok fill:#16A34A,stroke:#86EFAC,color:#053B17;
    classDef store fill:#1E3A8A,stroke:#60A5FA,color:#DBEAFE;
    class Grab,Reuse,Bulk ok;
    class DB store;
```

## 5. External connections

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'primaryTextColor': '#EDE9FE', 'primaryBorderColor': '#A78BFA', 'lineColor': '#A78BFA', 'fontSize': '14px'}}}%%
flowchart LR
    E["cycle_8_embedding_stage"]
    E -->|"72k to 53<br/>model.embed calls"| GPU["YOLO backbone<br/>(CPU + GPU)"]
    E -->|"4541 to ~53<br/>cv2 seeks"| VC["cv2.VideoCapture"]
    E -->|"72k single inserts<br/>to ~145 bulk calls"| PG[("PostgreSQL<br/>FrameEmbedding")]
    E -->|"7 new tests"| CI["CI workflow gate"]
    E -->|"bench evidence"| FS["data/videos/<br/>d2de80a0/"]
    classDef store fill:#1E3A8A,stroke:#60A5FA,color:#DBEAFE;
    class GPU,VC,PG,FS store;
```

## 6. API surface (env knobs)

| Variable | Default | Prod value | Effect |
|---|---|---|---|
| `OFFLINE_EMBEDDING_REUSE_BY_TRACK` | `0` | **`1`** | Enables track-level reuse via `get_cached_job_track_embedding` + process-local `track_vector_cache` |
| `EMBEDDING_BULK_BATCH_SIZE` | `500` | **`500`** | Batch size for `persist_embeddings_bulk`'s `bulk_create` calls |

Both knobs registered in `tools/prod/prod_enable_parallel_flow.sh` (lines 63, 73, 155-156).

## 7. Dependencies

| Dependency | Role |
|---|---|
| Cycle 7 (redis client cache) | Baseline reference (job `515fe118`) |
| `apps.tracking.embeddings` | Owns `extract_crop_embedding`, `get_cached_job_track_embedding`, `persist_embeddings_bulk`, `persist_embedding` |
| `apps.video_analysis.tasks` | Owns `generate_embeddings` |
| `cv2.VideoCapture` (OpenCV) | The video reader whose `set()`-then-`read()` was the per-frame seek tax |
| Django ORM `bulk_create` | The mechanism behind `persist_embeddings_bulk` |
| `tools/prod/prod_enable_parallel_flow.sh` | Sets both env knobs in prod profile |

## 8. Environment variables read

`OFFLINE_EMBEDDING_REUSE_BY_TRACK`, `EMBEDDING_BULK_BATCH_SIZE` — both via `backend/config/settings/base.py`.

## 9. Sequence diagram (one new-track first appearance vs reuse)

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'primaryTextColor': '#EDE9FE', 'actorBkg': '#5B21B6', 'actorBorder': '#A78BFA', 'actorTextColor': '#EDE9FE', 'signalColor': '#A78BFA', 'signalTextColor': '#EDE9FE', 'noteBkgColor': '#1E3A8A', 'noteTextColor': '#DBEAFE', 'noteBorderColor': '#60A5FA', 'fontSize': '14px'}}}%%
sequenceDiagram
    autonumber
    participant GE as generate_embeddings
    participant Cap as cv2.VideoCapture
    participant Mod as YOLO backbone
    participant TC as track_vector_cache
    participant Buf as pending_rows
    participant DB as PostgreSQL
    GE->>Cap: grab and skip 5 frames forward
    GE->>Cap: read frame f17 (first track 41 appearance)
    Cap-->>GE: numpy crop
    GE->>Mod: extract_crop_embedding
    Mod-->>GE: vector v
    GE->>TC: track_vector_cache 41 equals v
    GE->>Buf: append FrameEmbedding row
    GE->>GE: detection 2 in f17 already in cache
    GE->>TC: lookup track 41
    TC-->>GE: cached v reuse
    GE->>Buf: append row 2
    Buf->>DB: bulk_create batch reached 500
```

## 10. State machine

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'primaryTextColor': '#EDE9FE', 'primaryBorderColor': '#A78BFA', 'lineColor': '#A78BFA', 'fontSize': '14px'}}}%%
stateDiagram-v2
    [*] --> investigation: cv2 + bulk_create costs measured on prod RTX 5090
    investigation --> hypothesis: 3 levers projected -80 to -90 percent embedding wall
    hypothesis --> implementation: 3 code changes plus 2 env knobs plus 7 tests
    implementation --> prod_benchmark: d2de80a0 on RTX 5090
    prod_benchmark --> accepted: -61.5 percent embedding wall and zero correctness regression
    accepted --> sla_gap_remaining: still 14.36 min over 7.5 min SLA
    sla_gap_remaining --> [*]
```

## 11. Failure modes

| Considered | Why rejected |
|---|---|
| Embed once per session globally (not per track) | Would lose track-scoped semantics; ReID + attendance assume per-track vectors |
| Skip lazy cv2 (always sequential read) | Drops random-access ability needed for retry paths |
| `bulk_create` with `batch_size=10000` | Hits PostgreSQL parameter limits on FrameEmbedding's vector column |
| Drop the FrameEmbedding-per-Detection contract | Breaks ReID queries that assume one row per detection |

## 12. Performance characteristics (the bench)

| Metric | Cycle 7 `515fe118` | Cycle 8 `d2de80a0` | Δ vs Cycle 7 | Δ vs baseline `77650001` |
|---|---:|---:|---:|---:|
| Step 2 frame inference | 842.6 s | 852.8 s | +1.2 % (noise) | — |
| Pose post | 220.6 s | 220.7 s | unchanged (designed) | — |
| Persistence | 39.6 s | 39.4 s | unchanged | — |
| Render | 25.7 s | 25.7 s | unchanged | — |
| **Embedding** | **450.7 s** | **~174 s** | **−61.5 %** | — |
| **TOTAL DB completed** | **1 582.1 s** | **1 312.3 s** | **−17.1 %** | **−72.2 %** |
| **Overall FPS (DB-completed)** | **2.87** | **3.46** | **+20.5 %** | **+164 %** |
| Frames | 4 541 | 4 541 | parity | parity |
| Detections | 72 745 | 72 749 | ±9 (0.003 %) | ±2 vs baseline |
| Embeddings | 72 579 | 72 583 | ±9 (0.003 %) | ±2 vs baseline |
| **Distinct `model.embed()` calls** | ~72 579 | **~53** | **−99.93 %** | — |

Source: `docs/crop_frame_optimization_execution.md` § Cycle 8 Phase 4 (lines 394-415).

## 13. Operational notes

- The SLA gap toward 7.5 min stands at **14.36 min** after this
  cycle. The dominant remaining stage is Step 2 frame inference
  (~852 s ≈ 14.2 min); Cycle 9 targets this with a Triton ensemble
  and smaller behavior input.
- Track reuse is a **content-level** change but not a contract-
  level one. `FrameEmbedding.objects.count()` and per-detection
  joins remain identical; only the cosine distance between two
  detections of the same track is now exactly 0 (was numerically
  near-0 before). Downstream code does not branch on that.
- `persist_embeddings_bulk` deliberately does NOT trigger save
  hooks; `FrameEmbedding` has none (verified by grep) so this is
  safe. If a future migration adds one, this helper must be
  updated.

## 14. Historical diagrams

> Not applicable: no diagrams in this cycle doc have been
> superseded yet.

## 15. Related entities

| Entity | Path | Relationship |
|---|---|---|
| Cycle 7 (redis cache + falsification) | `docs/entity/cycles/cycle_7_redis_client_cache.md` | predecessor; its hypothesis falsification identified Cycle 8's three levers |
| Cycle 9 (behavior ensemble — NOT-ACCEPTED) | `docs/entity/cycles/cycle_9_behavior_ensemble.md` (planned next DSP commit) | successor; first cycle that did NOT clear the acceptance gate |
| Offline inference pipeline | `docs/entity/systems/offline_inference_pipeline.md` | the system this cycle optimised |
| `apps.tracking` | `docs/entity/modules/apps.tracking.md` | owns `embeddings.py` (new `persist_embeddings_bulk`) |
| `apps.video_analysis` | `docs/entity/modules/apps.video_analysis.md` | owns `generate_embeddings` (lazy cv2 + track cache) |
| `tools/prod/prod_enable_parallel_flow.sh` | (planned DSP Cycle 5) | the env switchboard that ships both runtime knobs |

## 16. Open questions

> All closed by the prod-bench acceptance on 2026-06-01. SLA gap
> tracking moves to Cycle 9 + Cycle 10.

## 17. Change log

| Date | What changed | Commit |
|---|---|---|
| 2026-06-01 | Cycle 8 accepted by production benchmark `d2de80a0` | (Cycle 8 implementation; doc field "Phase 3" shows `<pending push>` at write time but the change is observable on `main` via the prod env knobs in `prod_enable_parallel_flow.sh:63,73,155-156` and the `persist_embeddings_bulk` helper at `embeddings.py:312`) |
| 2026-06-03 | DSP Cycle 4 entry 4/N — entity doc consolidating the cycle. All 5 diagrams verified locally with `mmdc` per constitution § 19.3.1 before push. | (this commit) |
