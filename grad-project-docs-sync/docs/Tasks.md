# Consolidated Implementation Tasks

This file consolidates execution tasks from:
- `docs/identity_consistency_frontend_backend_plan.md`
- `docs/upload_pose_inference_activation_plan.md`
- `docs/pose_quality_occlusion_per_student_stability_plan.md`

## Delivery Rules

- Execute in phases.
- For each phase: spawn multiple agents, distribute tasks, integrate, run tests, iterate until green.
- Close all phase agents before spawning next phase agents.
- Follow `AGENTS.md` parallel testing policy.

## Phase 0 - Baseline and Safety Rails

- [ ] Capture baseline:
  - [ ] `git status` snapshot
  - [ ] existing failing/passing targeted backend/frontend test status
- [ ] Identify touched modules for:
  - [ ] upload pipeline
  - [ ] live stream pipeline
  - [ ] websocket payload builders
  - [ ] frontend overlay stores/hooks/canvas
- [ ] Create implementation log section in PR notes (for traceability)

Acceptance:
- Baseline test commands and current outcomes documented.

## Phase 1 - Backend Canonical Identity and Idempotency

- [ ] Add canonical entity structures and builders:
  - [ ] canonical entity keying (`entity_id` + `identity_scope/tracking_id` fallback)
  - [ ] model evidence aggregation into one canonical entity per tracking identity
- [ ] Make persistence/emission idempotent:
  - [ ] duplicate `event_id` protection
  - [ ] frame upsert semantics (`camera_id + frame_number` and/or `event_id`)
  - [ ] duplicate-safe `StudentTrack` resolver behavior
- [ ] Extend payload contracts:
  - [ ] websocket `overlay.frame.entities[]` + `event_id` metadata
  - [ ] keep backward-compatible `boxes[]`
- [ ] Add backend debug instrumentation:
  - [ ] canonical create/update/end logs
  - [ ] duplicate evidence merge logs
  - [ ] invariant checks in debug mode

Acceptance:
- One logical student per tracking identity in canonical payloads.
- Replayed/duplicate events do not create duplicate logical entities.

## Phase 2 - Upload Pose Inference Activation + Artifacts

- [ ] Integrate RTMPose in upload pipeline after tracking finalization.
- [ ] Add env config: `POSE_UPLOAD_FRAME_STRIDE` (default `1`) and enforce cadence.
- [ ] Implement silent-skip error policy with structured telemetry/audit evidence.
- [ ] Implement Triton pose I/O shaping:
  - [ ] input `input`
  - [ ] outputs `simcc_x`, `simcc_y`
  - [ ] expected tensor shape handling
- [ ] Add SimCC decode + ROI-to-frame mapping integration.
- [ ] Persist upload artifacts:
  - [ ] `pose_results.json`
  - [ ] job metadata pointers and summary stats
- [ ] Extend frame/API/websocket payloads with pose (`pose_records` / `poses`) while preserving existing contracts.
- [ ] Ensure pose visibility toggle (`pose_estimation`) is correctly honored end-to-end.

Acceptance:
- Upload jobs can produce pose artifacts and pose payloads.
- Pose failures do not fail upload job completion; warning evidence is present.

## Phase 3 - Shared Pose Quality, Occlusion, Stability (Offline + Live)

- [ ] Create shared pose quality service used by offline and live flows.
- [ ] Implement quality metrics and labels:
  - [ ] joint visibility ratios
  - [ ] confidence summaries
  - [ ] lower-body occlusion window logic
  - [ ] temporal stability scoring
  - [ ] `usable_for_overlay` / `usable_for_behavior_analysis`
- [ ] Add display-only smoothing (`display_keypoints`) without mutating raw keypoints.
- [ ] Extend offline artifacts:
  - [ ] augment `pose_results.json` with `quality`
  - [ ] add `pose_per_student.json`
  - [ ] add `pose_quality_summary.json`
- [ ] Extend live path:
  - [ ] apply per-tracking-id quality evaluation
  - [ ] persist live pose quality artifacts
  - [ ] include `poses[].quality` (backward compatible) in websocket payloads
- [ ] Extend metadata/API:
  - [ ] pose quality artifact pointers
  - [ ] optional per-student pose endpoint

Acceptance:
- Quality and occlusion are computed per student (`tracking_id`) in both upload and live.
- Lower-body occlusion does not suppress usable upper-body overlays.

## Phase 4 - Frontend Canonical Entities + Pose Rendering Behavior

- [ ] Update frontend types:
  - [ ] canonical entities
  - [ ] pose records and pose quality fields
- [ ] Update live/session hooks and stores:
  - [ ] normalize entities by canonical key
  - [ ] stale frame / duplicate event rejection
  - [ ] websocket generation token reconnection guard
  - [ ] reset state on job/session/camera changes
- [ ] Update rendering:
  - [ ] student cards from canonical entities (not raw boxes)
  - [ ] one primary bbox per canonical entity
  - [ ] optional model evidence overlays
  - [ ] skeleton drawing gated by keypoint confidence and visibility toggle
  - [ ] upper-body-only and unstable visual handling
- [ ] Add dev warnings:
  - [ ] duplicate canonical keys
  - [ ] stale/duplicate event drops
  - [ ] raw boxes > canonical entities mismatch warnings

Acceptance:
- `ONE TRACK ID == ONE UI ENTITY` in rendered results.
- Pose overlay remains additive and toggle-controlled.

## Phase 5 - Tests, Regression, and Iteration to Green

Backend unit tests:
- [ ] canonical entity aggregation behavior
- [ ] duplicate-safe persistence/idempotency
- [ ] SimCC decode correctness
- [ ] upload stride behavior
- [ ] silent-skip telemetry behavior
- [ ] pose quality metrics, occlusion windows, stability scoring

Backend integration tests:
- [ ] detection/tracking/canonical/websocket one-entity-per-track behavior
- [ ] repeated offline/live updates are upsert/replace, not logical duplicates
- [ ] upload pose artifacts and frame payload exposure
- [ ] live websocket includes `poses[].quality`

Frontend tests:
- [ ] duplicate websocket packets do not duplicate entities
- [ ] canonical cards render once per student identity
- [ ] pose draw/hide by confidence and `pose_estimation` toggle
- [ ] upper-body-only rendering remains visible
- [ ] stale reconnect stream prevention

Execution gates:
- [ ] targeted backend suites pass
- [ ] targeted frontend suites pass
- [ ] reproduction scenario pass (upload + live replay/reconnect)

## Phase Agent Strategy

Per phase:
1. Spawn multiple agents with disjoint ownership.
2. Parallel implementation and/or tests.
3. Integrate and resolve conflicts.
4. Run phase test gate, iterate until pass.
5. Close all agents.
6. Spawn fresh agents for next phase.

## Done Criteria

- All phase acceptance criteria checked.
- All targeted tests for changed surfaces pass.
- No duplicate student identity regression in upload/live UI and payloads.
- Pose inference activation, quality, and artifacts are present and backward compatible.
