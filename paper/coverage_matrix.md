# Coverage Matrix

**Last updated:** 2026-06-02

| subsystem/module | covered_by_report | status | evidence_refs |
|---|---|---|---|
| Architecture boundaries and module dependency guardrails | 01_architecture_deployment.md | covered | E:/grad_project/paper/01_architecture_deployment.md |
| Deployment/runtime profile policy and Triton endpoint switching | 01_architecture_deployment.md | covered | E:/grad_project/paper/01_architecture_deployment.md |
| Production env and infra script consistency | 01_architecture_deployment.md | partial | E:/grad_project/paper/01_architecture_deployment.md |
| Upload ingestion lifecycle and task enqueue | 02_ingestion_orchestration.md | covered | E:/grad_project/paper/02_ingestion_orchestration.md |
| RTSP/live ingestion lifecycle and buffering | 02_ingestion_orchestration.md | covered | E:/grad_project/paper/02_ingestion_orchestration.md |
| Queue routing/retry/backpressure | 02_ingestion_orchestration.md | partial | E:/grad_project/paper/02_ingestion_orchestration.md |
| Detection cadence/reuse/interpolation controls | 03_detection_tracking.md | covered | E:/grad_project/paper/03_detection_tracking.md |
| Tracking lifecycle and identity persistence/re-entry | 03_detection_tracking.md | partial | E:/grad_project/paper/03_detection_tracking.md |
| Pose ROI mapping + Triton inference path | 04_pose_temporal_artifacts.md | covered | E:/grad_project/paper/04_pose_temporal_artifacts.md |
| Pose temporal smoothing/jitter controls | 04_pose_temporal_artifacts.md | partial | E:/grad_project/paper/04_pose_temporal_artifacts.md |
| Data model + artifacts + REST/WebSocket contracts | 05_data_api_frontend.md | covered | E:/grad_project/paper/05_data_api_frontend.md |
| Frontend overlay/debug/data-readiness integration | 05_data_api_frontend.md | partial | E:/grad_project/paper/05_data_api_frontend.md |
| Observability and runtime telemetry integrity | 06_observability_testing_science.md | covered | E:/grad_project/paper/06_observability_testing_science.md |
| Benchmark validity and scientific test readiness | 06_observability_testing_science.md | partial | E:/grad_project/paper/06_observability_testing_science.md |

## Coverage Status

- Implemented-system scope coverage across currently implemented modules: `100% covered by at least one report`.
- Rows marked `partial` indicate implementation maturity gaps, not audit coverage gaps.
