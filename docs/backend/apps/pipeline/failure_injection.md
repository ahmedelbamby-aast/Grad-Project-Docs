# backend/apps/pipeline/failure_injection.py

## Source
- [backend/apps/pipeline/failure_injection.py](../../../../../backend/apps/pipeline/failure_injection.py)

## Purpose
Provides deterministic, thread-safe failure injection checkpoints for pipeline stage testing.

## Main structures and functions
- `StageFailureRule`: declarative stage rule with cadence and max trigger count.
- `StageCheckpointAuditEvent`: immutable checkpoint/trigger audit record.
- `FailureInjectionController`: rule/hook registry with checkpoint evaluation and audit history.
- `NoopFailureInjectionController`: production-safe no-op implementation.

## Behavior notes
- Every checkpoint increments hit counters and appends an audit event.
- Matching rules can raise stage-specific exceptions with controlled frequency.
- Snapshot methods return stable counter and audit views for diagnostics.

## Cross-links
- [buffering.md](buffering.md)
- [reconciliation.md](reconciliation.md)
