# Exam Monitoring Dashboard

**Last updated:** 2026-06-02

Temporal behavioral intelligence platform with a Django backend, React
frontend, WebSocket live updates, offline video analysis, camera streaming
through go2rtc, and Triton-authoritative production inference.

**Maintainer:** Eng.Ahmed ElBamby

---

## Documentation Reading Order (read these in this order to understand the complete project)

This is the canonical reading path through every project-owned narrative
document. The dates in the right column are each file's
`**Last updated:**` header (auto-populated from each file's last git
commit date if it had no header before). If a date and the reading order
disagree, the **reading order is authoritative** — dates only reflect
when a doc was last touched, not where it belongs in the narrative.

The order below is grouped into 9 reading phases. Each phase has a
purpose; you can stop after any phase if you only need that level of
context.

### Phase 0 — Orientation (start here, in this exact order)

| # | File | Last updated | Why read this here |
|---|---|---|---|
| 1 | [`README.md`](README.md) | 2026-06-02 | You are here. The orientation map and quick-start environment knobs. |
| 2 | [`quickstart.md`](quickstart.md) | 2026-05-30 | Step-by-step local + Linux production setup. Run the commands; don't just read. |
| 3 | [`AGENTS.md`](AGENTS.md) | 2026-06-02 | Constitutional discipline + production server topology + the full log of every accepted / not-accepted optimization cycle. Memorize the "no acceptance without prod evidence" rule + the DSP non-negotiables. |
| 3a | [`docs/documentation_systematization_plan.md`](docs/documentation_systematization_plan.md) | 2026-06-02 | Documentation Systematization Program (DSP) master plan. Read before touching any markdown file. Constitution Section 19 backstops it. |
| 3b | [`docs/per_entity_doc_template.md`](docs/per_entity_doc_template.md) | 2026-06-02 | The template every entity doc MUST follow. Required by constitution § 19.2. |
| 3c | [`docs/mermaid_theme_contract.md`](docs/mermaid_theme_contract.md) | 2026-06-02 | The Mermaid theme + palette + label-fitting contract. Required by constitution § 19.3, § 19.4. |
| 3d | [`.specify/memory/constitution.md`](.specify/memory/constitution.md) | 2026-06-02 (v2.5.0) | Full constitutional text. Section 19 is the DSP backstop. Footer carries `**Version** \| **Ratified** \| **Last Amended**`. |

### Phase 1 — System architecture & module boundaries

