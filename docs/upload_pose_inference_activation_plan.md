# Upload Pose Inference Activation (RTMPose + Artifacts + API + UI)

## Summary
Wire the real upload path (`video_analysis/tasks.py`) to run pose estimation on uploaded videos, persist pose artifacts, expose pose in API/WebSocket payloads, and render skeleton overlays in frontend playback.

Policy choices:
1. Output scope: **Artifact + API**
2. Failure policy: **Silent Skip**
3. Frame cadence: **Every frame**, with `.env` control

## Key Implementation Changes

### 1) Backend upload pipeline integration
- In the upload processing flow, after frame detections/tracking IDs are finalized, run pose inference per frame for person boxes.
- Use Triton `pose_estimation -> rtmpose_model` route for upload pose runtime.
- Add `POSE_UPLOAD_FRAME_STRIDE` env setting (default `1`) and honor it in upload pipeline.
- If pose inference errors, continue job completion (silent skip), but write structured warning evidence into job metadata/component failures and inference audit.

### 2) Triton pose request/response support
- Extend Triton request shaping to support task-specific I/O contract for pose:
  - input name: `input`
  - outputs: `simcc_x`, `simcc_y`
  - tensor shape: `[1, 3, 256, 192]`
- Add SimCC decode utility to convert `simcc_x/simcc_y` into per-joint `(x, y, confidence)` keypoints in ROI space, then map back to source-frame coordinates with existing RTMPose mapping utilities.

### 3) Pose artifact persistence (no DB migration)
- Persist upload pose sidecar artifact in job directory: `pose_results.json`.
- Store per-frame pose records keyed by frame index and tracking ID (bbox, keypoints, scores, timestamps).
- Add metadata pointers and summary fields on job (`pose_artifact_path`, pose frame count, pose runtime provider/device, pose latency/fps summaries).

### 4) API/WebSocket contract additions
- Extend frame payloads from job/frame endpoints to include `pose_records` when available.
- Extend `overlay.frame` events with `poses` array alongside existing `boxes`.
- Keep existing `boxes` contract unchanged for backward compatibility.

### 5) Frontend playback rendering
- Extend TS types and live hooks to parse/store `pose_records` + websocket `poses`.
- Add skeleton/keypoint rendering on overlay canvas, controlled by existing `pose_estimation` visibility toggle.
- Preserve current bounding-box behavior; pose overlay is additive.

### 6) Visibility and model-toggle correctness
- Add `pose_estimation` to backend visibility defaults and enabled-model extraction so upload pipeline can actually activate pose from user toggles.
- Ensure pose toggle state is persisted/read consistently across Redis + metadata paths.

## Public Interface / Type Changes
- `VideoAnalysisFrame` response gains optional `pose_records`.
- `overlay.frame` WebSocket payload gains optional `poses`.
- Job metadata gains pose runtime/artifact summary fields.
- New env var: `POSE_UPLOAD_FRAME_STRIDE` (default `1`).

## Test Plan

### 1) Unit tests
- SimCC decode correctness (shape/value bounds/confidence extraction).
- Frame-stride behavior (`POSE_UPLOAD_FRAME_STRIDE`).
- Pose failure handling path with silent-skip metadata/audit output.

### 2) Integration tests
- Upload workflow with mocked Triton pose responses writes `pose_results.json` and exposes pose records via frame endpoints.
- Upload workflow with pose runtime failure completes without hard-fail and records skip reason.
- Visibility toggle test verifies `pose_estimation` affects returned overlays.

### 3) Frontend tests
- Hook/type parsing tests for `pose_records` and websocket `poses`.
- Overlay canvas test verifies skeleton drawing appears/disappears with `pose_estimation` toggle.

## Assumptions
- Triton `rtmpose_model` metadata currently exposes:
  - input `input` with `[N,3,256,192]`
  - outputs `simcc_x`, `simcc_y`
- Pose persistence will be file-based sidecar (`pose_results.json`) to avoid schema migration and keep rollout fast.
- “Silent Skip” means no job failure on pose errors, but explicit telemetry/audit evidence is still recorded.
