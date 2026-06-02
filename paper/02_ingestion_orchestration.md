# 1. Executive Summary

**Last updated:** 2026-06-02

- `IMPLEMENTED`: Upload and live ingestion entrypoints enqueue Celery tasks with explicit queue selection and queue timestamp metadata (`queued_at_ms`). Evidence: `E:/grad_project/backend/apps/video_analysis/views.py:71-109`, `E:/grad_project/backend/apps/cameras/views.py:103`, `E:/grad_project/backend/apps/sessions/views.py:90-94`.
- `IMPLEMENTED`: Celery queue topology, route isolation for control/live-worker queues, and retry/ack resilience flags are configured. Evidence: `E:/grad_project/backend/config/celery.py:57-80`.
- `PARTIAL`: Offline per-model queues are declared but not mapped in `task_routes` for upload fanout. Evidence: queues declared at `E:/grad_project/backend/config/celery.py:44-46,64-66`; routes listed at `E:/grad_project/backend/config/celery.py:68-74`.
- `INCONSISTENT`: Live task records reconnect policy examples but no reconnect loop/counter mutation is evidenced in the live task body. Evidence: `E:/grad_project/backend/apps/video_analysis/tasks.py:3612-3617`; reconnect execution exists in camera health task instead (`E:/grad_project/backend/apps/cameras/tasks.py:100-146`).

# 2. Audit Scope
- Scope included only: ingestion paths, RTSP/upload buffering lifecycle, queue routing/retries/backpressure, failure handling.
- Primary artifacts audited:
  - `E:/grad_project/backend/apps/video_analysis/views.py`
  - `E:/grad_project/backend/apps/cameras/views.py`
  - `E:/grad_project/backend/apps/sessions/views.py`
  - `E:/grad_project/backend/config/celery.py`
  - `E:/grad_project/backend/apps/video_analysis/tasks.py`
  - `E:/grad_project/backend/apps/cameras/tasks.py`
  - `E:/grad_project/backend/apps/tracking/rtsp_worker.py`
  - `E:/grad_project/backend/apps/pipeline/buffering.py`
  - `E:/grad_project/backend/apps/pipeline/live_buffer.py`
  - `E:/grad_project/backend/apps/pipeline/runtime_ingestion.py`
  - Supporting tests under `E:/grad_project/backend/tests/**`.

# 3. Implemented (Evidence-Backed)
- `IMPLEMENTED`: Upload enqueue path with worker-availability fallback and queue timestamp injection.
  - Evidence: `E:/grad_project/backend/apps/video_analysis/views.py:57-67`, `E:/grad_project/backend/apps/video_analysis/views.py:71-109`.
- `IMPLEMENTED`: Upload validation + file persistence path.
  - Evidence: `E:/grad_project/backend/apps/video_analysis/views.py:348`, `E:/grad_project/backend/apps/video_analysis/views.py:382`.
- `IMPLEMENTED`: Live enqueue from camera API.
  - Evidence: `E:/grad_project/backend/apps/cameras/views.py:103`.
- `IMPLEMENTED`: Celery queue definitions and control/live route mapping.
  - Evidence: `E:/grad_project/backend/config/celery.py:57-74`.
- `IMPLEMENTED`: Upload task timeout/retry envelope (`soft_time_limit`, `time_limit`, `max_retries`) and retry countdowns.
  - Evidence: `E:/grad_project/backend/apps/video_analysis/tasks.py:1936-1941`, `E:/grad_project/backend/apps/video_analysis/tasks.py:1975-1979`.
- `IMPLEMENTED`: Upload queue wait/status telemetry hooks.
  - Evidence: `E:/grad_project/backend/apps/video_analysis/tasks.py:245-259`, `E:/grad_project/backend/apps/video_analysis/tasks.py:1961-1964`, `E:/grad_project/backend/apps/video_analysis/tasks.py:3330`.
- `IMPLEMENTED`: RTSP reconnect primitives (`compute_backoff_seconds`, `should_reconnect`).
  - Evidence: `E:/grad_project/backend/apps/tracking/rtsp_worker.py:6`, `E:/grad_project/backend/apps/tracking/rtsp_worker.py:13`.
