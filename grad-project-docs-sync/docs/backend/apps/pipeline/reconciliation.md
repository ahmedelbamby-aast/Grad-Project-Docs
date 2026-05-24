# backend/apps/pipeline/reconciliation.py

## Source
- [backend/apps/pipeline/reconciliation.py](../../../../../backend/apps/pipeline/reconciliation.py)

## Purpose
Frame-level reconciliation helpers for pose/object/association completeness and threshold gates.

## Main structures and functions
- `ReconciliationThresholds`: gate thresholds for pose coverage, unresolved frames, duplicates.
- `FrameReconciliationSummary`: summarized decoded/missing/gap counters.
- `PoseGapVerdict`: threshold verdict with reason codes.
- `summarize_run_reconciliation_counters(...)`: run-level monotonic counter rollup.
- `reconcile_frame_artifacts(...)`: computes missing/unresolved/duplicate and buffering counters.
- `evaluate_reconciliation_threshold_gates(...)`: pass/fail decision for reconciliation thresholds.
- `summarize_frame_reconciliation(...)`: decoded-frame pose gap summary.
- `evaluate_pose_gap_thresholds(...)`: ratio + contiguous-gap threshold enforcement.

## Feature alignment
- Includes buffered/spilled/dropped/delayed frame counters in final reconciliation output.
- Emits schema marker `frame-reconciliation.v2` for downstream consumers.

## Cross-links
- [buffering.md](buffering.md)
- [runtime_ingestion.md](runtime_ingestion.md)