| # | File | Last updated | Why read this here |
|---|---|---|---|
| 4 | [`docs/INDEX.md`](docs/INDEX.md) | 2026-06-02 | Documentation index across the whole `docs/` tree (auto-generated mirrors + narrative docs). |
| 5 | [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | 2026-05-25 | High-level system architecture: Django + Celery + Triton + React shape. |
| 6 | [`docs/architecture/modular-system-overview.md`](docs/architecture/modular-system-overview.md) | 2026-05-09 | The module-decomposition principle the codebase follows. |
| 7 | [`docs/architecture/module-boundary-map.md`](docs/architecture/module-boundary-map.md) | 2026-05-09 | Per-module ownership and allowed cross-imports. |
| 8 | [`docs/architecture/runtime-scenario-matrix.md`](docs/architecture/runtime-scenario-matrix.md) | 2026-05-25 | Live vs offline vs hybrid runtime profiles and which models load where. |
| 9 | [`docs/architecture/compatibility-contracts.md`](docs/architecture/compatibility-contracts.md) | 2026-05-09 | Public contracts that can't break between modules. |
| 10 | [`docs/architecture/coupling-risk-register.md`](docs/architecture/coupling-risk-register.md) | 2026-05-09 | Catalogued coupling hotspots. |
| 11 | [`docs/architecture/documentation-diagram-coverage.md`](docs/architecture/documentation-diagram-coverage.md) | 2026-05-09 | How the mermaid coverage gate validates that every module is diagrammed. |

### Phase 2 — Inference pipeline plan & SLA (the active optimization work)

| # | File | Last updated | Why read this here |
|---|---|---|---|
| 12 | [`docs/runtime_sla_video_plus_5min.md`](docs/runtime_sla_video_plus_5min.md) | 2026-06-02 | **The SLA contract**: `total_wall ≤ duration(video) + 5 min`. For `combined.mp4` (4 541 frames, 2 m 31 s) that means total ≤ 7 m 31 s = **≥ 10.07 FPS overall**. |
| 13 | [`docs/triton_models_and_tensor_anatomy.md`](docs/triton_models_and_tensor_anatomy.md) | 2026-06-01 | What every Triton-loaded model does + tensor shapes + byte math for the dense-output inefficiency the cycles attack. |
| 14 | [`docs/inference_parallelization_plan.md`](docs/inference_parallelization_plan.md) | 2026-06-02 | The original parallelization plan (binary tensors, async fan-out, dynamic batching). |
| 15 | [`docs/inference_bottlenecks_and_solution_matrix.md`](docs/inference_bottlenecks_and_solution_matrix.md) | 2026-05-22 | Catalog of bottlenecks ranked by leverage. |
| 16 | [`docs/production_inference_benchmark.md`](docs/production_inference_benchmark.md) | 2026-06-02 | **The per-cycle benchmark table** — current-state and historical metrics for every prod run, with FPS column. |
| 17 | [`docs/cycle_9_and_10_improvements_todo.md`](docs/cycle_9_and_10_improvements_todo.md) | 2026-06-02 | **The single TODO entry point.** § Z inside has the map of every cycle (accepted, not accepted, staged, planned, deferred). Start every new session here. |
| 18 | [`docs/cycles_9_to_12_implementation_playbook.md`](docs/cycles_9_to_12_implementation_playbook.md) | 2026-06-02 | The full Cycles 9–12 implementation roadmap and projections. |
| 19 | [`docs/next_agent_starter_prompt.md`](docs/next_agent_starter_prompt.md) | 2026-06-02 | The self-contained briefing for the next agent picking this work up. |

### Phase 3 — Per-cycle investigation + results (read each pair in order; investigation → results)

| # | File | Last updated | Why read this here |
|---|---|---|---|
| 20 | [`docs/crop_frame_optimization_audit.md`](docs/crop_frame_optimization_audit.md) | 2026-06-01 | Original crop-frame audit that kicked off the optimization plan. |
| 21 | [`docs/crop_frame_optimization_execution.md`](docs/crop_frame_optimization_execution.md) | 2026-06-02 | Per-cycle execution log of the crop-frame pipeline changes. |
| 22 | [`docs/crop_frame_rtx5090_bottleneck_investigation.md`](docs/crop_frame_rtx5090_bottleneck_investigation.md) | 2026-06-01 | RTX 5090 GPU-side bottleneck breakdown. |
| 23 | [`docs/crop_gpu_vs_cpu_comparison.md`](docs/crop_gpu_vs_cpu_comparison.md) | 2026-05-31 | GPU vs CPU crop-preprocessing comparison. |
| 24 | [`docs/rtt_root_cause_investigation_77650001.md`](docs/rtt_root_cause_investigation_77650001.md) | 2026-06-01 | The initial RTT decomposition for job `77650001` (192-212 ms baseline). |
| 25 | [`docs/cycle_9_investigation.md`](docs/cycle_9_investigation.md) | 2026-06-01 | Cycle 9 investigation (Triton behavior ensemble). |
| 26 | [`docs/cycle_9_results.md`](docs/cycle_9_results.md) | 2026-06-01 | Cycle 9 production outcome + post-mortem (NOT ACCEPTED — the canonical "name the lever" precedent). |
| 27 | [`docs/cycle_9b_output_fusion_investigation.md`](docs/cycle_9b_output_fusion_investigation.md) | 2026-06-02 | Cycle 9b output-fusion design space (B.2.a/b/c). |
| 28 | [`docs/cycle_9b_output_fusion_results.md`](docs/cycle_9b_output_fusion_results.md) | 2026-06-02 | Cycle 9b output-fusion comparison matrix (B.2.b rejected, slice + Top-K accepted). |
| 29 | [`docs/cycle_9b_exact_slice_investigation.md`](docs/cycle_9b_exact_slice_investigation.md) | 2026-06-02 | Exact server-side slicing investigation (B.2.b ACCEPTED path). |
| 30 | [`docs/cycle_9b_topk_anchor_packing_investigation.md`](docs/cycle_9b_topk_anchor_packing_investigation.md) | 2026-06-02 | Top-K anchor packing investigation (B.2.c). |
| 31 | [`docs/cycle_9b_topk_anchor_packing_results.md`](docs/cycle_9b_topk_anchor_packing_results.md) | 2026-06-02 | Top-K accepted-with-caveat results (current accepted baseline: job `be4ba9ee`). |
| 32 | [`docs/cycle_9b_child_critical_path_results.md`](docs/cycle_9b_child_critical_path_results.md) | 2026-06-02 | B.3 Step 1 measurement vs pre-Top-K baseline. |
| 33 | [`docs/cycle_9b_child_critical_path_remeasure_topk_results.md`](docs/cycle_9b_child_critical_path_remeasure_topk_results.md) | 2026-06-02 | B.3 Step 1 **remeasurement** against the accepted Top-K baseline (most recent — read this for the next-Step-2 lever ranking). |
| 33a | [`docs/cycle_9b_compact_postproc_investigation.md`](docs/cycle_9b_compact_postproc_investigation.md) | 2026-06-02 | B.1 compact-postprocessing investigation against the current accepted Top-K baseline. |
| 33b | [`docs/cycle_9b_compact_postproc_results.md`](docs/cycle_9b_compact_postproc_results.md) | 2026-06-02 | B.1 production comparison matrix placeholder; fill only after real benchmarks run. |
| 33c | [`docs/cycle_9b_batch_window_investigation.md`](docs/cycle_9b_batch_window_investigation.md) | 2026-06-02 | B.4 batch-window hypothesis and RSS guardrails before the rejected max-frames benchmark. |
| 33d | [`docs/cycle_9b_batch_window_results.md`](docs/cycle_9b_batch_window_results.md) | 2026-06-02 | B.4 production result: NOT ACCEPTED because track/model-agreement gates failed. |
| 34 | [`docs/logical_path_matrix_spec.md`](docs/logical_path_matrix_spec.md) | 2026-06-01 | Cycle 10 LPM formal mathematical spec (C1–C4 constraints, acceptance gates). |
| 35 | [`docs/cycle_10_investigation.md`](docs/cycle_10_investigation.md) | 2026-06-01 | Cycle 10 LPM investigation. |
| 36 | [`docs/cycle_10_lpm_phase1_results.md`](docs/cycle_10_lpm_phase1_results.md) | 2026-06-02 | Cycle 10 LPM Phase 1 NOT-ACCEPTED outcome + safety-fix re-run. |
| 37 | [`docs/new_models_yoloe_depth_anything_v2_timing_decision.md`](docs/new_models_yoloe_depth_anything_v2_timing_decision.md) | 2026-06-01 | YOLOE / Depth Anything v2 timing-of-integration decision (DEFERRED until SLA reached). |
| 37b | [`docs/cycle_11_input_size_investigation.md`](docs/cycle_11_input_size_investigation.md) | 2026-06-02 | Cycle 11 investigation (320 → 256 input). The Step 2 lever-attack design. |
| 37c | [`docs/cycle_11_input_size_results.md`](docs/cycle_11_input_size_results.md) | 2026-06-02 | Cycle 11.A real benchmark result: 256 input improved Step 2/RTT but was not accepted because persisted behavior signals regressed. |

### Phase 4 — Triton-specific deep dives

| # | File | Last updated | Why read this here |
|---|---|---|---|
| 38 | [`docs/triton_inference_speed_stabilization_plan.md`](docs/triton_inference_speed_stabilization_plan.md) | 2026-05-25 | Triton stabilization plan covering memory pool sweep + readiness gates. |
| 39 | [`docs/triton_strategy_implementation_matrix.md`](docs/triton_strategy_implementation_matrix.md) | 2026-05-15 | Strategy matrix for Triton routing decisions per model. |
| 40 | [`docs/triton_throughput_latency_gpu_optimization.md`](docs/triton_throughput_latency_gpu_optimization.md) | 2026-05-10 | Triton GPU-utilization optimization guide. |
| 41 | [`docs/triton_phase0_baseline_lock_report.md`](docs/triton_phase0_baseline_lock_report.md) | 2026-05-22 | Phase 0 baseline lock for the Triton plan. |

### Phase 5 — Runtime / deployment / system tuning

| # | File | Last updated | Why read this here |
|---|---|---|---|
| 42 | [`INFRASTRUCTURE_DEPLOYMENT_TUNING.md`](INFRASTRUCTURE_DEPLOYMENT_TUNING.md) | 2026-05-29 | Production deployment infrastructure tuning. |
| 43 | [`SYSTEM_OPTIMIZATION_AND_TUNING.md`](SYSTEM_OPTIMIZATION_AND_TUNING.md) | 2026-05-29 | OS-level + service-level system tuning. |
| 44 | [`OPTIMIZATION_QUICK_REFERENCE.md`](OPTIMIZATION_QUICK_REFERENCE.md) | 2026-05-29 | Quick-reference card for the optimization knobs that exist. |
| 45 | [`THREADING_AND_MULTIPROCESSING_ANALYSIS.md`](THREADING_AND_MULTIPROCESSING_ANALYSIS.md) | 2026-05-30 | Threading / multiprocessing decisions in the Python pipeline. |
| 46 | [`docs/heterogeneous_production_runtime_maturity_plan.md`](docs/heterogeneous_production_runtime_maturity_plan.md) | 2026-05-29 | Production-runtime maturity plan across heterogeneous backends. |
| 47 | [`docs/hybrid_runtime_inference_adventure.md`](docs/hybrid_runtime_inference_adventure.md) | 2026-05-22 | Hybrid local + Triton inference exploration journal. |
| 48 | [`docs/linux_production_optimization_execution_phases.md`](docs/linux_production_optimization_execution_phases.md) | 2026-05-29 | Linux production optimization phased execution log. |
| 49 | [`docs/runtime_v2_optimization_log.md`](docs/runtime_v2_optimization_log.md) | 2026-05-22 | Runtime v2 optimization log. |
| 50 | [`docs/runtime_ux_pass.md`](docs/runtime_ux_pass.md) | 2026-05-13 | Runtime UX pass. |
| 50a | [`docs/heterogeneous_production_runtime_maturity/tasks.md`](docs/heterogeneous_production_runtime_maturity/tasks.md) | 2026-05-29 | Per-task list for the heterogeneous-runtime maturity plan (mirror of plan §). |
| 50b | [`docs/runtime_stability_remediation/plan.md`](docs/runtime_stability_remediation/plan.md) | 2026-05-29 | Runtime stability remediation plan (job lifecycle + vector integrity). Backstop for constitution § 17. |
| 50c | [`docs/runtime_stability_remediation/tasks.md`](docs/runtime_stability_remediation/tasks.md) | 2026-05-29 | Remediation task list. |
| 50d | [`docs/production/behavioral_ontology_v1.md`](docs/production/behavioral_ontology_v1.md) | 2026-05-25 | Behavioral ontology v1 — production label semantics. |
| 50e | [`docs/production/runtime_maturity_operator_checklist.md`](docs/production/runtime_maturity_operator_checklist.md) | 2026-05-27 | Operator-facing runtime-maturity checklist. |
| 50f | [`docs/production/temporal_sequence_retention.md`](docs/production/temporal_sequence_retention.md) | 2026-05-25 | Temporal-sequence retention policy. |

### Phase 6 — Adjacent feature work (pose, models, identity)

| # | File | Last updated | Why read this here |
|---|---|---|---|
| 51 | [`docs/pose_eval_workflow_step5.md`](docs/pose_eval_workflow_step5.md) | 2026-05-23 | Pose evaluation workflow. |
| 52 | [`docs/pose_quality_occlusion_per_student_stability_plan.md`](docs/pose_quality_occlusion_per_student_stability_plan.md) | 2026-05-22 | Pose-quality + occlusion + per-student stability plan. |
| 53 | [`docs/upload_pose_inference_activation_plan.md`](docs/upload_pose_inference_activation_plan.md) | 2026-05-22 | Upload-pose inference activation plan. |
| 54 | [`docs/yolo26_pose.md`](docs/yolo26_pose.md) | 2026-06-02 | YOLO26 pose notes. |
| 55 | [`rtmpose_plan.md`](rtmpose_plan.md) | 2026-05-17 | RTMPose integration plan. |
| 56 | [`rtmpose_tasks.md`](rtmpose_tasks.md) | 2026-05-21 | RTMPose task breakdown. |
| 57 | [`docs/openvino_ultralytics_integration_checklist.md`](docs/openvino_ultralytics_integration_checklist.md) | 2026-05-22 | OpenVINO + Ultralytics integration checklist (DEV-only path). |
| 58 | [`docs/openvino_v2_sweep_results.md`](docs/openvino_v2_sweep_results.md) | 2026-05-22 | OpenVINO v2 sweep results. |
| 59 | [`docs/teacher_student_rollout.md`](docs/teacher_student_rollout.md) | 2026-05-11 | Teacher / student detector rollout. |
| 60 | [`docs/teacher_student_rollout_independent_system.md`](docs/teacher_student_rollout_independent_system.md) | 2026-05-11 | Teacher / student rollout as an independent system. |
| 61 | [`docs/identity_consistency_frontend_backend_plan.md`](docs/identity_consistency_frontend_backend_plan.md) | 2026-05-22 | Identity-consistency plan between frontend + backend. |

### Phase 7 — Operations & infrastructure

| # | File | Last updated | Why read this here |
|---|---|---|---|
| 62 | [`docs/docker-compose.dev.md`](docs/docker-compose.dev.md) | 2026-05-22 | Development docker-compose layout. |
| 63 | [`docs/nginx.md`](docs/nginx.md) | 2026-05-09 | Nginx reverse-proxy config notes. |
| 64 | [`docs/go2rtc.md`](docs/go2rtc.md) | 2026-05-09 | go2rtc streaming bridge notes. |
| 65 | [`docs/multimodal.md`](docs/multimodal.md) | 2026-06-02 | Multimodal integration notes. |
| 66 | [`docs/protocol_decision_harness.md`](docs/protocol_decision_harness.md) | 2026-05-22 | Protocol-decision harness. |
| 67 | [`docs/protocol_decision_summary_example.md`](docs/protocol_decision_summary_example.md) | 2026-05-22 | Protocol-decision summary example. |
| 67a | [`docs/infra/docker/triton/Dockerfile.md`](docs/infra/docker/triton/Dockerfile.md) | 2026-05-09 | Triton Dockerfile commentary (dev profile only; production is no-Docker). |
| 67b | [`docs/infra/systemd/triton-server.md`](docs/infra/systemd/triton-server.md) | 2026-05-09 | Triton systemd-service notes for prod runtime. |
| 67c | [`tools/prod/README.md`](tools/prod/README.md) | 2026-06-02 | Index of every script under `tools/prod/`. Read before invoking any prod helper. |
| 67d | [`scripts/ci/specs/README.md`](scripts/ci/specs/README.md) | 2026-05-22 | CI spec helpers index. |
| 67e | [`scripts/models/README.md`](scripts/models/README.md) | 2026-05-10 | Model-management helper scripts index. |

### Phase 8 — Release, gates, and miscellaneous

| # | File | Last updated | Why read this here |
|---|---|---|---|
| 68 | [`docs/release-beta-signoff-checklist.md`](docs/release-beta-signoff-checklist.md) | 2026-05-21 | Beta release sign-off checklist (consolidated quality state). |
| 69 | [`docs/diagram_compiler_plan.md`](docs/diagram_compiler_plan.md) | 2026-05-22 | Mermaid diagram compiler plan. |
| 70 | [`docs/mermaid_drift_audit.md`](docs/mermaid_drift_audit.md) | 2026-05-22 | Diagram drift audit. |
| 71 | [`closure_gaps.md`](closure_gaps.md) | 2026-05-27 | Closure-gap log across recent feature batches. |
| 72 | [`docs/Tasks.md`](docs/Tasks.md) | 2026-05-22 | High-level tasks register. |
| 72a | [`waves/maturity_closure_plan.md`](waves/maturity_closure_plan.md) | 2026-05-25 | Production-maturity wave closure plan. |

### Phase 9 — App and component READMEs (skim once, return for details)

| # | File | Last updated | Why read this here |
|---|---|---|---|
| 73 | [`backend/README.md`](backend/README.md) | 2026-05-24 | Backend overview, environment setup, run commands. |
| 74 | [`backend/apps/pipeline/README.md`](backend/apps/pipeline/README.md) | 2026-05-25 | Pipeline app: model lifecycle, layers, orchestrator. |
| 75 | [`backend/apps/video_analysis/README.md`](backend/apps/video_analysis/README.md) | 2026-05-23 | Video analysis app: offline jobs + Celery tasks. |
| 76 | [`backend/apps/tracking/README.md`](backend/apps/tracking/README.md) | 2026-05-25 | Tracking app: ByteTrack / BoT-SORT + ReID. |
| 77 | [`backend/apps/telemetry/README.md`](backend/apps/telemetry/README.md) | 2026-06-01 | Telemetry layer: sessions, dual-sink writer, ContextVar boundaries. |
| 78 | [`backend/apps/cameras/README.md`](backend/apps/cameras/README.md) | 2026-05-13 | Cameras + go2rtc + ONVIF. |
| 79 | [`backend/apps/sessions/README.md`](backend/apps/sessions/README.md) | 2026-05-08 | Monitoring sessions app. |
| 80 | [`backend/apps/anomalies/README.md`](backend/apps/anomalies/README.md) | 2026-05-08 | Anomaly triage app. |
| 81 | [`backend/apps/behavior/README.md`](backend/apps/behavior/README.md) | 2026-05-25 | Behavior app. |
| 82 | [`backend/apps/detections/README.md`](backend/apps/detections/README.md) | 2026-03-05 | Detections app. |
| 83 | [`backend/apps/health/README.md`](backend/apps/health/README.md) | 2026-05-08 | Health app. |
| 84 | [`backend/apps/contracts/README.md`](backend/apps/contracts/README.md) | 2026-05-25 | Cross-app data contracts. |
| 85 | [`backend/apps/telemetry_mcp/README.md`](backend/apps/telemetry_mcp/README.md) | 2026-05-23 | Telemetry MCP integration. |
| 86 | [`frontend/README.md`](frontend/README.md) | 2026-03-05 | Frontend overview. |
| 86a | [`backend/apps/pipeline/layers/README.md`](backend/apps/pipeline/layers/README.md) | 2026-04-30 | Pipeline layers sub-app: preprocessing, postprocessing, dispatch. |
| 86b | [`backend/apps/pipeline/model_lifecycle/README.md`](backend/apps/pipeline/model_lifecycle/README.md) | 2026-04-30 | Model lifecycle sub-app: loading, warming, eviction. |
| 86c | [`frontend/WCAG_AUDIT.md`](frontend/WCAG_AUDIT.md) | 2026-05-23 | Frontend WCAG accessibility audit. |
| 86d | [`frontend/src/AUDIT_FRONTEND_ASYNC_A11Y_RESPONSIVE.md`](frontend/src/AUDIT_FRONTEND_ASYNC_A11Y_RESPONSIVE.md) | 2026-05-23 | Frontend async + a11y + responsive audit. |
| 86e | [`frontend/src/api/README.md`](frontend/src/api/README.md) | 2026-05-25 | Frontend REST client / Axios wrapper. |
| 86f | [`frontend/src/hooks/README.md`](frontend/src/hooks/README.md) | 2026-03-05 | Frontend hooks index (WS, WHEP, paged data). |
| 86g | [`frontend/src/components/camera/README.md`](frontend/src/components/camera/README.md) | 2026-03-05 | Camera UI components. |
| 86h | [`frontend/src/constants/README.md`](frontend/src/constants/README.md) | 2026-03-05 | Frontend constants. |
| 86i | [`frontend/src/features/bsil/README.md`](frontend/src/features/bsil/README.md) | 2026-05-27 | BSIL feature module (Behavioral Semantic Inference Layer). |

### Phase 10 — Academic paper sources

These docs are the source-of-truth chapters for the research paper.
They cite real files / lines from this repo per constitution § 19.6
(every Evidence: anchor maps to a `<file>:<line>` reference).

| # | File | Last updated | Why read this here |
|---|---|---|---|
| 87 | [`paper/01_architecture_deployment.md`](paper/01_architecture_deployment.md) | 2026-06-02 | Architecture + deployment chapter (modular boundaries, Triton authority). |
| 88 | [`paper/02_ingestion_orchestration.md`](paper/02_ingestion_orchestration.md) | 2026-06-02 | Ingestion + orchestration chapter (Celery, pipeline, dispatch). |
| 89 | [`paper/03_detection_tracking.md`](paper/03_detection_tracking.md) | 2026-06-02 | Detection + tracking chapter (YOLO + ByteTrack + BoT-SORT + ReID). |
| 90 | [`paper/04_pose_temporal_artifacts.md`](paper/04_pose_temporal_artifacts.md) | 2026-06-02 | Pose + temporal artifacts chapter (RTMPose + occlusion handling). |
| 91 | [`paper/05_data_api_frontend.md`](paper/05_data_api_frontend.md) | 2026-06-02 | Data + API + frontend chapter (REST, WS, WHEP, schemas). |
| 92 | [`paper/06_observability_testing_science.md`](paper/06_observability_testing_science.md) | 2026-06-02 | Observability + testing + scientific evidence chapter. |
| 93 | [`paper/coverage_matrix.md`](paper/coverage_matrix.md) | 2026-06-02 | Per-section paper coverage matrix. |
| 94 | [`paper/latex/figures_manifest.md`](paper/latex/figures_manifest.md) | 2026-06-02 | LaTeX figures manifest. |
| 95 | [`paper/latex/equations_manifest.md`](paper/latex/equations_manifest.md) | 2026-06-02 | LaTeX equations manifest. |
| 96 | [`paper/latex/traceability_matrix.md`](paper/latex/traceability_matrix.md) | 2026-06-02 | LaTeX traceability matrix (figure/equation → claim). |

### Phase 11 — DSP entity docs (one per System / Module / Phase / Script / Code / API)

These are the per-entity deep-dive docs produced by DSP Cycles 2-7
(see [`docs/documentation_systematization_plan.md`](docs/documentation_systematization_plan.md)
§ 1, § 2 and constitution § 19.2). Each follows
[`docs/per_entity_doc_template.md`](docs/per_entity_doc_template.md)
exactly and carries a source-of-truth references block the CI gate
validates against the working tree.

| # | File | Last updated | Why read this here |
|---|---|---|---|
| 97 | [`docs/entity/systems/offline_inference_pipeline.md`](docs/entity/systems/offline_inference_pipeline.md) | 2026-06-02 | DSP Cycle 2 — first system entity doc. Celery-driven offline video pipeline; current accepted baseline is Cycle 9b Top-K (job `be4ba9ee`, 4.43 FPS, 9.5-min SLA gap). |

### Conventions used in this reading order

- **Reading order is authoritative.** If a file has an old `**Last updated:**` date but appears in a later phase, that simply means the doc has been stable; it does NOT mean it can be skipped.
- Every file above carries a `**Last updated:** YYYY-MM-DD` header at line 2 or 3. If you ever see one missing, run `python scripts/add_doc_date_header.py <path>` to repopulate it from the file's last git commit date.
- The reading order intentionally skips auto-generated mirror docs under `docs/backend/`, `docs/frontend/`, `docs/scripts/`, and `docs/diagrams/` — those are produced by `scripts/ci/verify_docs_diagrams.py` and exist for one-page-per-source-file traceability, not narrative reading. Browse them from [`docs/INDEX.md`](docs/INDEX.md) when you need a specific file's per-source documentation.
- Point-in-time evidence reports under `ci_evidence/` and feature specs under `specs/` are also not in the reading order — they are referenced from the relevant cycle / feature docs above.

---

## Repository Status

- Active local branches: `master`, `006-modular-low-coupling`
- Primary remote branches present: `origin/001-exam-monitor-dashboard`, `origin/master`, `origin/006-modular-low-coupling`
- Documentation coverage status: the authored backend, frontend, script, and infrastructure inventory currently maps to `docs/` with `Missing docs: 0`
- Required documentation gate: `scripts/ci/verify_docs_diagrams.py` currently passes

## Documentation Map

- [Documentation index](docs/INDEX.md)
- [System Mermaid atlas](docs/diagrams/SYSTEM_MERMAID_ATLAS.md)
- [Source file mirror](docs/diagrams/SOURCE_FILE_MIRROR.md)
- [Backend architecture docs](docs/backend/architecture/)
- [Frontend docs](docs/frontend/)
- [Script docs](docs/scripts/)
- [Infrastructure docs](docs/infra/)

```mermaid
flowchart LR
    Root["README.md + quickstart.md"] --> Atlas["docs/diagrams/SYSTEM_MERMAID_ATLAS.md"]
    Root --> Mirror["docs/diagrams/SOURCE_FILE_MIRROR.md"]
    Atlas --> Backend["backend apps, tests, scripts, infrastructure"]
    Atlas --> Frontend["frontend pages, components, hooks, stores, tests"]
    Mirror --> Files["tracked source file roles"]
    Files --> Flows["cross-file implementation flows"]
```

The flowchart shows how the root project documentation fans out into the architecture atlas, inventory mirror, and per-entity reference pages.

## Implemented System Areas

- Authentication, roles, and admin account management
- Cameras, go2rtc registration, and live feed status
- Monitoring sessions and dashboard views
- Live detections, anomaly triage, and comments
- Recordings and session exports
- Offline video upload, processing, playback, and overlay visibility
- Shared pipeline, tracking, and runtime policy logic
- Health monitoring and release-gate verification scripts

## High-Level Architecture

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart LR
    subgraph Client["Client (React 19 SPA / Vite)"]
        SPA["Pages: dashboard · video-analysis · camera-feed · sessions · runtime · anomalies · health"]
        Hooks["WebSocket + WHEP hooks"]
    end

    subgraph Edge["Gateway / Edge (Nginx)"]
        REST["/api/v1/*"]
        WS["/ws/*"]
        WHEP["/whep/{camera_id}/"]
    end

    subgraph Backend["Backend (Django ASGI + Channels)"]
        API["REST surfaces: auth · cameras · sessions · video-analysis · anomalies · behavior · health · admin"]
        VA["VideoAnalysis + runtime analytics + per-job telemetry APIs"]
        CAM["CameraService + StreamGateway"]
        CH["WS groups: detections · anomalies · health · cameras · video-analysis · sessions"]
    end

    subgraph Async["Async Orchestration (Celery prefork + beat)"]
        OFFLINE["process_video_upload (offline)"]
        LIVE["run_live_stream_inference (live)"]
        BEAT["beat: stale-job reconciler · metrics flush · camera health"]
    end

    subgraph Offline["Offline Frame Pipeline (_run_triton_frame_level_inference)"]
        DECODE["FrameReadPipeline (threaded decode)"]
        PREP["letterbox + preprocess"]
        BQ["async batch queue (max_concurrency · max_inflight · dynamic batching)"]
        TRACKO["ByteTrack / BoT-SORT + ReID cache"]
    end

    subgraph AI["Inference Plane — Triton-only in production"]
        ORCH["InferenceOrchestrator + ModelRouteService"]
        TRITON["Triton Inference Server (HTTP/gRPC · TensorRT .plan engines)"]
        MODELS6["person_detector · posture · gaze x3 · rtmpose"]
        LOCAL["Local adapters (OpenVINO / ONNX / Ultralytics) — DEV ONLY"]
    end

    subgraph Stream["Streaming Layer"]
        GO2RTC["go2rtc relay (API + WHEP)"]
        CAMSRC["RTSP / ONVIF cameras"]
    end

    subgraph Tel["Telemetry Layer (apps.telemetry)"]
        TS["TelemetrySession (Celery signals + ContextVar)"]
        SINK["dual sink: PostgreSQL + JSON file (JSON-first fallback)"]
    end

    subgraph Data["Data / Storage"]
        PG["PostgreSQL: jobs · detections · tracks · telemetry_sessions/videos/frames/model_calls/students"]
        REDIS["Redis: channel layer · Celery broker/result · toggles · tracking IDs · embeddings"]
        FILES["Filesystem: data/videos · data/sessions · logs/telemetry/*.json · inference_audit.json"]
        ART["Model artifacts: triton_repository_cuda12 (.plan)"]
    end

    subgraph Ops["Ops / Deployment"]
        HEALTH["/api/v1/health + model-serving checks"]
        BENCH["prod_run_benchmark.sh · runtime_ingest_video"]
        SYSD["systemd / tools/prod (Triton + Celery + beat lifecycle)"]
    end

    SPA --> Hooks
    SPA --> REST --> API
    SPA --> WS --> CH
    Hooks --> WHEP --> GO2RTC

    API --> VA
    API --> CAM
    VA --> OFFLINE
    CAM --> LIVE
    CH --> REDIS

    OFFLINE --> DECODE --> PREP --> BQ --> ORCH
    OFFLINE --> TRACKO
    LIVE --> ORCH
    ORCH --> TRITON --> MODELS6
    ORCH -. dev fallback .-> LOCAL

    CAMSRC -->|RTSP/ONVIF| CAM
    CAM --> GO2RTC
    LIVE -->|reads stream URL| GO2RTC

    OFFLINE --> TS
    LIVE --> TS
    ORCH --> TS
    TS --> SINK
    SINK --> PG
    SINK --> FILES

    API --> PG
    OFFLINE --> PG
    LIVE --> PG
    TRACKO --> REDIS
    BEAT --> PG
    TRITON --> ART

    HEALTH --> TRITON
    HEALTH --> API
    BENCH --> OFFLINE
    SYSD --> TRITON
```

This diagram reflects the **currently implemented** runtime contracts in code.
Video-analysis and camera APIs launch Celery `process_video_upload` (offline) and
`run_live_stream_inference` (live) pipelines. The offline pipeline decodes frames
on a background thread (`FrameReadPipeline`), preprocesses them, and feeds an
**async batch queue** that dispatches concurrent, dynamically-batched requests to
Triton through the `InferenceOrchestrator` + `ModelRouteService`. Every run is
captured by the **telemetry layer** (`apps.telemetry`) via Celery signals and a
ContextVar, written atomically to PostgreSQL tables and a JSON file. Tracking
(ByteTrack/BoT-SORT) and ReID embeddings are cached in Redis; Celery **beat**
drives the stale-job reconciler and periodic maintenance. Under
`.specify/memory/constitution.md`, the local OpenVINO/ONNX/Ultralytics adapters
are **development-only** — production inference is native-Linux, GPU-backed,
**Triton-only**, with exactly one active live or offline endpoint profile at a time.

## Triton Model Inference End to End lifecycle (Sequential Order )

This is the **currently implemented** offline inference lifecycle, in the real
execution order: trigger → Celery task boot → runtime-policy decision → the
Triton-only frame pipeline (threaded decode → preprocess → async batch queue →
Triton/TensorRT) → per-frame side effects (DB progress, WebSocket overlay,
telemetry) → post-detection stages (tracking → pose → embeddings/ReID →
persistence → render) → telemetry session flush. The hybrid local path is shown
dashed; it is development-only and bypassed in production.

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TD
    %% ---------- Trigger ----------
    subgraph Trigger["1 · Trigger"]
        CLI["manage.py runtime_ingest_video / prod_run_benchmark.sh"]
        API["POST /api/v1/video-analysis/jobs/ (upload)"]
    end
    CLI --> ENQ
    API --> ENQ
    ENQ["Celery enqueue → process_video_upload\n(queue: offline_control)"]

    %% ---------- Task boot ----------
    subgraph Boot["2 · Task boot (Celery worker, prefork)"]
        SIG["task_prerun signal →\nTelemetrySession.start() bound via ContextVar"]
        VAL["validate video · read enabled-model toggles (Redis)"]
        POL["evaluate_runtime_policy(path=offline)\nINFERENCE_STRATEGY, triton_ready, allow_local_fallback"]
    end
    ENQ --> SIG --> VAL --> POL
    POL --> SHADOW["Triton shadow precheck\n(1 frame → orchestrator, per model)"]
    SHADOW --> BR{"allow_local_fallback?"}

    %% ---------- Triton-only path ----------
    BR -- "False (triton_only / prod)" --> TRP
    subgraph TRP["3 · _run_triton_frame_level_inference  (PRODUCTION PATH)"]
        ORCHB["build InferenceOrchestrator\n+ ModelRouteService + TritonClient"]
        DEC["FrameReadPipeline (background thread)\ndisk read + cv2 decode"]
        PRE["letterbox 640x640 · BGR→RGB · NCHW float32"]
        CAD{"detect this frame?\nOFFLINE_DETECT_EVERY_N_FRAMES"}
        REUSE["reuse last person boxes\n(TTL frames) + interpolate"]
        QUE["pending_frames queue\n_flush_pending: batch (TARGET_BATCH)\nmax_concurrency / max_inflight"]
        DISP["concurrent dispatch → Triton\n(dynamic batching, HTTP/gRPC)"]
        DECODE["decode YOLO output0 → DetectionBox\n(per model: person/posture/gaze x3)"]
        FD["FrameDetections[frame_index]"]
        ORCHB --> DEC --> PRE --> CAD
        CAD -- "skip" --> REUSE --> FD
        CAD -- "run" --> QUE --> DISP --> DECODE --> FD
    end

    %% ---------- Triton serving ----------
    subgraph TS["Triton Inference Server (GPU, native Linux)"]
        TR["HTTP 39100 / gRPC 39101"]
        ENG["TensorRT .plan engines (FP16, batch 1..8..32)\nperson_detector · posture_model\ngaze_horizontal/vertical/depth · rtmpose"]
        TR --> ENG
    end
    DISP <-->|"infer() RTT"| TR

    %% ---------- Per-frame side effects ----------
    FD --> CB["on_frame_inferred callback"]
    CB --> DBP["DB: VideoAnalysisJob.update\nprocessed_frames · progress_percent"]
    CB --> WS["WebSocket 'overlay_frame'\n(video-analysis group)"]
    CB --> TFRAME["TelemetrySession.record_frame(FrameMeta)"]
    DISP --> TCALL["TelemetrySession.record_model_call\n(ModelCallMeta: model, rtt_ms, status)"]

    %% ---------- Post-detection stages ----------
    FD --> TRACK
    subgraph POST["4 · Post-detection (per job)"]
        TRACK["_assign_tracking_ids\nByteTrack / BoT-SORT"]
        POSE["PoseRuntime → rtmpose ROI keypoints\n+ pose quality"]
        EMB["embeddings: extract_crop → vector\nReID find_best_match (cosine)"]
        PERSIST["persist: Detection · BoundingBox · StudentTrack"]
        RENDER["render_aggregated_video\n(annotated + pose overlays)"]
        TRACK --> POSE --> EMB --> PERSIST --> RENDER
    end

    %% ---------- Stores ----------
    subgraph DATA["Stores"]
        PG[("PostgreSQL\njobs · detections · tracks\ntelemetry_*")]
        REDIS[("Redis\ntoggles · tracking IDs · embeddings\nCelery broker/result")]
        FILES[("Filesystem\ndata/videos · predict/input_frames/*.jpg\nlogs/telemetry/*.json · rendered mp4")]
    end
    DBP --> PG
    EMB --> REDIS
    TRACK --> REDIS
    PERSIST --> PG
    RENDER --> FILES
    DEC --> FILES

    %% ---------- Session end ----------
    RENDER --> END["task_postrun signal →\nTelemetrySession.end()"]
    END --> SINK["writer: JSON-first + PostgreSQL\n(db_commit_failed fallback)"]
    SINK --> PG
    SINK --> FILES
    TFRAME -.-> END
    TCALL -.-> END

    %% ---------- Hybrid fallback (non-prod) ----------
    BR -- "True (hybrid / dev only)" --> HYB["run_multi_model_inference_streaming\nLOCAL TensorRT/OpenVINO engines (multi_model.py)\n→ same on_frame_complete callback"]
    HYB -. "merges into" .-> TRACK

    classDef prod fill:#5B21B6,stroke:#A78BFA,color:#EDE9FE;
    classDef store fill:#1E3A8A,stroke:#60A5FA,color:#DBEAFE;
    class TRP,TS prod;
    class DATA store;
```

> **Known limitation (sequential):** within a frame-batch the 5–6 models are
> dispatched one after another (`for task_key in task_keys`), so per-batch
> latency is the **sum** of model RTTs; frame-batches and post-stages also run
> serially, and tensors are JSON-encoded. These are the targets of the
> parallelization plan below.

## Triton Model Inference End to End (Parallel )

This is the **intended parallelized** design (see
[docs/inference_parallelization_plan.md](docs/inference_parallelization_plan.md),
the project's current **highest-priority** performance plan). It overlaps the
decode/preprocess, dispatch, and post/persist stages (double/triple buffering),
**fans out all models concurrently** with `asyncio.gather` so per-batch latency
is the **max** model RTT instead of the sum, sends **binary tensors over gRPC**
(no JSON `.tolist()`), lets **Triton dynamic batching** coalesce concurrent
requests into large GPU batches, **batches DB writes**, and **offloads** pose /
embeddings / render to dedicated Celery queue workers.

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TD
    ENQ["process_video_upload (offline_control)\nTelemetrySession.start()"] --> PROD

    %% ---------- Stage A: produce (threads) ----------
    subgraph PROD["Stage A · Decode + Preprocess (thread pool, runs ahead)"]
        DEC["FrameReadPipeline\ndisk read + cv2 decode"]
        PRE["letterbox 640 · NCHW float32\nBINARY tensor bytes (no JSON .tolist)"]
        DEC --> PRE
    end
    PRE --> RB[("bounded prepared-frame ring buffer\n(backpressure decouples A from B)")]

    %% ---------- Stage B: concurrent dispatch ----------
    RB --> BATCH["Stage B · accumulate batch (TARGET_BATCH=32)"]
    BATCH --> FORK{{"asyncio.gather — fan out ALL models concurrently"}}
    FORK --> M1["person_detector"]
    FORK --> M2["posture_model"]
    FORK --> M3["gaze_horizontal"]
    FORK --> M4["gaze_vertical"]
    FORK --> M5["gaze_depth"]
    FORK --> M6["rtmpose"]
    M1 --> TR
    M2 --> TR
    M3 --> TR
    M4 --> TR
    M5 --> TR
    M6 --> TR

    subgraph GPU["Triton (gRPC 39101 · HTTP/2 multiplexed)"]
        TR["dynamic batching coalesces\nconcurrent requests"]
        ENGV["TensorRT FP16 engines (batch 1..8..32)\nGPU saturated (>50% util)"]
        TR --> ENGV
    end
    TR --> JOIN{{"join — batch latency = MAX(model RTT), not SUM"}}
    JOIN --> DECODE["decode outputs → FrameDetections"]

    %% ---------- Stage C: parallel post / persist ----------
    DECODE --> TRACK["Stage C · tracking (ByteTrack / BoT-SORT)"]
    TRACK --> DBW["batched DB writer\nbulk_create Detections/BBoxes\nprogress throttled every N frames"]
    subgraph OFFLOAD["Offloaded to dedicated Celery queues (parallel workers)"]
        POSEW["pipeline.offline.rtmpose_model.worker → pose"]
        EMBW["pipeline.offline.behavior.worker → embeddings / ReID"]
        RENW["render worker → annotated mp4"]
    end
    TRACK --> POSEW
    TRACK --> EMBW
    TRACK --> RENW

    %% ---------- Telemetry (off hot path) ----------
    subgraph TEL["Telemetry (async, off the hot path)"]
        RTT["record_model_call (per-model rtt_ms)"]
        FRM["record_frame (stage latencies)"]
    end
    M1 -.-> RTT
    DECODE -.-> FRM

    subgraph DATA["Stores"]
        PG[("PostgreSQL")]
        REDIS[("Redis · tracks · embeddings")]
        FILES[("Filesystem · frames · telemetry JSON · mp4")]
    end
    DBW --> PG
    EMBW --> REDIS
    POSEW --> FILES
    RENW --> FILES

    DBW --> ENDS["TelemetrySession.end() → JSON-first + PostgreSQL"]
    ENDS --> PG
    ENDS --> FILES

    classDef par fill:#5B21B6,stroke:#A78BFA,color:#EDE9FE;
    classDef store fill:#1E3A8A,stroke:#60A5FA,color:#DBEAFE;
    class PROD,GPU,OFFLOAD par;
    class DATA store;
```

> Stages A/B/C run **overlapped**: while batch *K* is inferring (B), batch *K+1*
> is decoding/preprocessing (A) and batch *K−1* is persisting (C). Expected
> trajectory: today ~0.5–5 fps (GPU ~1%) → Phase 1 (binary + concurrent models)
> ~15–40 fps → Phase 3 (gRPC) ~40–80 fps → Phase 4–5 (batched DB + GPU batching)
> 100+ fps with GPU >50%. Every phase is measured via the telemetry layer.

## Repository Layout

```text
grad_project/
  backend/                Django apps, config, core, tests, tools
  frontend/               React SPA, tests, Vite config
  docs/                   Mermaid-backed documentation suite
  infra/                  Docker and systemd support files
  scripts/                CI and Triton helper scripts
  specs/                  Feature specs and evidence
  docker-compose.dev.yml  Dev services
  nginx.conf              Reverse proxy rules
  go2rtc.yaml             Streaming bridge config
  quickstart.md           Step-by-step local setup
```

## Linux Deployment Custom Notes (No Docker, No sudo)

For constrained Linux deployment servers where Docker and sudo are not allowed, Triton and TensorRT require a custom user-space setup.

What is different from standard setup:

- Triton server is built from source in user space (no container build).
- TensorRT runtime is installed in project venv and linked through `LD_LIBRARY_PATH`.
- Extra third-party headers/libraries are vendored in user space:
  - RapidJSON
  - Boost headers
  - libb64 headers and static lib
  - TensorRT headers (for backend compilation)
- Triton TensorRT backend plugin is built separately and installed into Triton backends.

Recommended paths used in production deployment:

- Triton source: `/home/bamby/services/triton_src`
- Triton build tree: `/home/bamby/services/triton_build_r2502`
- Triton runtime binary: `/home/bamby/services/triton_build_r2502/tritonserver/install/bin/tritonserver`
- Triton model repository (CUDA12 profile): `/home/bamby/grad_project/backend/models/triton_repository_cuda12`

Key requirement:

- Always start Triton with explicit backend directory:
  - `--backend-directory=/home/bamby/services/triton_build_r2502/tritonserver/install/backends`
- Validate exactly one active production endpoint profile before admitting
  inference: live `39000/39001/39002` or offline `39100/39101/39102`.
- Treat local inference adapters as non-production only; unavailable Triton
  causes a fail-closed or explicitly non-authoritative degraded state.

See [quickstart.md](/E:/grad_project/quickstart.md) section `Linux Deployment Runbook (No Docker, No sudo)` for exact commands.

## Backend Surface

- Django 5.1.5
- Channels 4.2.2 with Daphne
- DRF 3.15.2
- Celery 5.4.0
- PostgreSQL 16
- Redis 7
- Ultralytics 8.3.61
- ONNX 1.21.0
- ONNX Runtime 1.25.1
- OpenVINO 2026.1.0

Installed backend apps from `backend/config/settings/base.py`:

- `apps.accounts`
- `apps.cameras`
- `apps.detections`
- `apps.anomalies`
- `apps.sessions`
- `apps.recordings`
- `apps.exams`
- `apps.exports`
- `apps.audit`
- `apps.health`
- `apps.pipeline`
- `apps.video_analysis`
- `apps.tracking`

## Frontend Surface

- React 19.2.0
- TypeScript 5.9
- Vite 7.3.1
- Zustand 5.0.11
- Axios 1.13.6
- React Router 7.13.1
- Vitest 4.0.18
- Playwright 1.58.2

Implemented protected routes from `frontend/src/App.tsx`:

- `/dashboard`
- `/sessions`
- `/sessions/:id`
- `/camera-feed`
- `/cameras`
- `/predictions`
- `/video-analysis`
- `/video-analysis/:jobId`
- `/anomalies`
- `/recordings`
- `/recordings/:id`
- `/health`
- `/settings`
- `/change-password`

Public route:

- `/login`

## Frontend Quality Gates (Spec 001 Cross-Cutting)

- UI consistency (FR-033): shared UI primitives are centralized in `frontend/src/components/ui` and consumed across pages/components (see `docs/frontend/src/components/README.md`).
- Loading states (FR-034): all async page/workflow fetches expose loading UX (`LoadingSpinner`, button busy states, or section placeholders).
- Keyboard navigation (FR-035): modal focus trap, skip-link to `#main-content`, focusable sidebar/header controls, and page-level route navigation remain keyboard-operable.
- Error handling (FR-036): page-level `ErrorBoundary` wrappers in `frontend/src/App.tsx` provide user-safe fallback messages and structured `console.error` diagnostics.
- 1920×1080 baseline (FR-037): app shell, header, and collapsible sidebar preserve functionality at minimum supported desktop resolution.
- Auto-restart policy (FR-049): dev compose services use `restart: unless-stopped` for supervisor-style automatic recovery.
- WCAG contrast audit (T164): run `cd frontend && npm run audit:contrast` to validate theme token pairs against AA (4.5:1) thresholds.

## Environment Files

The root `.env` is the active runtime file used by Django, Celery, pipeline services, and some shared frontend defaults. `.env.example` is the starter template. `frontend/.env.example` documents the frontend-only variables used by the Vite dev server and browser build.

The tables below document:

- every variable present in the checked-in root `.env.example`
- additional variables currently present in the active root `.env`
- the frontend-specific variables present in `frontend/.env.example`

### Root App And Service Variables

| Variable | Example / Default | Used by | Why it exists | Acceptable values | Role / operational effect |
| --- | --- | --- | --- | --- | --- |
| `DJANGO_SECRET_KEY` | `change-me` | `backend/config/settings/base.py` | Provides Django cryptographic signing for sessions, CSRF, and internal secrets. | Any long random string. Production should use a high-entropy secret. | Changing it invalidates existing signed sessions/tokens. Weak values reduce application security. |
| `DJANGO_DEBUG` | `True` in `.env.example` | Not currently consumed by the active settings modules | Present as a common Django convenience variable, but this repository currently decides debug mode from the selected settings module (`development.py` vs `production.py`). | `True` / `False` / `1` / `0` if you keep it for operator clarity. | At the moment it has no direct effect unless the settings layout is changed to read it. |
| `DJANGO_ALLOWED_HOSTS` | `localhost,127.0.0.1` | `backend/config/settings/base.py`, `backend/config/settings/production.py` | Controls which `Host` headers Django accepts. | Comma-separated hostnames/IPs, for example `localhost,127.0.0.1,example.com`. | Wrong values cause `DisallowedHost` errors; overly broad values weaken host-header protection. |
| `POSTGRES_DB` | `exam_monitor` | `backend/config/settings/base.py` | Selects the PostgreSQL database name. | Existing PostgreSQL database name string. | If it does not match the provisioned DB, backend startup and migrations fail. |
| `POSTGRES_USER` | `exam_user` | `backend/config/settings/base.py` | Selects the PostgreSQL login user. | Existing PostgreSQL username. | Wrong value prevents DB authentication. |
| `POSTGRES_PASSWORD` | `exam_pass` | `backend/config/settings/base.py` | Supplies the PostgreSQL password. | Password matching the selected DB user. | Wrong value prevents DB authentication. |
| `POSTGRES_HOST` | `localhost` | `backend/config/settings/base.py` | Tells Django where PostgreSQL is reachable. | Hostname, service name, or IP. Example: `localhost`, `db`. | Wrong host breaks backend DB connectivity. |
| `POSTGRES_PORT` | `5432` | `backend/config/settings/base.py` | Tells Django which PostgreSQL port to use. | Integer port, usually `5432`. | Wrong port breaks backend DB connectivity. |
| `REDIS_URL` | `redis://localhost:6379/0` | `backend/config/settings/base.py` | Shared Redis endpoint used by Django Channels and some app storage paths. | Valid Redis URL with DB index. Example `redis://host:6379/0`. | Wrong value breaks channel layers and any Redis-backed coordination. |
| `CELERY_BROKER_URL` | `redis://localhost:6379/1` | `backend/config/celery.py` | Supplies Celery's task broker endpoint. | Valid broker URL; current stack expects Redis. | Wrong value prevents task dispatch and background job execution. |
| `CELERY_RESULT_BACKEND` | `redis://localhost:6379/2` | `backend/config/celery.py` | Stores Celery task state/results. | Valid backend URL; current stack expects Redis. | Wrong value removes or corrupts async task result tracking. |
| `CELERY_WORKER_POOL` | `solo` on Windows, `prefork` otherwise | `backend/config/celery.py` | Lets operators choose the Celery worker execution pool. Present in the active `.env`, not in `.env.example`. | Typical values: `solo`, `prefork`, `threads`. | Bad values can prevent workers from starting; `solo` is safest on Windows but lower throughput. |
| `CELERY_WORKER_CONCURRENCY` | `1` on Windows or CPU-count on Linux | `backend/config/celery.py` | Overrides Celery worker concurrency. Present in the active `.env`, not in `.env.example`. | Positive integer. | Higher values increase parallelism but also CPU/RAM pressure. |
| `FIELD_ENCRYPTION_KEY` | `change-me-generate-a-fernet-key` | `backend/apps/cameras/fields.py`, `backend/apps/cameras/migrations/0003_encrypt_existing_urls.py` | Encrypts camera connection URLs at rest. | A valid Fernet key generated by `cryptography.fernet.Fernet.generate_key()`. | Missing or invalid values break encrypted camera field access and the URL-encryption migration. |
| `GO2RTC_API_URL` | `http://localhost:1984` | `backend/apps/cameras/services.py` | Backend endpoint used to register/query go2rtc stream state. | Base HTTP URL for the go2rtc API. | Wrong value prevents stream provisioning and status calls. |
| `GO2RTC_WHEP_URL` | `http://localhost:8555` | Shared runtime documentation; current frontend WHEP flow uses `/whep` proxy instead of reading this variable directly | Documents the WHEP bridge endpoint used by the streaming topology. | HTTP URL for the WHEP-capable go2rtc endpoint. | Keep it aligned with the deployed go2rtc WHEP listener even if the browser path is currently proxied. |
| `ONVIF_USERNAME` | empty by default | `backend/apps/cameras/services.py` (`OnvifResolver`) | Username for ONVIF camera device-service authentication. | Valid ONVIF account username. | Required when connecting cameras via ONVIF device-service URLs. |
| `ONVIF_PASSWORD` | empty by default | `backend/apps/cameras/services.py` (`OnvifResolver`) | Password for ONVIF camera device-service authentication. | Valid ONVIF account password. | Required when connecting cameras via ONVIF device-service URLs. |
| `ONVIF_WSDL_DIR` | empty by default | `backend/apps/cameras/services.py` (`OnvifResolver`) | Filesystem path to ONVIF WSDL definitions used by the ONVIF client. | Absolute directory path containing ONVIF WSDL files. | Required for ONVIF resolution; missing path causes ONVIF connection failure. |
| `VITE_API_BASE_URL` | `http://localhost:8000/api/v1` | `frontend/src/api/client.ts` | Sets the browser-facing REST API base URL. | Absolute URL or a same-origin path such as `/api/v1`. | Wrong value breaks all frontend REST calls. |
| `VITE_WS_BASE_URL` | `ws://localhost:8000/ws` in root `.env.example` | `frontend/src/hooks/useWebSocket.ts` | Sets the WebSocket base used by the monitoring hook. | `ws://...`, `wss://...`, or leave unset to derive from page origin. | Wrong value breaks live socket subscriptions and reconnect flow. |
| `VITE_GO2RTC_WHEP_URL` | `http://localhost:8555` | Shared root env template only | Legacy/shared root variable documenting the WHEP listener. The current frontend hook uses the `/whep/{camera_id}/` proxy route and `frontend/.env.example` uses `VITE_GO2RTC_URL` instead. | HTTP URL. | Keep only if you want one root env to describe all streaming endpoints; it is not the browser variable currently consumed by `useWhepClient.ts`. |

### Root Pipeline And Model Variables

| Variable | Example / Default | Used by | Why it exists | Acceptable values | Role / operational effect |
| --- | --- | --- | --- | --- | --- |
| `PYRAMID_MODELS_BASE_DIRECTORY` | `backend/models` | `backend/config/settings/base.py`, `backend/apps/pipeline/config.py` | Points the pipeline to the model artifact root. | Relative or absolute filesystem path. | Wrong path makes model resolution fail. |
| `PYRAMID_RAW_DATA_DIRECTORY` | `Raw Data` | `backend/config/settings/base.py`, `backend/apps/pipeline/config.py` | Points to raw dataset/test video material. | Relative or absolute filesystem path. | Wrong path breaks dataset-backed validation and some test/development flows. |
| `PYRAMID_INFERENCE_BACKEND` | `tensorrt` | `backend/apps/pipeline/config.py` | Chooses a local adapter for development or expressly non-production validation only. | `onnx`, `openvino`, `tensorrt`. | MUST NOT satisfy production inference authority. |
| `PYRAMID_PERSON_DETECTOR_PATH` | `student_teacher/weights/student_teacher.engine` | `backend/config/settings/base.py`, `backend/apps/pipeline/config.py` | Selects the person detector artifact. | Relative path under the models base directory or absolute path. | Wrong path breaks student/teacher detection. |
| `PYRAMID_PERSON_DETECTOR_RUNTIME` | `TensorRT` | `backend/apps/pipeline/config.py` | Per-model runtime override for the person detector. | `auto`, `onnx`, `openvino`, `tensorrt` and supported aliases. | Forces detector execution backend regardless of global backend. |
| `PYRAMID_POSTURE_MODEL_PATH` | `standing_sitting/weights/standing_sitting.engine` | `backend/config/settings/base.py`, `backend/apps/pipeline/config.py` | Selects the posture model artifact. | Valid model path. | Wrong path breaks standing/sitting classification. |
| `PYRAMID_POSTURE_MODEL_RUNTIME` | `TensorRT` | `backend/apps/pipeline/config.py` | Per-model runtime override for posture inference. | `auto`, `onnx`, `openvino`, `tensorrt`. | Overrides the runtime for this model family only. |
| `PYRAMID_HORIZONTAL_GAZE_MODEL_PATH` | `right_left/weights/right_left.engine` | `backend/config/settings/base.py`, `backend/apps/pipeline/config.py` | Selects the horizontal gaze model artifact. | Valid model path. | Wrong path breaks left/right gaze classification. |
| `PYRAMID_HORIZONTAL_GAZE_MODEL_RUNTIME` | `TensorRT` | `backend/apps/pipeline/config.py` | Per-model runtime override for horizontal gaze inference. | `auto`, `onnx`, `openvino`, `tensorrt`. | Overrides backend selection for this model family. |
| `PYRAMID_DEPTH_GAZE_MODEL_PATH` | `forward_backward/weights/forward_backward.engine` | `backend/config/settings/base.py`, `backend/apps/pipeline/config.py` | Selects the depth gaze model artifact. | Valid model path. | Wrong path breaks forward/backward gaze classification. |
| `PYRAMID_DEPTH_GAZE_MODEL_RUNTIME` | `TensorRT` | `backend/apps/pipeline/config.py` | Per-model runtime override for depth gaze inference. | `auto`, `onnx`, `openvino`, `tensorrt`. | Overrides backend selection for this model family. |
| `PYRAMID_VERTICAL_GAZE_MODEL_PATH` | `up_down/weights/up_down.engine` | `backend/config/settings/base.py`, `backend/apps/pipeline/config.py` | Selects the vertical gaze model artifact. | Valid model path. | Wrong path breaks up/down gaze classification. |
| `PYRAMID_VERTICAL_GAZE_MODEL_RUNTIME` | `TensorRT` | `backend/apps/pipeline/config.py` | Per-model runtime override for vertical gaze inference. | `auto`, `onnx`, `openvino`, `tensorrt`. | Overrides backend selection for this model family. |
| `PYRAMID_TRACKING_MODEL_RUNTIME` | `TensorRT` | `backend/apps/pipeline/config.py` | Runtime override for the tracking model path where applicable. | `auto`, `onnx`, `openvino`, `tensorrt`. | Can change tracker hardware/runtime behavior. |
| `PYRAMID_TRACKING_ALGORITHM` | `bytetrack` | `backend/apps/pipeline/config.py` | Default tracker family. | Common values in this codebase: `bytetrack`, `botsort`. | Sets the default tracker when no user preference overrides it. |
| `PYRAMID_TRACKING_REID_ALGORITHM` | `botsort` | `backend/apps/pipeline/config.py` | Declares which tracker is considered ReID-capable. | Typically `botsort` or another configured tracker key. | Used when users enable tracking IDs and appearance matching. |
| `PYRAMID_TRACKING_LOW_COMPUTE_ALGORITHM` | `bytetrack` | `backend/apps/pipeline/config.py` | Defines the lower-cost tracker used when users disable tracking IDs. | Typical value `bytetrack`. | Reduces compute cost for ID-free overlays. |
| `PYRAMID_TRACKING_REID_ENABLED` | `true` | `backend/apps/pipeline/config.py` | Master flag for appearance-based re-identification. | Boolean-like values: `true/false`, `1/0`, `yes/no`. | If `false`, ReID is disabled even when a ReID-capable tracker is selected. |
| `PYRAMID_TRACKING_IOU_THRESHOLD` | `0.3` | `backend/apps/pipeline/config.py` | Threshold for frame-to-frame IoU association. | Float from `0.0` to `1.0`. | Lower values allow looser matching; higher values are stricter and can fragment tracks. |
| `PYRAMID_TRACKING_MAX_CENTER_DISTANCE` | `160.0` | `backend/apps/pipeline/config.py` | Maximum centroid distance for association. | Positive float, validated up to `4096.0`. | Larger values permit more aggressive reassociation across movement. |
| `PYRAMID_TRACKING_REID_SIMILARITY_THRESHOLD` | `0.85` | `backend/apps/pipeline/config.py` | Similarity cutoff for ReID matching. | Float from `0.0` to `1.0`. | Higher values reduce false matches but can miss legitimate re-links. |
| `PYRAMID_EMBEDDING_REDIS_TTL_SECONDS` | `2592000` | `backend/apps/pipeline/config.py` | TTL for student embedding entries stored in Redis. | Integer from `60` to `31536000`. | Higher TTL keeps ReID memory longer but increases stale-data lifetime. |
| `PYRAMID_RTSP_OVER_TCP` | `true` | `backend/apps/cameras/services.py`, `backend/apps/pipeline/config.py` | Forces RTSP transport over TCP on unstable networks. | Boolean-like values. | Improves reliability on lossy networks, sometimes at latency cost. |
| `PYRAMID_RTSP_TRANSPORT` | `tcp` | `backend/apps/cameras/services.py`, `backend/apps/pipeline/config.py` | Declares the preferred RTSP transport mode. | `tcp`, `udp`, `auto`. | Affects how RTSP URLs are normalized before going to go2rtc/pipeline code. |
| `PYRAMID_INFERENCE_AUDIT_ENABLED` | `true` | `backend/apps/pipeline/config.py`, `backend/apps/video_analysis/tasks.py` | Persists detailed inference audit JSON for offline analysis jobs. | Boolean-like values. | Enables/disables audit artifact generation under the video storage tree. |
| `PYRAMID_INFERENCE_AUDIT_LIVE_ENABLED` | `true` | `backend/apps/pipeline/config.py` | Persists audit events for live streaming flows. | Boolean-like values. | Controls live-stream audit persistence overhead. |
| `PYRAMID_DEVICE` | `cuda` | `backend/config/settings/base.py`, `backend/apps/pipeline/config.py` | Preferred device for non-production local adapter work. | `cpu`, `cuda`, `mps`. | Does not govern native Triton production execution. |
| `PYRAMID_OPENVINO_DEVICE` | `intel:gpu` | `backend/apps/pipeline/config.py` | Target device string for OpenVINO execution. | Strings such as `intel:gpu`, `intel:cpu`, `intel:npu`. | Changes OpenVINO deployment target. |
| `PYRAMID_PERSON_CONFIDENCE_THRESHOLD` | `0.5` | `backend/config/settings/base.py`, `backend/apps/pipeline/config.py` | Minimum confidence for person detection acceptance. | Float from `0.0` to `1.0`. | Higher values reduce false positives but can drop real detections. |
| `PYRAMID_BEHAVIOR_CONFIDENCE_THRESHOLD` | `0.5` | `backend/config/settings/base.py`, `backend/apps/pipeline/config.py` | Minimum confidence for posture/gaze/behavior acceptance. | Float from `0.0` to `1.0`. | Higher values reduce noisy labels but can suppress valid behavior events. |
| `PYRAMID_WORKER_COUNT` | `4` | `backend/config/settings/base.py`, `backend/apps/pipeline/config.py` | Parallel worker count for inference workloads. | Integer from `1` to `16` in config validation. | Higher values improve throughput until CPU/GPU saturation. |
| `PYRAMID_INFERENCE_TIMEOUT` | `30.0` | `backend/config/settings/base.py`, `backend/apps/pipeline/config.py` | Per-inference timeout in seconds for local model execution. | Float from `1.0` to `300.0`. | Prevents stuck inference calls from hanging the job indefinitely. |
| `PYRAMID_BENCHMARK_WARMUP_ITERATIONS` | `10` | `backend/config/settings/base.py`, `backend/apps/pipeline/config.py` | Warm-up cycle count for benchmark runs. | Integer from `0` to `100`. | More warm-up gives more stable benchmark data at higher startup cost. |
| `PYRAMID_BENCHMARK_SAMPLE_COUNT` | `100` | `backend/config/settings/base.py`, `backend/apps/pipeline/config.py` | Sample count for benchmark measurement. | Integer from `10` to `1000`. | Higher values improve benchmark stability but lengthen test runs. |

### Root `.env` Operational Extensions

These variables are present in the active root `.env` and are supported by the code, but they are not currently pre-seeded in the checked-in root `.env.example`.

| Variable | Example / Default | Used by | Why it exists | Acceptable values | Role / operational effect |
| --- | --- | --- | --- | --- | --- |
| `PYRAMID_PRESERVE_SOURCE_QUALITY` | `true` | `backend/apps/pipeline/config.py`, `backend/apps/pipeline/multi_model.py`, `backend/apps/tracking/video_exporter.py` | Keeps annotated output closer to source fidelity. | Boolean-like values. | When `true`, output encoding defaults stay closer to source quality; when `false`, exports can be more compressed. |
| `PYRAMID_INFERENCE_IMGSZ` | `0` | `backend/apps/pipeline/config.py`, `backend/apps/pipeline/detector.py`, `backend/apps/pipeline/multi_model.py`, `backend/apps/tracking/tracker.py` | Global inference image-size override. | Integer from `0` to `4096`. `0` means use backend defaults. | Larger sizes can improve accuracy but increase latency and memory. |
| `PYRAMID_STREAM_INFERENCE_IMGSZ` | `0` | `backend/apps/pipeline/config.py` and stream inference call sites | Stream-specific image-size override. | Integer from `0` to `4096`. `0` falls back to `PYRAMID_INFERENCE_IMGSZ` or backend defaults. | Lets live streaming use a different inference size from offline jobs. |
| `PYRAMID_OUTPUT_MAX_WIDTH` | `0` | `backend/apps/pipeline/config.py`, `backend/apps/pipeline/multi_model.py` | Caps annotated output width. | Integer from `0` to `8192`. `0` keeps source width. | Useful to downscale exports and reduce file size. |
| `PYRAMID_OUTPUT_MAX_HEIGHT` | `0` | `backend/apps/pipeline/config.py`, `backend/apps/pipeline/multi_model.py` | Caps annotated output height. | Integer from `0` to `8192`. `0` keeps source height. | Useful to downscale exports and reduce file size. |
| `PYRAMID_OUTPUT_VIDEO_CRF` | `18` | `backend/apps/pipeline/config.py`, `backend/apps/pipeline/multi_model.py`, `backend/apps/tracking/video_exporter.py` | Sets H.264 CRF for annotated outputs. | Integer from `0` to `51`. Lower means better quality and larger files. | Directly controls output video quality/size tradeoff. |
| `PYRAMID_OUTPUT_VIDEO_PRESET` | `medium` | `backend/apps/pipeline/config.py`, `backend/apps/pipeline/multi_model.py`, `backend/apps/tracking/video_exporter.py` | Sets FFmpeg H.264 preset. | Typical FFmpeg presets such as `ultrafast`, `fast`, `medium`, `slow`. | Faster presets reduce CPU cost but usually produce larger files. |
| `PYRAMID_OUTPUT_VIDEO_BITRATE` | empty string or values like `6M` | `backend/apps/pipeline/config.py`, `backend/apps/pipeline/multi_model.py`, `backend/apps/tracking/video_exporter.py` | Optional explicit bitrate target. | Empty string or FFmpeg bitrate notation such as `4M`, `6000k`. | If set, it steers output bitrate instead of CRF-only encoding. |
| `PYRAMID_OUTPUT_JPEG_QUALITY` | `95` | `backend/apps/pipeline/config.py`, `backend/apps/tracking/video_exporter.py` | Controls JPEG quality for intermediate images. | Integer from `1` to `100`. | Higher values improve image fidelity and increase storage size. |
| `TRITON_ENABLED` | `0` in base defaults, `1` in development defaults | `backend/config/settings/base.py`, `backend/config/settings/development.py`, inference client factory, runtime policy, video analysis tasks | Transitional switch for local/dev wiring; production MUST enable Triton authority. | Boolean-like values. | Production startup MUST reject disabled Triton inference. |
| `TRITON_FORCE_DOCKER` | `0` in base defaults, `1` in development defaults | `backend/config/settings/base.py`, `backend/config/settings/development.py`, runtime policy, multi-model loader | Development Docker-specific behavior only. | Boolean-like values. | MUST NOT be a native Linux production dependency. |
| `TRITON_HYBRID_LOCAL_PERSISTENCE` | `1` | `backend/config/settings/base.py`, runtime policy, multi-model loader | Allows local model persistence while Triton routing is enabled. | Boolean-like values. | If `false` with forced Triton, local model loading is intentionally disabled. |
| `TRITON_REQUIRED_OFFLINE` | `0` in base defaults, `1` in development defaults | `backend/config/settings/base.py`, `backend/config/settings/development.py`, runtime policy, offline analysis tasks | Requires Triton for offline jobs. | Boolean-like values. | MUST be true for production offline activation. |
| `TRITON_REQUIRED_LIVE` | `0` in base defaults, `1` in development defaults | `backend/config/settings/base.py`, `backend/config/settings/development.py`, runtime policy | Requires Triton for live streaming analysis. | Boolean-like values. | MUST be true for production live activation. |
| `TRITON_URL` | `http://localhost:8000` | `backend/config/settings/base.py`, health views, inference client factory, video analysis tasks, core configuration | Base HTTP endpoint for Triton inference and health checks. | Reachable Triton HTTP URL. | Wrong value breaks Triton health checks and inference calls. |
| `TRITON_TIMEOUT_MS` | `1500` in base defaults | `backend/config/settings/base.py`, inference client factory, video analysis tasks, core configuration | Default per-request Triton timeout. | Positive integer milliseconds. | Lower values fail fast; overly low values can create false timeouts. |
| `TRITON_MAX_TIMEOUT_MS` | `5000` in base defaults | `backend/config/settings/base.py`, inference client factory, video analysis tasks, core configuration | Upper bound for requested Triton timeout. | Positive integer milliseconds greater than or equal to the normal timeout. | Prevents callers from requesting unbounded Triton waits. |
| `TRITON_RETRY_ATTEMPTS` | `2` | `backend/config/settings/base.py` | Retry count for Triton request logic. | Non-negative integer. | Higher values improve resilience but increase worst-case latency. |
| `TRITON_RETRY_TIMEOUT_SCALE` | `2.0` | `backend/config/settings/base.py` | Backoff scale applied between Triton retries. | Positive float. | Higher values back off more aggressively between attempts. |
| `TRITON_OFFLINE_FRAME_STRIDE` | `10` | `backend/config/settings/base.py`, `backend/apps/video_analysis/tasks.py` | Thins frame submission cadence for offline Triton processing. | Positive integer. | Higher values reduce load and accuracy granularity; lower values process more frames. |
| `TRITON_CROP_BEHAVIOR_INPUT_SIZE` | `640` | `backend/config/settings/base.py`, `backend/apps/video_analysis/tasks.py` | Selects the square crop-frame input size used only for posture/gaze behavior crops. | Integer >= 32 divisible by 32. Values below 640 require matching TensorRT plans and Triton configs built by `tools/prod/prod_enable_roi_crop_behavior.sh`. | Reduces crop-frame behavior pixels and Python/gRPC tensor traffic when paired with matching engines; mismatched configs make Triton reject requests. |
| `TRITON_BEHAVIOR_ENSEMBLE` | `0` in base defaults, `1` in optimized production helper | `backend/config/settings/base.py`, `backend/apps/video_analysis/tasks.py`, `backend/apps/pipeline/services/triton_client.py`, `tools/prod/prod_start_triton.sh` | Routes crop-frame behavior fan-out through `behavior_ensemble` so one crop tensor upload feeds posture plus the three gaze models. | Boolean-like values. Requires `behavior_ensemble` in `TRITON_LOAD_MODEL` and a valid Triton ensemble config. | When enabled, reduces behavior request fragmentation; rollback is setting this to `0`, which restores the four standalone model calls. |
| `TRITON_BEHAVIOR_TOP_K_ENABLED` | `0` | `backend/config/settings/base.py`, `backend/scripts/build_tensorrt_engines.py`, `tools/prod/prod_enable_behavior_topk.sh` | Enables the accepted-with-caveat Cycle 9b B.2.c route that packs only the top task-valid behavior anchors inside Triton before returning outputs to Python. | Boolean-like values. Requires `GAZE_HORIZONTAL_HEAD_VARIANT=slice`, `behavior_ensemble_gaze_slice_topk`, and four FP32 Top-K adapter plans loaded in Triton. | Production benchmark `cycle9b-topk-crop-frame-20260602T041900` passed decoded parity and improved Step 2 wall/RTT; average GPU utilization did not improve. Rollback is `0` plus `MODEL_ROUTE_BEHAVIOR_ALL_MODEL_NAME=behavior_ensemble_gaze_slice`. |
| `TRITON_BEHAVIOR_TOP_K_VALUE` | `100` | `backend/config/settings/base.py`, `backend/scripts/build_tensorrt_engines.py`, `tools/prod/prod_enable_behavior_topk.sh`, `tools/prod/prod_behavior_topk_parity.py` | Controls the number of behavior anchors kept per crop/model by the Top-K adapters. | Positive integer; must match `TRITON_YOLO_MAX_DECODE_CANDIDATES` for parity. | Lower values reduce output traffic but risk dropping detections; do not change without decoded parity and full production benchmark. |
| `GAZE_HORIZONTAL_HEAD_VARIANT` | `coco80`; accepted production profile sets `slice`; `gaze2` is rejected | `backend/config/settings/base.py`, `backend/apps/video_analysis/tasks.py`, `tools/prod/prod_start_triton.sh`, `tools/prod/prod_enable_gaze_horizontal_gaze2.sh`, `tools/prod/prod_enable_gaze_horizontal_slice.sh` | Selects the horizontal-gaze output contract. `coco80` expects `[84,2100]`; `gaze2` expects `[6,2100]` from the rejected standalone `gaze_horizontal_gaze2_model`; `slice` expects `[6,2100]` from exact server-side slicing after the legacy model. | `coco80`, `gaze2`, or `slice`. Non-`coco80` values require matching model route overrides and Triton configs. | `slice` is the accepted exact-output-fusion route from production benchmark `cycle9b-exactslice-crop-frame-20260601T233211`; compact class IDs remap back to legacy DB IDs `4/5`; rollback is `coco80` via `prod_enable_parallel_flow.sh --profile per-frame-signals`. |

### Frontend `frontend/.env.example` Variables

These are the frontend-specific names currently documented by the Vite app.

| Variable | Example / Default | Used by | Why it exists | Acceptable values | Role / operational effect |
| --- | --- | --- | --- | --- | --- |
| `VITE_API_BASE_URL` | `http://localhost:8000/api/v1` | `frontend/src/api/client.ts` | Sets the REST base URL for Axios. | Absolute URL or same-origin path. | Wrong value breaks REST requests in the browser. |
| `VITE_WS_BASE_URL` | `ws://localhost:8000` | `frontend/src/hooks/useWebSocket.ts` | Sets the WebSocket host base; the hook appends route paths such as `/ws/monitor`. | `ws://...` or `wss://...`, no trailing slash preferred. | Wrong value breaks monitor socket connections. |
| `VITE_GO2RTC_URL` | `http://localhost:1984` | `frontend/src/hooks/useWhepClient.ts`, `frontend/vite.config.ts` `/whep` proxy design context | Documents the go2rtc base endpoint used by the WebRTC/WHEP streaming flow. | HTTP URL for go2rtc. | Keep this aligned with the deployed go2rtc instance and the Vite/nginx proxy target expectations. |

Frontend Vite proxy behavior from `frontend/vite.config.ts`:

- `/api` proxies to `VITE_API_PROXY_TARGET` or `http://localhost:8000`
- `/ws` proxies to the matching WebSocket target derived from the API proxy target
- `/whep` proxies to `http://localhost:1984` and rewrites requests into the go2rtc `/api/webrtc?src=camera_{id}` format

### ONVIF Endpoints

ONVIF integration is available through camera API endpoints:

- `POST /api/v1/cameras/onvif/resolve/` with JSON body `{"connection_url":"http://<host>/onvif/device_service"}` resolves ONVIF to RTSP URL.
- `POST /api/v1/cameras/onvif/test/` validates resolved RTSP using the active stream provider (default `gst_mediamtx` path).
- `POST /api/v1/cameras/{camera_id}/onvif/sync/` resolves RTSP for a stored ONVIF camera.

Required env vars for ONVIF resolution:

- `ONVIF_USERNAME`
- `ONVIF_PASSWORD`
- `ONVIF_WSDL_DIR`

## Development Commands

### Infrastructure

```powershell
docker compose -f docker-compose.dev.yml up -d
docker compose -f docker-compose.dev.yml down
docker compose -f docker-compose.dev.yml logs -f postgres
docker compose -f docker-compose.dev.yml --profile triton up -d triton
```

### Backend

```powershell
cd backend
uv sync --dev
uv run python manage.py migrate
uv run python manage.py createsuperuser
uv run python manage.py runserver
uv run celery -A config worker -l info
uv run celery -A config beat -l info
uv run pytest
```

### Frontend

```powershell
cd frontend
npm install
npm run dev
npm run lint
npm run type-check
npm run test
npm run test:coverage
npm run test:e2e
npm run build
```

### Documentation and CI Gates

```powershell
python scripts\ci\verify_docs_diagrams.py
python scripts\ci\verify_module_boundaries.py
python scripts\ci\verify_release_gate.py --phase blocking --unit-result success --integration-result success --contract-result success
```

### Triton Helper Scripts

```powershell
.\scripts\pull-triton-image.ps1
.\scripts\pull-triton-image.ps1 -BuildProjectImage
.\scripts\export-tensorrt-models.ps1
.\scripts\sync-triton-tensorrt-repository.ps1
```

## Quick Setup

Use [quickstart.md](quickstart.md) for the full step-by-step workflow. The short version is:

1. Copy `.env.example` to `.env`
2. Start `docker-compose.dev.yml`
3. Sync backend dependencies with `uv`
4. Run backend migrations and services
5. Install frontend dependencies and start Vite
6. Optionally start Triton using the compose profile
7. Run verification commands

## Beta Sign-off

For consolidated release-readiness status, open [docs/release-beta-signoff-checklist.md](docs/release-beta-signoff-checklist.md).

Current consolidated state (as of 2026-05-21): `CONDITIONAL GO` with explicit remaining risks and non-green gates documented in that checklist.

## Known Issues And Fixes

### 1. Celery beat schedule corruption on Windows

Symptoms:

- `celery beat` fails with schedule database errors

Fix:

```powershell
cd backend
Remove-Item -Force celerybeat-schedule -ErrorAction SilentlyContinue
uv run celery -A config beat -l info
```

### 2. Frontend cannot reach backend in development

Symptoms:

- API calls fail from `localhost:5173`
- WebSocket connection fails

Checks:

- Backend is running on `http://localhost:8000`
- `frontend/vite.config.ts` proxy is active
- `VITE_API_BASE_URL` and `VITE_WS_BASE_URL` are consistent with your chosen setup

### 3. go2rtc WHEP feed does not open

Checks:

- `docker compose -f docker-compose.dev.yml up -d go2rtc`
- `GO2RTC_API_URL` and `GO2RTC_WHEP_URL` are correct
- Camera source registration exists in the backend

### 4. Triton profile does not start

Checks:

- Use `docker compose -f docker-compose.dev.yml --profile triton up -d triton`
- NVIDIA GPU and Docker GPU runtime are available
- Model repository path `backend/models/triton_repository` exists

### 5. Missing encrypted camera URL key

Symptoms:

- Camera persistence or decrypt/encrypt operations fail

Fix:

```powershell
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

Put the generated value into `FIELD_ENCRYPTION_KEY` in `.env`.

### 6. Documentation drift

Run:

```powershell
python scripts\ci\verify_docs_diagrams.py
python scripts\ci\verify_module_boundaries.py
```

## Recommended Validation Sequence

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart LR
    A[Write Test<br>Red] --> B[Implement<br>Green]
    B --> C[Refactor]
    C --> D[Lint + Format]
    D --> E[Run Full Suite]
    E --> F{100%<br>Coverage?}
    F -->|Yes| G[Commit<br>Conventional]
    F -->|No| A
    G --> H[Push + PR]
```

The diagram shows the recommended development and validation loop used in the project workflow.

## Testing Notes

- Backend `pytest.ini` uses `config.settings.test`
- Frontend coverage thresholds in `vite.config.ts` are set to `80` for lines, functions, branches, and statements
- Playwright scripts are available through `npm run test:e2e` and `npm run test:e2e:ui`

## Next References

- [Quickstart](quickstart.md)
- [Backend README](backend/README.md)
- [Frontend README](frontend/README.md)