- `IMPLEMENTED`: Camera health reconnect workflow with exponential delay and failure state transition.
  - Evidence: `E:/grad_project/backend/apps/cameras/tasks.py:25-27`, `E:/grad_project/backend/apps/cameras/tasks.py:100-146`, `E:/grad_project/backend/apps/cameras/tasks.py:150-153`.
- `IMPLEMENTED`: Buffering primitives (RAM/spill/drain/or overflow policy).
  - Evidence: `E:/grad_project/backend/apps/pipeline/buffering.py:45`, `E:/grad_project/backend/apps/pipeline/buffering.py:91`, `E:/grad_project/backend/apps/pipeline/buffering.py:219`, `E:/grad_project/backend/apps/pipeline/buffering.py:265`.
- `IMPLEMENTED`: Offline backpressure controls for Triton frame queue and forced flush on `max_pending` overflow.
  - Evidence: `E:/grad_project/backend/apps/video_analysis/tasks.py:1517-1525`, `E:/grad_project/backend/apps/video_analysis/tasks.py:1752-1759`, `E:/grad_project/backend/apps/video_analysis/tasks.py:1821-1826`.
- `IMPLEMENTED`: Runtime ingestion persistence for buffer pressure and worker rollups.
  - Evidence: `E:/grad_project/backend/apps/pipeline/runtime_ingestion.py:224`, `E:/grad_project/backend/apps/pipeline/runtime_ingestion.py:415`, `E:/grad_project/backend/apps/pipeline/runtime_ingestion.py:776`.

# 4. Partially Implemented (Evidence-Backed)
- `PARTIAL`: Offline model-specific queues exist but are not explicitly routed in `task_routes` for upload fanout.
  - Evidence: queues defined at `E:/grad_project/backend/config/celery.py:44-46,64-66`; route entries only at `E:/grad_project/backend/config/celery.py:68-74`.
- `PARTIAL`: Upload enqueue helper explicitly selects `CELERY_VIDEO_INFERENCE_QUEUE`, while central route maps `process_video_upload` to `CELERY_OFFLINE_CONTROL_QUEUE` alias path.
  - Evidence: `E:/grad_project/backend/apps/video_analysis/views.py:74-79`, `E:/grad_project/backend/config/celery.py:37-40`, `E:/grad_project/backend/config/celery.py:70`.
- `PARTIAL`: Live ingestion task accepts `queued_at_ms` via call sites, but live task does not record queue wait telemetry using `_record_task_queue_start`.
  - Evidence enqueue call sites: `E:/grad_project/backend/apps/cameras/views.py:103`, `E:/grad_project/backend/apps/sessions/views.py:90-94`; live task signature/body: `E:/grad_project/backend/apps/video_analysis/tasks.py:3334-3343`.

# 5. Missing (Evidence-Backed)
- `MISSING`: Direct integration of `DelayedLiveBuffer` into `run_live_stream_inference` is not evidenced.
  - Evidence: `DelayedLiveBuffer` exists in `E:/grad_project/backend/apps/pipeline/live_buffer.py:38`; no references from live task file search (`E:/grad_project/backend/apps/video_analysis/tasks.py`, search terms `DelayedLiveBuffer|live_buffer` returned no matches).
- `MISSING`: Explicit starvation-prevention scheduler (for example weighted-fair queue scheduler) is not evidenced in ingestion execution paths.
  - Evidence: Celery fairness knobs exist (`E:/grad_project/backend/config/celery.py:77-80`), but no starvation scheduler implementation located in audited ingestion modules listed in Section 2.

# 6. Architecturally Incorrect / Inconsistent
- `INCONSISTENT`: Live reconnect policy is audited as metadata/examples in `run_live_stream_inference`, while actual reconnect loop exists in a separate periodic health task.
  - Evidence in live task metadata only: `E:/grad_project/backend/apps/video_analysis/tasks.py:3612-3617`.
  - Evidence of operative reconnect loop elsewhere: `E:/grad_project/backend/apps/cameras/tasks.py:100-146`.
