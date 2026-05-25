## Plan: Complete Only the ⚠️ Items in `rtmpose-engineering-validation-report.md` (Spec 007 Pose)

### Summary
Implement only the report’s **partial/indirect (⚠️)** items and leave all **not-implemented (❌)** items untouched.  
Execution focus is to convert each ⚠️ item to **directly implemented + directly test-evidenced** using current runtime architecture (no new major feature families).

### Implementation Changes
1. **Stream runtime hardening (timeout/FPS/timestamp/buffer/overflow/backpressure)**
- Add explicit stream control settings in pipeline config (`*_TIMEOUT_MS`, `*_BUFFER_MAX_FRAMES`, `*_QUEUE_MAX_EVENTS`, `*_MAX_EMPTY_READS`), with safe defaults.
- In stream loop, emit normalized per-frame timing fields (`source_timestamp_ms`, `ingest_timestamp_ms`, `frame_delta_ms`, `effective_fps`) and enforce monotonic timestamp sync.
- Add bounded in-memory frame/event buffers (`deque(maxlen=...)`) with overflow counters; drop policy = `drop_oldest_keep_latest`.
- Emit overflow/backpressure counters into runtime lifecycle payloads and runtime summary events.

2. **Detection-layer partials completion**
- Add deterministic post-detection suppression pass:
  - intra-model duplicate suppression (IoU threshold),
  - cross-model duplicate guard for person boxes.
- Add explicit occlusion-aware box carry-forward policy (short hold with expiration) for continuity.
- Add detector benchmark hooks and metrics capture (`fps`, `latency_ms`) as first-class runtime payload fields.
- Preserve existing detection behavior; this pass is additive and configurable.

3. **Tracking continuity and lifecycle completion**
- Add track lifecycle state machine in track metadata/event payload (`active`, `occluded`, `reidentified`, `lost`, `ended`) without introducing new DB tables.
- Make lost-track timeout fully configurable via config (frame-based + ms-based options).
- Add continuity scoring per track and emit in runtime events and student timeline payload.
- Add tracker desync recovery flow (state reset/reassociate trigger) and explicit recovery event logging.

4. **RTMPose runtime partials completion**
- Replace mock-only inference path with provider-backed execution path (ONNXRuntime primary, existing backend policy fallback), while keeping current ROI crop/map contract.
- Enforce RTMPose model identity validation at startup (manifest path/version check) and emit `model_identity` event.
- Emit runtime device/provider fields to prove GPU execution status when active.
- Add pose-FPS and end-to-end latency instrumentation at per-frame and per-session summary levels.
- Keep existing fallback route and make fallback reason codes explicit (`provider_error`, `timeout`, `model_unavailable`, etc.).

5. **Temporal buffer/timestamp alignment/real-time update completion**
- Implement per-track rolling temporal buffer (windowed by seconds + max frames) keyed by tracking ID.
- Store aligned timestamps with each buffered point and enforce continuity checks (no backward time, bounded gaps).
- Expose buffer health and update freshness in runtime timeline APIs for direct verification.

6. **Behavior-feature and temporal-embedding partial storage completion**
- Persist currently available behavior signals (head-orientation proxy, posture deviation proxy, continuity/jitter summaries) into structured metadata blocks.
- Persist lightweight temporal embeddings from buffered keypoint deltas/windows (no new deep model training path).
- Optimize storage format for these blocks (compact numeric encoding + capped history) to satisfy efficient memory storage partial.
- Keep this scoped to storage/readiness only; do not add unimplemented temporal anomaly modeling.

7. **Runtime observability + temporal replay visualization completion**
- Extend runtime API payloads with GPU and memory runtime factors (`gpu_util_percent`, `gpu_memory_percent`, `gpu_power_watts`, process/rss memory, queue depth, throughput_fps`).
- Add temporal replay data surface to runtime timeline/student timeline responses (ordered points with lifecycle state and timestamps).
- Update runtime frontend types/page to render replay controls and GPU/queue/backpressure metrics directly from API.

8. **Scientific/readiness partial validation completion**
- Add deterministic validation scripts/tests for:
  - temporal stability score from trajectories,
  - noise-reduction score,
  - temporal continuity score,
  - multi-person/long-session continuity robustness,
  - contrastive-learning-ready tensor export sanity.
- These are evidence-and-quality validations for existing partial capabilities, not new ❌ feature implementations.

### Public Interface / Contract Changes
- Extend runtime event payload schema with:
  - stream timing sync fields,
  - buffer/overflow/backpressure counters,
  - track lifecycle and continuity fields,
  - pose provider/device/fallback reason fields,
  - GPU/memory throughput metrics.
- Extend runtime API response shapes (`runtime/timeline`, `runtime/student-timeline`, `runtime/summary`) to expose the above fields.
- Extend frontend runtime types accordingly (`frontend/src/types/runtime.ts`).

### Test Plan (must pass)
1. **Unit**
- Stream timeout and buffer overflow policy behavior.
- Duplicate suppression and occlusion hold/release logic.
- Track lifecycle transitions and timeout configurability.
- Timestamp alignment and continuity scoring.
- Pose provider fallback reason mapping.
2. **Integration**
- Live stream event payload completeness (new fields present and typed).
- Runtime API includes replay and GPU/queue metrics under filters.
- Metadata persistence for behavior features and temporal embeddings.
3. **System/Benchmark**
- Multi-person classroom continuity scenario.
- Long-running session stability scenario.
- Detector FPS/latency benchmark output.
- Pose FPS/e2e latency benchmark output.
- Throughput/backpressure benchmark output.
4. **Evidence update**
- Update `specs/007-pose-behavior-pipeline/evidence/final/rtmpose-engineering-validation-report.md` by flipping only addressed ⚠️ items to ✅.
- Keep all ❌ items unchanged and explicitly marked out-of-scope.

### Assumptions and Defaults
- Scope includes **all and only** ⚠️ items from the referenced report.
- ❌ items are intentionally deferred; no implementation for them in this plan.
- No new core DB tables; use existing JSON fields and runtime event payload extensions unless a blocker is discovered.
- Existing release-gate flow remains unchanged; new checks are added as tests/evidence within current CI/testing structure.
