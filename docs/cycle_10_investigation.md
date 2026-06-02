# Cycle 10 Investigation: LPM Phase 1 Hook

**Last updated:** 2026-06-01

**Status:** Phase A/B record before code changes. This is not accepted until a
fresh production benchmark satisfies `docs/logical_path_matrix_spec.md` §10.

## Problem Statement

Cycle 10 Logical Path Matrix (LPM) math exists in
`backend/apps/pipeline/services/logical_path_matrix.py`, with unit coverage for
C1-C4, disabled identity, and exclusion rules. It is still not active in the
offline crop-frame path: `backend/apps/video_analysis/tasks.py` decodes
posture/gaze outputs inside `_run_crop_behaviour_for_items()` and appends every
decoded `DetectionBox` directly to `fd.boxes`.

The missing C.2.1 work is the production hook: after the crop behavior/gaze
models decode for a frame, LPM must be able to filter only the three gaze axes
before persistence, while leaving `person_detection`, `rtmpose_model`, and
`sitting_standing` untouched.

## Root Cause

Current `crop_frame` behavior dispatch has no per-frame constraint step between
gaze decode and persistence. The existing module is isolated and default-off,
but no caller threads its prior state or applies it to newly emitted behavior
boxes.

## Evidence

- `docs/logical_path_matrix_spec.md` §4.1 places Phase 1 inside
  `_run_crop_behaviour_for_items()` after behavior/gaze decode and before
  `fd.boxes` becomes the persistence source.
- `backend/apps/video_analysis/tasks.py` currently appends each crop behavior
  `DetectionBox` directly in `_run_crop_behaviour_for_items()`.
- `backend/apps/pipeline/services/logical_path_matrix.py` already exposes
  `apply_to_detection_boxes(...)`, which is default-off identity when
  `LpmConfig.enabled=False` and structurally excludes non-gaze outputs.
- `.github/workflows/inference-parallelization.yml` already includes the staged
  LPM module and unit test, but it does not yet gate an integration test for the
  `tasks.py` hook.

## Hypothesis

Hooking the existing LPM helper into `_run_crop_behaviour_for_items()` behind
`LPM_ENABLED` will make the offline crop-frame path capable of enforcing the
staged C1-C4 gaze constraints without changing baseline behavior when the flag
is disabled.

**Named lever:** single-process Python orchestration.

This phase is primarily correctness/staging work, not a throughput optimization.
The lever is named because the added work lives on the Python post-decode path:
the acceptance target is no Step 2 regression above 1%, LPM latency below
5 ms/frame, and measurable contradiction/gaze-flip reduction once the prod
benchmark runs.

## Expected Gain

- Disabled baseline: byte-identical behavior for `fd.boxes` and row counts.
- Enabled candidate: ambiguous or contradictory gaze-axis boxes are suppressed
  per tracked person; non-gaze detections pass through unchanged.
- Performance expectation: neutral, with CPU overhead below the spec budget of
  5 ms/frame.

## Risk Assessment

Risk is low-to-medium:

- Low because `LPM_ENABLED=0` remains the production default and rollback is one
  env flag.
- Medium because enabled LPM can change attention/gaze boxes. The hook must not
  alter `person_detection`, `sitting_standing`, hand-raising, or pose outputs.
- Current Phase 1 uses post-decode box confidences through the existing helper;
  pre-argmax probability telemetry remains future work. Aggregate
  `telemetry_lpm_events` storage is implemented in C.2.2 and remains pending
  production migration/benchmark evidence.

## Rollback Strategy

Set:

```bash
LPM_ENABLED=0
```

and restart the offline workers. If the hook itself causes a fault, revert the
small `tasks.py` integration change; the standalone LPM module remains safe and
unused.

## Acceptance Criteria

Local C.2.1 staging acceptance:

- `LPM_ENABLED=0` integration test proves crop-frame behavior output is
  unchanged and no LPM helper is invoked.
- `LPM_ENABLED=1` integration test proves unambiguous gaze inputs keep one
  winner per axis and non-gaze counts are preserved.
- `.github/workflows/inference-parallelization.yml` gates the new integration
  test and the docs touched by this cycle.

Production acceptance remains deferred until C.2.3:

- Fresh Linux RTX 5090 production benchmark on `combined.mp4`.
- `docs/logical_path_matrix_spec.md` §10 gates pass.
- Metrics are recorded in `docs/cycle_10_lpm_phase1_results.md`.

## C.2.2 Telemetry Addendum

**Problem statement.** The LPM hook can reduce contradictory gaze boxes, but a
production acceptance gate cannot be computed from logs alone. The benchmark
needs durable per-frame counters for persons seen, contradictions eliminated,
C1-C4 violations, and LPM latency.

**Root cause.** Before C.2.2, `TelemetrySession` only collected frames, model
calls, videos, students, and session summaries. There was no schema or writer
path for LPM-specific events.

**Evidence.** `docs/logical_path_matrix_spec.md` §8 defines
`telemetry_lpm_events`, and §10.2 requires C3, C1, and eliminated-contradiction
counts from that table.

**Expected gain.** No throughput gain is claimed. This creates the measurement
surface needed for the C.2.3 production benchmark and avoids log scraping.

**Risk assessment.** Low. The migration is additive, `LPM_ENABLED=0` remains
the default, and telemetry errors are swallowed so inference cannot fail due to
LPM event recording.

**Rollback strategy.** Set `LPM_ENABLED=0` to stop event generation. If the
table migration causes production drift, revert migration deployment before
enabling LPM; existing telemetry tables are not modified.

**Acceptance criteria.**

- `TelemetryLpmEvent` model and migration exist.
- `TelemetrySession.record_lpm_event(...)` persists through JSON fallback and
  PostgreSQL bulk insert.
- The crop-frame LPM hook records one event when LPM runs.
- Focused telemetry/LPM tests pass locally.
- Production acceptance remains blocked until migration + benchmark evidence
  are recorded in `docs/cycle_10_lpm_phase1_results.md`.
