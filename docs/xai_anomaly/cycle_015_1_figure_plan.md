# Cycle 015.1 Figure Plan

**Figure Planner:** Claude Cycle 015.1 Figure Planner (named 2026-06-11)

Planned figures (from the same raw artifacts as the decision tables):
1. signal-registry coverage (registered/enabled/fast-explainer counts vs active routes);
2. envelope payload-size distribution vs `XAI_EVIDENCE_MAX_PAYLOAD_BYTES`;
3. missingness-state distribution (unknown/unavailable/degraded/valid);
4. normalization + persist latency percentiles;
5. unavailable-metric summary.

Unavailable-metric policy: every absent input renders a placeholder with the
explicit reason; nothing is plotted as zero. Markdown embed target:
`docs/xai_anomaly/cycle_015_1_results.md`.
