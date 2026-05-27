# BSIL Evidence Contract

## Required Evidence Classes

| Evidence class | Minimum content | Blocks acceptance when missing |
| --- | --- | --- |
| Offline classroom runs | at least three real datasets, source digests, replay metadata, semantic traces, episodes and graph exports | offline behavioral maturity |
| Live RTSP runs | at least two real datasets, reconnect evidence, queue behavior, latency and degradation state | live behavioral maturity |
| Crowded interaction runs | multi-person identity continuity, interaction edges, suppression reasons | interaction maturity |
| Occlusion/re-entry runs | identity uncertainty, recovery state, invalid windows | temporal/identity maturity |
| Long-duration sessions | state decay, queue growth, memory/backlog, drift and SLO evidence | operational stability |
| Behavioral exports | episodes, decision traces, confidence breakdown, non-accusatory labels | reviewer readiness |
| Runtime causality exports | task, queue, database, telemetry, artifact and frontend reconciliation | production readiness |
| Queue pressure exports | backlog, age, retry, timeout, DLQ and dropped/deferred work | resilience maturity |
| GPU attribution | utilization, memory, batch, latency and model/runtime attribution where GPU is used | performance claim |

## Artifact Rules

- Evidence MUST be immutable after publication and superseded by a new snapshot
  when corrected.
- Every artifact MUST include schema version, producing command, code revision,
  runtime fingerprint, dataset/source reference, digest and generated timestamp.
- Evidence MUST distinguish mock/real, CPU/GPU, synthetic/production, dev/prod
  and fallback/canonical execution.
- Placeholder-only, temporary-path-only, stale, dev-only-for-production,
  SQLite-backed or fallback-generated evidence is rejected.

