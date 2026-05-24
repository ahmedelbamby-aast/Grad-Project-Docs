# Inference Bottlenecks and Solution Decision Matrix

## Table 1: Current Issues, Bottlenecks, and Direct Fixes

| Area | Issue / Bottleneck | Evidence | Impact | Solution |
|---|---|---|---|---|
| End-to-end runtime | High total latency for short video, improved in latest full-frame validation | Baseline job `e2e83ad0-7e15-4821-8a91-5f8d18611ade` took `835,716.724 ms` (~13.93 min); latest full-frame job `3a5989c0-f8fc-4276-880a-050ca6d3fbc2` took `637,069.179 ms` (~10.62 min), ~23.8% faster | Still significant runtime, but trend is improved | Continue reducing per-frame inference load, tune Triton concurrency/queueing, overlap stages safely |
| Inference stage share | Inference dominates runtime | Stage totals show inference time much larger than decode/preprocess/postprocess | Main bottleneck | Prioritize model-level and scheduler-level optimization before other work |
| Triton timeouts | Repeated timeout responses under load | Multiple `TRITON_TIMEOUT` warnings in benchmark logs | Adds latency and drops frame/model results | Increase timeout budget appropriately, reduce burst concurrency, tune `instance_group` and batch scheduler |
| Pose reliability | Pose records can drop silently | Runtime bug around ambiguous numpy truth evaluation caused result drops | Missing skeleton artifacts | Fixed parsing guard in `pose_runtime` and added regression test |
| Pipeline sequencing | Pose runs after detection stage (serial) | Current offline flow executes pose after frame detections are built | Adds tail latency | Consider frame-level parallelization strategy with bounded queues |
| Worker topology | Duplicate workers race risk | Prior duplicate node observations | Duplicate consumption / unstable execution | Enforce unique worker names and one worker per queue role |
| Worker pool model | `solo` pool limits concurrency for heavy jobs | Current process model for heavy pipeline | Lower parallelism potential | Move heavy offline queue to prefork if stable on host |
| Frame volume | Processing too many frames | 9s input still drives large model-call volume | Unnecessary compute | Keep `TRITON_OFFLINE_FRAME_STRIDE=2`; test `3` with quality gate |
| Observability | Stage attribution historically weak | Root-cause needed deep log inspection | Slow troubleshooting | Keep explicit stage telemetry and benchmark command as standard gate |
| Event delivery robustness | Redis channel broadcast can fail near completion | Observed `WinError 10054` in Channels broadcast | UI event reliability issue | Add retry/guarded non-fatal broadcast path; decouple completion from WS publish |
| Validation quality gate | Need explicit artifact/pose completeness in benchmark verdict | Latest full-frame benchmark (`3a5989c0...`) produced non-empty pose records (`pose_record_count=125`), rendered `pose_output_video.mp4`, rendered `final_output_video.mp4`, and exported parseable JSON artifacts (`inference_audit.json`, `pose_results.json`, `pose_per_student.json`, `pose_quality_summary.json`) | Prevents false pass from timing-only checks | Keep artifact-completeness gate mandatory for benchmark pass/fail |

## Table 2: Deep Solution Matrix (Docs-backed) and Recommendation

