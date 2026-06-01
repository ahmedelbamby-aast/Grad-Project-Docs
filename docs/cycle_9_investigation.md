# Cycle 9 Investigation: Behavior Triton Ensemble

Date: 2026-06-01

## Problem Statement

The accepted Cycle 8 production run still misses the offline SLA on the
`combined.mp4` benchmark. The latest accepted baseline is:

| Metric | Cycle 8 Baseline |
|---|---:|
| Replay key | `cycle8-embed-bulk-crop-frame-20260601T125627` |
| Job id | `d2de80a0-31b7-4a47-b9f1-d2e2156ea3a8` |
| Video | `/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4` |
| Frames | `4541/4541` |
| DB-completed elapsed | `1312.34 s` / `21.87 min` |
| DB-completed FPS | `3.46` |
| Step 2 inference wall | `852.8 s` |
| Avg GPU utilization | `9.65 %` |
| Peak GPU utilization | `36 %` |
| Behavior model calls | `14391` app-level calls across 4 behavior/gaze models |

Cycle 8 removed most embedding/post-inference waste, so the dominant remaining
wall time is still the crop-frame inference stage. Cycle 9 is scoped to reducing
request fragmentation and redundant crop tensor upload for the four behavior
models only:

- `posture_model`
- `gaze_horizontal_model`
- `gaze_vertical_model`
- `gaze_depth_model`

`person_detector` and `rtmpose_model` are out of scope for this cycle because
their input/output contracts differ.

## Root Cause

The current crop-frame path sends the same `[N, 3, 320, 320]` crop batch to four
independent Triton requests. The accepted investigations show that the four
models share the same input tensor contract but return different YOLO-style
output tensors:

| Model | Input | Output |
|---|---|---|
| `posture_model` | `images: [N, 3, 320, 320]` | `output0: [N, 14, 2100]` |
| `gaze_horizontal_model` | `images: [N, 3, 320, 320]` | `output0: [N, 84, 2100]` |
| `gaze_vertical_model` | `images: [N, 3, 320, 320]` | `output0: [N, 14, 2100]` |
| `gaze_depth_model` | `images: [N, 3, 320, 320]` | `output0: [N, 14, 2100]` |

The current Python orchestration must therefore serialize, schedule, and receive
four behavior requests per crop batch even though the crop input tensor is
identical for all four models.

## Evidence

Accepted documentation and production telemetry establish the following:

- `docs/crop_frame_rtx5090_bottleneck_investigation.md` proved the original
  limiter was behavior/gaze inference wall and Python/gRPC boundary overhead,
  not decode, crop slicing, DB persistence, Triton readiness, GPU memory, or OOM.
- `docs/rtt_root_cause_investigation_77650001.md` refined the current ROI-320
  state: the old 192-212 ms 640-input behavior RTT datum is obsolete, but
  per-frame Python orchestration and four behavior requests remain material.
- `docs/cycles_9_to_12_implementation_playbook.md` identifies Cycle 9 as the
  next measured optimization: a Triton ensemble that accepts one crop batch and
  fans out server-side to the four behavior models.
- Cycle 8 telemetry for job `d2de80a0-31b7-4a47-b9f1-d2e2156ea3a8` recorded:

| Model | App-level calls | Avg RTT ms | P95 RTT ms |
|---|---:|---:|---:|
| `posture_model` | `3598` | `143.061` | `247.646` |
| `gaze_horizontal_model` | `3598` | `165.715` | `268.220` |
| `gaze_vertical_model` | `3598` | `153.429` | `251.813` |
| `gaze_depth_model` | `3597` | `167.950` | `276.889` |

The accepted Cycle 8 benchmark also shows Step 2 inference wall is still
`852.8 s`, much larger than render and persistence stages. This keeps Cycle 9
focused on behavior request count and crop input transfer, not post-processing.

## Hypothesis

Adding a Triton `behavior_ensemble` model with one `images` input and four named
outputs will reduce app-level behavior requests from four per crop batch to one.
The GPU compute is intentionally unchanged; this cycle targets boundary overhead,
Triton scheduling overhead, and redundant input upload.

Expected app-side request count change:

| Item | Before | After |
|---|---:|---:|
| Behavior requests per crop batch | `4` | `1` |
| Behavior crop input uploads per crop batch | `4` | `1` |
| App-level behavior route keys | `posture_detection`, `gaze_horizontal`, `gaze_vertical`, `gaze_depth` | `behavior_all` |

## Expected Gain

The execution playbook bounds the expected improvement because Cycle 8 already
runs the four behavior calls concurrently at the client side. The expected gain
is not a 4x inference speedup.

