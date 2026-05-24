# Spec 008 Staged Load Validation (Simulated Harness)

Generated UTC: 2026-05-21T10:17:42.799360+00:00

## Before vs After

| Stage | Throughput Before (jobs/min) | Throughput After (jobs/min) | Upload P95 Before (ms) | Upload P95 After (ms) | Non-upload Queue P95 Before (ms) | Non-upload Queue P95 After (ms) | Starvation Before | Starvation After |
|---|---:|---:|---:|---:|---:|---:|---|---|
| low | 70.33 | 70.33 | 679.86 | 413.56 | 71.62 | 0.00 | FALSE | FALSE |
| medium | 149.67 | 149.67 | 770.10 | 420.62 | 237.28 | 40.72 | FALSE | FALSE |
| high | 259.33 | 259.33 | 1314.80 | 513.98 | 738.40 | 39.22 | FALSE | FALSE |

## Notes

- This harness is deterministic and infra-independent; use it when Triton/Celery/Redis/GPU infra is unavailable.
- Replace with real endpoint load results when full environment is available.