| Strategy | What It Means | Expected Gain | Risks / Tradeoffs | Prerequisites | Fit for Your System | Recommendation |
|---|---|---|---|---|---|---|
| Triton per-model tuning (`instance_group`, dynamic batching, queue delay) | Tune each model config using measured sweeps instead of fixed guesses | High for throughput stability; medium-high for latency when tuned correctly | Overtuning can increase queue wait or memory pressure | Model supports batching (`max_batch_size > 1` for real gains), benchmark tooling | Strong fit; your bottleneck is inference-heavy | **Top priority** |
| Model Analyzer + Perf Analyzer loop | Use automated search to find best config frontier (latency vs throughput) | High confidence optimal configs | Time to run experiments; requires stable test harness | Triton SDK tools available, reproducible sample workload | Strong fit for your 9s workload benchmarking | **Top priority** |
| Parallel pose with object detection (pipeline parallelism) | Run pose worker concurrently against detection outputs (streaming/queue) | Medium to high wall-time reduction when overlap is effective | Resource contention may increase Triton timeouts without queue controls | Clear frame contract, bounded queues, backpressure | Good fit after timeout stabilization | **Second phase after Triton tuning** |
| Multithreading in app layer (per-frame/per-model fan-out) | Increase parallel dispatch from backend workers | Medium in best case; can be negative if Triton saturates | Thread oversubscription, timeout amplification | Strict concurrency limits and pooling | Moderate fit; risky before scheduler tuning | Use **bounded** threads only |
| Move from `solo` to `prefork` for heavy offline workers | Multiple processes execute tasks in parallel | Medium throughput gain at queue level | Windows/runtime stability considerations | Stable Celery + broker behavior under prefork | Good fit if validated on your host | Do with controlled canary |
| Offline stride increase (2 -> 3) | Skip more frames to cut calls | High speed gain, direct cost reduction | Possible quality loss (missed short events/poses) | Acceptance criteria for detection/pose quality | Good fit as optional profile | Apply only with quality gate |
| Pose model architecture shift (top-down RTMPose -> one-stage RTMO or lighter RTMPose) | Reduce per-person crop-infer overhead in crowded scenes | Potentially high in crowd scenes | Migration complexity, retraining/validation | Model availability and integration work | Medium fit; strategic, not immediate | Evaluate after infra tuning |
| Tracker strategy tuning (BoT-SORT vs ByteTrack, ReID usage) | Select tracker complexity based on need | Medium latency savings if ReID-heavy path reduced | ID consistency tradeoff in difficult scenes | Scenario-based tracker policy | Good fit with classroom patterns | Use profile-based tracker policy |
| Broadcast resilience hardening | Retries/circuit breaker for Redis Channels publishes | Stability gain, not speed | Slight complexity | Centralized event publish utility | Strong fit due to observed errors | Implement now |
| Full frame-stage SLO enforcement | Enforce budgets per stage, alert on regressions | Prevents future 35min-style regressions | Requires telemetry discipline | Persistent metrics + CI/perf checks | Strong fit | Implement as policy |

## Best-Fit Execution Order for Your Case

| Priority | Action | Why This Order |
|---|---|---|
| 1 | Stabilize Triton request success rate and tune per-model scheduler params with benchmark sweeps | Current dominant loss is inference timeout/latency |
| 2 | Keep one worker per queue role + unique nodenames + validate prefork for offline queue | Prevents queue-level instability and unlocks safe concurrency |
| 3 | Introduce bounded pipeline parallelism (pose alongside detection) | Gains wall-time only when base inference path is stable |
| 4 | Evaluate stride=3 and/or lighter pose variant with quality gates | Largest extra speed wins, but quality-sensitive |
| 5 | Enforce SLOs from telemetry in CI/perf runs | Keeps optimizations from regressing |

## Sources

- Triton Model Configuration (dynamic batching, preferred sizes, queue delay): https://docs.nvidia.com/deeplearning/triton-inference-server/archives/triton_inference_server_230/user-guide/docs/model_configuration.html
- Triton concurrent model execution (`instance_group`): https://docs.nvidia.com/deeplearning/triton-inference-server/archives/triton-inference-server-2630/user-guide/docs/user_guide/model_execution.html
- Triton conceptual guide (dynamic batching + concurrent execution): https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/tutorials/Conceptual_Guide/Part_2-improving_resource_utilization/README.html
- Triton Model Analyzer overview: https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/model_analyzer/README.html
- RTMPose paper (speed/accuracy characteristics): https://arxiv.org/abs/2303.07399
- MMPose inference speed summary: https://mmpose.readthedocs.io/en/0.x/inference_speed_summary.html
- MMPose deployment guidance: https://mmpose.readthedocs.io/en/latest/user_guides/how_to_deploy.html
- Ultralytics tracking mode (BoT-SORT/ByteTrack): https://docs.ultralytics.com/modes/track/
- Ultralytics tracker references (BoT-SORT, ByteTrack internals): https://docs.ultralytics.com/reference/trackers/bot_sort/ and https://docs.ultralytics.com/reference/trackers/byte_tracker/
- Ultralytics configuration options (e.g., `half`, `imgsz`, batch-related settings): https://docs.ultralytics.com/usage/cfg/