Target acceptance gain:

| Metric | Required Direction |
|---|---|
| Step 2 inference wall | At least `10 %` lower than Cycle 8 baseline |
| Overall DB-completed FPS | Measurable increase versus `3.46 FPS` |
| Behavior RTT overhead | Measurable decrease in app-level behavior call wall |
| GPU utilization | Measurable increase or no regression |
| Correctness | Detection/bbox/embedding parity preserved within playbook gates |

Projected range from the playbook:

- Step 2 wall: `852.8 s` -> `620-700 s`
- Total wall: `21.87 min` -> `19-20.5 min`
- Overall FPS gain: `+7 %` to `+16 %`

## Current-Code Verification

The current code still contains the Cycle 9 bottleneck:

- `backend/apps/video_analysis/tasks.py::_run_crop_behaviour_for_items` builds
  one crop payload list and then dispatches one independent request group per
  behavior task key.
- `backend/apps/pipeline/services/model_route_service.py` has no `behavior_all`
  route.
- `backend/apps/pipeline/services/triton_client.py` only requests `output0` for
  default YOLO-style models and has no multi-output behavior ensemble contract.
- `backend/models/triton_repository_cuda12/behavior_ensemble/config.pbtxt` does
  not exist.
- Production `backend/.env` currently does not include `behavior_ensemble` in
  `TRITON_LOAD_MODEL`, so app-side model allow-listing would fail closed until
  the production helper updates it.

Baseline-governance note: the production host currently has accepted Cycle 8
runtime values that are stricter than the checked-in helper in two places:
`TRITON_MAX_INFLIGHT_REQUESTS=32` is active on production while the helper still
contains `24`, and the accepted Cycle 8 result depends on track-level embedding
reuse being enabled. Cycle 9 validation must preserve accepted Cycle 8 behavior
while changing only the behavior ensemble dispatch path.

## Risk Assessment

| Risk | Level | Mitigation |
|---|---|---|
| Ensemble output mapping returns tensors in a different shape/name contract | Medium | Add a startup/config validator and a production output parity probe comparing ensemble outputs to standalone model outputs. |
| Triton model repository loads standalone models but rejects the ensemble config | Medium | Validate config before Triton start and fail fast when `TRITON_BEHAVIOR_ENSEMBLE=1`. |
| Missing `TRITON_LOAD_MODEL` allow-list entry blocks app calls | Low | Update production helper and default load-model list to include `behavior_ensemble`. |
| Ensemble path changes behavior decoding semantics | Medium | Split ensemble outputs locally back into the existing task-key shape and reuse existing `_decode_yolo_output0` logic. |
| Benchmark mixes Cycle 9 gain with unrelated env drift | Medium | Preserve the accepted Cycle 8 production profile before running the Cycle 9 benchmark. |

## Rollback Strategy

Rollback is one environment flag:

```bash
TRITON_BEHAVIOR_ENSEMBLE=0
```

When disabled, the existing four-call behavior dispatch path remains active. The
standalone `posture_model`, `gaze_horizontal_model`, `gaze_vertical_model`, and
`gaze_depth_model` stay in the Triton model repository and remain the fallback
execution path.

If Triton rejects the ensemble during startup, the startup validator must fail
before benchmark execution. If app runtime receives malformed ensemble outputs,
the crop-frame behavior path must fall back to the existing standalone dispatch
path rather than silently dropping detections.

## Acceptance Criteria

Cycle 9 can be marked `ACCEPTED` only after all of the following complete:

- Code implements `TRITON_BEHAVIOR_ENSEMBLE` behind a default-off flag.
- Triton model repository includes a valid `behavior_ensemble` config.
- App routing includes `behavior_all`.
- App-side ensemble response splitting preserves existing per-task decode logic.
- Unit tests cover route resolution, Triton IO contract, ensemble response
  splitting, config validation, and fallback behavior.
- Local validation passes.
- Production deployment reaches the pushed commit SHA.
- Production Triton loads `behavior_ensemble` and all four child models as
  `READY`.
- Production parity probe compares standalone outputs to ensemble outputs with
  max absolute difference `<= 1e-6`.
- Production benchmark runs on
  `/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4`.
- Benchmark documentation records replay key, job id, FPS, total wall time,
  inference wall time, GPU utilization, VRAM, RTT, model-call telemetry, DB
  parity, embedding parity, and detection parity.
- Step 2 inference wall improves by at least `10 %` versus Cycle 8 baseline and
  correctness parity remains within the Cycle 9 gates.

Until the production benchmark and documentation updates are complete, Cycle 9
status is `INCOMPLETE`.
