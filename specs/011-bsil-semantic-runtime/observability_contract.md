# BSIL Observability Contract

## Required Telemetry

- semantic inference latency;
- temporal aggregation latency;
- semantic and behavioral queue depth, age, retries, timeouts and DLQ counts;
- behavioral state transition counts and transition reasons;
- episode creation, extension, closure, suppression and supersession counts;
- interaction graph node/edge counts, edge confidence and suppression reasons;
- confidence distribution metrics by semantic type and runtime mode;
- false escalation, false-positive and false-negative review metrics;
- uncertainty reason counts;
- dropped-frame and dropped-event attribution;
- baseline drift and adaptive threshold changes;
- runtime reconciliation mismatch counts.

## Required Dashboards

| Dashboard | Minimum panels |
| --- | --- |
| Behavioral runtime | mode, admission, queue health, semantic latency, aggregation latency |
| Temporal state | active states, transitions, decays, invalid windows, cooldowns |
| Interaction graph | active identities, edges, confidence, suppressions, crowded-scene metrics |
| Semantic confidence | confidence distributions, uncertainty reasons, occlusion impact |
| Evidence and reconciliation | artifacts, digests, task/DB/queue convergence, stale/degraded states |
| Confidence bands | advisory, review-needed, high-confidence, degraded and invalid counts by output type and runtime mode |
| Review workflow | high-confidence episode review tasks, reviewer labels, false-positive/false-negative trends and access-audit volume |

## Alerting Rules

- Any silent fallback attempt is critical.
- Runtime reconciliation mismatch is blocking for maturity closure.
- Queue budget breach creates degraded runtime state.
- Confidence decomposition absence blocks accepted behavioral output.
- Baseline drift budget breach freezes adaptive anomaly maturity claims.
- Confidence band contradiction between API, WebSocket, UI or evidence export
  blocks acceptance for the affected output scope.
