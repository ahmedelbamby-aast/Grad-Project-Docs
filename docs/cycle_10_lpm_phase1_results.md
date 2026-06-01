# Cycle 10 LPM Phase 1 Results

**Status:** **NEEDS FURTHER ITERATION — NOT ACCEPTED.**

Cycle 10 C.2.1/C.2.2 code was deployed to production at SHA
`377fb59e66fe46327e6cbf2e6e64fd3bb70a0bde`, migration
`telemetry.0002_telemetrylpmevent` was applied, and one LPM-enabled production
benchmark was executed on the Linux RTX 5090 server.

The run completed, but it failed the LPM acceptance gates:

- `telemetry_lpm_events` populated correctly (`4541` rows), proving the new
  telemetry path works.
- LPM did not detect or eliminate contradictions (`eliminated=0`, `C1=0`).
- `attention_tracking` boxes dropped from Cycle 9's `11776` to `2680`, a
  correctness regression.
- Production was rolled back to `LPM_ENABLED=0` after the benchmark.

## Evidence

| Item | Value |
|---|---|
| Replay key | `cycle10-lpm-crop-frame-20260601T201239` |
| Job ID | `17075418-4386-4b5f-85d4-ea23bec71f66` |
| Candidate SHA | `377fb59e66fe46327e6cbf2e6e64fd3bb70a0bde` |
| Telemetry session | `59c9e8bb-514f-41b4-a21a-ebda3a070a7d` |
| Bench summary | `backend/logs/bench_summary_20260601T231516.json` |
| Bench log | `backend/logs/parallel_flow_cycle10-lpm-crop-frame-20260601T201239.log` |
| GPU CSV | `backend/logs/gpu_monitor_bench_20260601T231516.csv` |
| Final status | `completed` |
| Rollback | `LPM_ENABLED=0`; workers restarted |

## Metric Comparison

| Metric | Cycle 9 Candidate | Cycle 10 LPM Run | Delta |
|---|---:|---:|---:|
| DB-completed elapsed | `1110.7 s` | `1076 s` | `-34.7 s` / `-3.1 %` |
| Overall FPS | `4.09` | `4.219` | `+3.2 %` |
| Telemetry session wall | `923.1 s` | `908.857 s` | `-14.2 s` / `-1.5 %` |
| App-level model calls | `9557` | `9557` | unchanged |
| Avg GPU utilization | `9.36 %` | `9.0 %` | `-0.36 pp` |
| Peak GPU utilization | `43 %` | `39 %` | `-4 pp` |
| Frame rows | `4541` | `4541` | 0 |
| Detection rows | `72749` | `63653` | `-9096` / `-12.5 %` |
| Bounding boxes | `72749` | `63653` | `-9096` / `-12.5 %` |
| Frame embeddings | `72583` | `63487` | `-9096` / `-12.5 %` |
| Student tracks | `53` | `53` | 0 |
| `attention_tracking` boxes | `11776` | `2680` | `-9096` / `-77.2 %` |
| `hand_raising` boxes | `8800` | `8803` | `+3` |
| `person_detection` boxes | `19162` | `19162` | 0 |
| `sitting_standing` boxes | `33011` | `33008` | `-3` |

## LPM Telemetry

| Metric | Value |
|---|---:|
| LPM event rows | `4541` |
| Total persons seen by LPM | `10587` |
| Eliminated contradictions | `0` |
| C1 violations | `0` |
| C2 violations | `0` |
| C3 violations | `0` |
| C4 violations | `0` |
| Frames with eliminated contradictions | `0` |
| Average LPM latency | `0.024 ms` |
| Max LPM latency | `0.272 ms` |

## Decision

**NEEDS FURTHER ITERATION — NOT ACCEPTED.**

The telemetry table and writer are validated in production, but the LPM behavior
failed two required acceptance conditions:

1. The contradiction-reduction signal did not appear (`C1=0`,
   `eliminated=0`).
2. Correctness regressed because attention boxes were suppressed without a
   recorded C1-C4 violation.

## Root Cause From This Run

The production run exposed a Phase 1 post-decode integration bug. With only one
directional box on an axis, `apply_to_detection_boxes(...)` synthesized UNKNOWN
probability as residual mass. When the directional confidence was below that
residual, the axis resolved to `UNKNOWN`, and the reconstruction step dropped
the model-emitted box even though no C1-C4 violation fired.

That is why `attention_tracking` dropped sharply while all LPM violation
counters remained zero.

## Rollback

Production was rolled back immediately after the benchmark:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_enable_parallel_flow.sh --profile per-frame-signals
bash tools/prod/prod_stop_celery_workers.sh
bash tools/prod/prod_start_celery_workers.sh
grep '^LPM_ENABLED=' backend/.env
```

Observed result:

```text
LPM_ENABLED=0
```

## Follow-Up Fix

A local safety fix now preserves model-emitted single-axis gaze boxes when no
C1-C4 violation fired. The new regression test is:

```text
backend/tests/unit/pipeline/test_logical_path_matrix.py::TestApplyToDetectionBoxes::test_single_low_confidence_direction_is_not_suppressed_without_violation
```

Focused local validation after the fix:

```text
65 passed
```

This fix still needs deployment and a fresh production benchmark before it can
change Cycle 10's decision.
