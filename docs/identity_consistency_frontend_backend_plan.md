**Last updated:** 2026-05-22

## Frontend Architecture Changes

Add normalized entity handling while preserving current frame playback.

State changes:

- Add canonical entity types: `VideoAnalysisStudentEntity`, `StudentEntityStore`, `OverlayEventIdentity`.
- Normalize incoming `overlay.frame.entities[]` by `entity_id` or `identity_scope + tracking_id`.
- Replace entity state immutably per event.
- Maintain frame history separately from current entity map.
- Do not render student cards from raw `bounding_boxes`; render cards from canonical entities.
- Keep raw `bounding_boxes` only for model debug overlays and visibility toggles.

Idempotency:

- `useVideoAnalysisLive` ignores stale lower `frame_index` and duplicate `event_id`.
- `useLiveSessionOverlay` rejects `<= lastAppliedFrame` unless `event_id` is new and explicitly marked as a correction.
- `detectionStore` gets an upsert path keyed by `camera_id + frame_number` or `event_id`.
- `useWebSocket` cleanup gets generation tokens so old sockets cannot reconnect after route/session changes.
- Reset job/session state on `jobId`, `sessionId`, and camera set changes.

Rendering fixes:

- `VideoPlayer` renders one canonical card per student entity.
- `OverlayCanvas` draws one primary bbox per canonical entity, with optional per-model evidence controlled by visibility.
- React keys use `entity_id`; fallback is `identity_scope:tracking_id`.
- Add development-only duplicate warnings when a frame contains duplicate canonical keys.

## Debugging And Runtime Validation

Backend instrumentation:

- Log canonical entity creation/update/end events.
- Log duplicate model evidence merges by `tracking_id`.
- Log duplicate `event_id` emission attempts.
- Add runtime assertions in debug mode:
  - `len(entities) == len(unique(entity_id))`
  - no duplicate `(identity_scope, tracking_id)` in one frame
  - no duplicate persisted `StudentTrack(job, tracking_id)`

Frontend instrumentation:

- Warn when duplicate `entity_id` appears in one payload.
- Warn when stale or duplicate `event_id` is discarded.
- Warn when raw boxes imply more student cards than canonical entities.

Artifacts:

- `identity_lifecycle.jsonl` per offline/live job.
- Optional `canonical_entities.json` per job/session for replay debugging.

## Test Plan

Backend unit tests:

- Tracker ID propagation creates one canonical entity with multiple model evidence records.
- Secondary model boxes do not become separate student entities.
- `tracking_id=0` does not collapse multiple real people into one canonical student.
- Offline frame persistence is idempotent across reruns/retries.
- `StudentTrack` resolver is safe under duplicate creation attempts.
- ReID self-comparison is excluded.
- Stale/lost lifecycle updates are stored in `track_metadata`.

Backend integration tests:

- Detection -> tracking -> canonical aggregation -> websocket emits one entity per `tracking_id`.
- Multi-model outputs for one student produce one canonical entity plus multiple evidence items.
- Repeated offline processing does not duplicate `Detection`/`BoundingBox`.
- Live repeated frame updates replace/upsert, not append duplicate logical entities.
- Live dual session/camera delivery carries the same `event_id`.

Backend stress/regression tests:

- 50 students across high FPS frames.
- Rapid movement and crossing students.
- Occlusion/lost/reappeared tracks.
- Reconnect/replayed websocket event.
- Explicit regression for duplicate `tracking-id:1_student`.

Frontend unit tests:

- Duplicate websocket packets do not duplicate entities.
- Same tracking ID across multiple model boxes renders one student card.
- Removed cameras clear stale buffers.
- Websocket generation token prevents stale reconnect streams.
- Simulate live websocket updates with repeated packets.
- Simulate rapid frames, reconnect, and replay.
- Assert `ONE TRACK ID == ONE UI ENTITY` in rendered DOM.
- Verify raw model overlays can still be toggled without duplicating student cards.

## Execution Order

1. Write root cause report and diagrams from current findings.
2. Add backend canonical identity registry and entity payload builder.
3. Make offline and live persistence idempotent and canonical-track safe.
4. Add `entities[]`, `event_id`, and event metadata to websocket/API payloads.
5. Update frontend types, hooks, stores, and rendering to use canonical entities.
6. Add debug instrumentation and runtime invariant checks.
7. Add backend unit/integration/regression/stress tests.
8. Add frontend unit and E2E tests.
9. Run targeted backend and frontend suites, then run the real upload/live reproduction scenario.

## Assumptions

- The invariant is scoped per offline job or live camera stream, not global across unrelated videos.
- Existing `boxes[]` remains backward compatible during rollout.
- `entities[]` becomes authoritative for student UI cards and canonical overlays.
- Raw model-level boxes remain available for debugging and model visibility.
- No database migration is required for the first fix because `StudentTrack.track_metadata` can hold lifecycle state.
- A later global identity feature would require a separate cross-session identity model.