- `INCONSISTENT`: Queue naming/routing intent split between per-call explicit queue selection and centralized route mapping may produce operator ambiguity.
  - Evidence: explicit queue at `E:/grad_project/backend/apps/video_analysis/views.py:74-109`; route map at `E:/grad_project/backend/config/celery.py:68-74`.

# 7. Scientifically Weak / Unvalidated
- `PARTIAL`: Ingestion resilience is tested for several queue/buffer behaviors, but no evidence of full empirical validation matrix for RTSP reconnect under production-like fault profiles in this file’s scope.
  - Evidence of existing tests: `E:/grad_project/backend/tests/unit/pipeline/test_live_buffer_coverage.py`, `E:/grad_project/backend/tests/unit/pipeline/test_buffering_recovery_and_injection.py`, `E:/grad_project/backend/tests/unit/video_analysis/test_triton_offline_batch_queue.py`.
- `NOT EVIDENCED`: End-to-end quantified backpressure SLO acceptance criteria (with thresholds bound to ingestion lifecycle states) in code-level gates for this subsystem.
  - Evidence basis: audited ingestion modules in Section 2 expose metrics/events, but no explicit SLO gate artifact was found in those modules.

# 8. Bottlenecks
- `PARTIAL`: Upload queue path includes fallback when no active worker is detected; this mitigates but does not eliminate queue stall risk if both preferred and fallback queues have no active consumer.
  - Evidence: `E:/grad_project/backend/apps/video_analysis/views.py:76-103`.
- `PARTIAL`: Offline pending frame queue can grow to `max_pending` before forced flush; latency can spike under overload.
  - Evidence: `E:/grad_project/backend/apps/video_analysis/tasks.py:1821-1826`.
- `PARTIAL`: Reconnect uses blocking `time.sleep` in camera health task retries.
  - Evidence: `E:/grad_project/backend/apps/cameras/tasks.py:144-145`.

# 9. Production Readiness Assessment
- `IMPLEMENTED`: Core ingestion entrypoints, queue dispatch, retry envelope, and failure persistence exist.
  - Evidence: `E:/grad_project/backend/apps/video_analysis/views.py:71-109`, `E:/grad_project/backend/apps/video_analysis/tasks.py:1936-1981`, `E:/grad_project/backend/apps/video_analysis/tasks.py:1860`, `E:/grad_project/backend/apps/cameras/tasks.py:150-157`.
- `PARTIAL`: Backpressure controls are stronger for offline frame-batch execution than for live stream ingestion.
  - Evidence: offline queue controls in `E:/grad_project/backend/apps/video_analysis/tasks.py:1517-1525,1752-1759,1821-1826`; no direct live-buffer hookup evidence (Section 5).
- `PARTIAL`: Queue routing semantics are operational but not fully normalized (Section 6).

# 10. Equations Actually Used
- `IMPLEMENTED`: Reconnect backoff equation `delay = base * 2^(attempt-1)` (bounded in helper).
  - Evidence: helper `E:/grad_project/backend/apps/tracking/rtsp_worker.py:6`; retry delay usage `E:/grad_project/backend/apps/cameras/tasks.py:144`.
- `IMPLEMENTED`: Effective FPS equation `effective_fps = 1000 / frame_delta_ms` when delta is positive.
  - Evidence: `E:/grad_project/backend/apps/pipeline/runtime_ingestion.py:165`.
- `IMPLEMENTED`: Worker error/timeout rates computed as count/total.
  - Evidence: `E:/grad_project/backend/apps/pipeline/runtime_ingestion.py:466-467`, `E:/grad_project/backend/apps/pipeline/runtime_ingestion.py:758-759`.

# 11. Figures/Images Candidates from Repo
- `IMPLEMENTED`: Candidate figure asset exists.
  - `E:/grad_project/Images_Docs/docs/backend/architecture/docs-backend-architecture-data-flow__2-offline-upload-inference-pipeline__sequencediagram__line-33__b002.png`
