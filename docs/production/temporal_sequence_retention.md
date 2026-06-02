# Temporal Sequence Retention Runbook

**Last updated:** 2026-05-25

## Policy
- Raw temporal sequence records are retained.
- Physical delete is forbidden during maturity closure.
- Retention actions are `soft_purge`, `archive`, and `restore`.

## Execution
1. Confirm actor authorization in production dashboard.
2. Record action scope (session/camera/canonical track).
3. Write tombstone and recovery reference metadata.
4. Export evidence impact report to `ci_evidence/production/wave5/sequence_retention_audit.json`.

## Validation
- Ensure recoverability of soft-purged rows.
- Ensure actor/reason/timestamp are immutable in audit entries.

