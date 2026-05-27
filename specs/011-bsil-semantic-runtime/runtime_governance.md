# BSIL Runtime Governance

## Runtime Authority

BSIL preserves the existing runtime constitution. Governed production model
inference remains Triton-authoritative. Lightweight semantic heuristics are
allowed only when explicitly labeled as heuristic, lineage-backed,
confidence-scored and excluded from scientific model maturity claims until
validated.

## Queue Families

| Workload | Live authority | Offline authority |
| --- | --- | --- |
| Semantic pose inference | live semantic queue | offline semantic queue |
| Behavioral aggregation | live behavior aggregation queue | offline behavior aggregation queue |
| Interaction reasoning | live interaction queue | offline interaction queue |
| Evidence export | live evidence queue | offline evidence queue |

Live workers MUST NOT consume offline semantic or behavioral queues during
production operation. Offline workers MUST NOT consume live queues during
production operation.

## Runtime State Requirements

- Semantic state cache and interaction graph cache are runtime accelerators,
  not durable authority.
- PostgreSQL remains authoritative for durable behavioral state, episodes,
  lineage, benchmark and acceptance evidence.
- Every work item MUST carry correlation ID, runtime mode, queue name, source
  window, identity scope, retry attempt and terminal outcome.
- Every degraded, fallback, timeout, dropped frame or rejected interval MUST
  emit telemetry and lineage.

