# backend/apps/pipeline/buffering.py

## Source
- [backend/apps/pipeline/buffering.py](../../../../../backend/apps/pipeline/buffering.py)

## Purpose
Tiered buffering primitives that preserve order across RAM and disk spill queues with deterministic overflow handling.

## Main structures and functions
- `BufferFrame`: frame envelope tracked across tiers.
- `OverflowDecision`: normalized result for queue/spill/block/drop outcomes.
- `DrainRecord`: drained frame plus source tier (`ram` or `spill`).
- `RamFrameQueue`: bounded FIFO queue with max-frames and max-bytes limits.
- `DiskSpillQueue`: bounded spill queue persisted as JSONL with recovery support.
- `SpillDrainOrchestrator`: deterministic global-order drain coordinator.
- `apply_overflow_policy(...)`: policy executor for `block`, `drop_oldest`, and `drop_newest`.

## Contracts mirrored for this feature
- RAM-first acceptance, then spill tier.
- Overflow policy decisions are explicit and auditable.
- Spill recovery rebuilds queue index from persisted records.
- Drain ordering uses lowest sequence first (RAM tie-breaker).

## Cross-links
- [runtime_ingestion.md](runtime_ingestion.md)
- [failure_injection.md](failure_injection.md)
- [reconciliation.md](reconciliation.md)
