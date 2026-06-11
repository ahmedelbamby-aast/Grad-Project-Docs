# Cycle 015.4 Investigation: Bounded Temporal Evidence And Episode Explanations

**Created:** 2026-06-11
**Status:** `implemented_local; awaiting production benchmark`
**Streaming compatibility:** `stream-safe` (O(1) ingest, fixed-capacity ring
buffers, LRU-bounded scope state; no unbounded queries or arrays on the path)

## Problem

Anomaly scores are momentary. A reviewer needs to know whether a signal is a
transient spike or a sustained pattern, which signals *drove* an episode, and
whether the timeline is even trustworthy — all on an **indefinite live stream**
where memory must stay bounded forever. The existing semantic temporal services
(`apps/behavior/temporal/`) decay and smooth a single semantic-pose state; they
do not compose multivariate signal patterns or episode-level attribution, and
the online cache had no hard entry ceiling.

## Bounded-state contract

| Mechanism | Bound | Where |
|---|---|---|
| Per (student, signal) history | `ring_capacity` samples (`deque(maxlen)`) | `temporal_windows.BoundedMultivariateWindowEngine` |
| Distinct tracked students | `max_tracked_students` (LRU evict) | same engine, `OrderedDict` index |
| Online cache scopes | `max_entries` (LRU evict) | `temporal/cache.OnlineTemporalStateCache` |
| Window computation | one window pass over the ring (no re-scan on ingest) | `window_for` |

Ingest is a single `deque.append` plus an LRU touch — **O(1)**, no re-scan.
Memory is `O(max_tracked_students × signals × ring_capacity)` regardless of
stream length. The soak test (T097) feeds 50 000 samples across 5 000 tracks
into a 64-scope engine and asserts the ceiling holds with non-zero eviction.

## Pattern semantics (missing != zero)

| Pattern | Truth state | Rule |
|---|---|---|
| `cold_start` | `unknown` (0 valid) / `degraded` | total samples `< min_samples`, or valid `< min_samples` |
| `quarantined` | `suppressed` | contamination ratio `>= contamination_quarantine_ratio` (enough volume, too dirty) |
| `gap_dominated` | `degraded` | invalid continuity gaps cover `>= 50%` of the window span |
| `spike` | `valid` | exactly one robust outlier (median ± k·MAD; flat-baseline fallback to `>0`) |
| `drift` | `valid` | OLS slope magnitude `>= drift_min_slope_per_s` |
| `sustained` | `valid` | `>= sustained_ratio` of valid samples sit off-center by `> scale` |
| `steady` | `valid` | none of the above |

Ordering precedence (deliberate): **cold-start by volume → quarantine →
cold-start by valid coverage → gap → spike → drift → sustained → steady**. A
single bad sample is "not enough data" (cold-start), not "mostly contaminated"
(quarantine). Contaminated samples (None / missingness truth-state) are counted
and **excluded from the robust center** — they never shift the median.

## Episode contribution + counterfactual

An *episode* is a maximal run of consecutive actionable windows (sustained /
drift / spike) for one track, with no inter-window gap above
`max_episode_gap_ms`, of at least `min_episode_windows` windows.

- **Contribution share** = a signal's actionable-window count ÷ total actionable
  mass over the episode.
- **Counterfactual necessity**: hold a signal at baseline (remove its pattern
  from every window); a window stays actionable iff *some other* signal in it is
  actionable; the episode survives iff `>= min_episode_windows` windows remain.
  A signal whose removal collapses the episode is **necessary**; otherwise
  **redundant**. This is reported per signal, not summarized away.

## Determinism / replay

Window classification is order-independent (sorted by source time + ref) and
emits a `replay_digest`; episode digests chain their window digests. Same
sample multiset → identical classification, center, and digest (unit + soak
tests assert this, including export→rebuild after a simulated restart).

## Integration

`temporal/windows.ingest_temporal_observations` bridges the existing semantic
temporal observation stream into the new engine: each observation becomes a
`behavior.semantic_confidence` sample (numeric value only when `valid`; a
non-valid observation contributes its missingness, never a fabricated 0). No
upstream extraction is duplicated.

## Files

- `backend/apps/behavior/explainability/temporal_windows.py` (T091)
- `backend/apps/behavior/explainability/temporal_explanations.py` (T092)
- `backend/apps/behavior/explainability/episode_explanations.py` (T093)
- `backend/apps/behavior/temporal/windows.py` — bridge (T094)
- `backend/apps/behavior/temporal/cache.py` — LRU bound + reconnect (T095)
- `backend/tests/unit/behavior/test_xai_temporal_explanations.py` (T096)
- `backend/tests/integration/behavior/test_xai_temporal_bounded_state.py` (T097)

## Open acceptance gates (production, post-015.17)

- Temporal state/soak probe (T098): bounded-state holds under a live soak.
- Stride-1 benchmark (T099): ingest path adds no measurable per-frame latency.
- Figures + manifest/digests (T100); live soak helper (T101); rollback (T102);
  decision + ledger (T103/T104).