- `IMPLEMENTED`: Candidate figure asset exists.
  - `E:/grad_project/Images_Docs/docs/backend/architecture/docs-backend-architecture-data-flow__3-live-stream-inference-pipeline__flowchart__line-70__b003.png`
- `IMPLEMENTED`: Candidate figure asset exists.
  - `E:/grad_project/Images_Docs/docs/backend/config/docs-backend-config-celery__architecture-diagram__flowchart__line-25__b001.png`
- `IMPLEMENTED`: Candidate figure asset exists.
  - `E:/grad_project/Images_Docs/docs/backend/apps/pipeline/docs-backend-apps-pipeline-runtime-ingestion__data-path__flowchart__line-20__b001.png`

# 12. Contradictions (Code vs Docs vs Tests)
- `INCONSISTENT`: Code has both explicit queue override at enqueue-time and central route declaration for upload task; this can conflict with single-source-of-truth expectations.
  - Evidence code: `E:/grad_project/backend/apps/video_analysis/views.py:74-109`, `E:/grad_project/backend/config/celery.py:68-74`.
  - Evidence tests asserting explicit queue behavior: `E:/grad_project/backend/tests/contract/test_upload.py:79-92`, `E:/grad_project/backend/tests/contract/test_upload.py:95-112`.
- `INCONSISTENT`: Live reconnect behavior appears in two conceptual layers: metadata policy in live task vs operational reconnect in camera health task.
  - Evidence code: `E:/grad_project/backend/apps/video_analysis/tasks.py:3612-3617`, `E:/grad_project/backend/apps/cameras/tasks.py:100-146`.

# 13. Open Questions / Unknowns
- `NOT EVIDENCED`: Whether production deployment wires `run_live_stream_inference` through `DelayedLiveBuffer` in non-repo runtime hooks.
  - Evidence: no direct code reference found from audited live task/module set in Section 2.
- `NOT EVIDENCED`: Whether offline per-model Celery queues are currently consumed by dedicated workers in production process supervisors.
  - Evidence: queue declarations present (`E:/grad_project/backend/config/celery.py:44-46,64-66`), but worker-process binding artifacts were outside this file scope.

# 14. Subsystem Coverage Checklist
- Upload ingestion path (request -> enqueue -> task): `IMPLEMENTED`.
  - Evidence: `E:/grad_project/backend/apps/video_analysis/views.py:71-109`, `E:/grad_project/backend/apps/video_analysis/tasks.py:1943-1983`.
- Live ingestion path (camera/session -> enqueue -> live task): `IMPLEMENTED`.
  - Evidence: `E:/grad_project/backend/apps/cameras/views.py:103`, `E:/grad_project/backend/apps/sessions/views.py:90-94`, `E:/grad_project/backend/apps/video_analysis/tasks.py:3334`.
- RTSP reconnect lifecycle: `PARTIAL`.
  - Evidence: helper + health task implemented (`E:/grad_project/backend/apps/tracking/rtsp_worker.py:6-13`, `E:/grad_project/backend/apps/cameras/tasks.py:100-146`); live-task reconnect loop not evidenced (`E:/grad_project/backend/apps/video_analysis/tasks.py:3612-3617`).
- Buffering lifecycle (RAM/spill/drop): `IMPLEMENTED`.
  - Evidence: `E:/grad_project/backend/apps/pipeline/buffering.py:45,91,219,265`.
- Queue routing/retries/backpressure: `PARTIAL`.
  - Evidence: routing/retry implemented (`E:/grad_project/backend/config/celery.py:68-80`, `E:/grad_project/backend/apps/video_analysis/tasks.py:1936-1981`), offline backpressure implemented (`E:/grad_project/backend/apps/video_analysis/tasks.py:1517-1525,1752-1759,1821-1826`), live buffering integration not evidenced (Section 5).
- Failure handling and persistence: `IMPLEMENTED`.
  - Evidence: `E:/grad_project/backend/apps/video_analysis/tasks.py:1860`, `E:/grad_project/backend/apps/cameras/tasks.py:150-157`, `E:/grad_project/backend/apps/pipeline/runtime_ingestion.py:224`.
