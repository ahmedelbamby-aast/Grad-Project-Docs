# forensic-systems-audit-orchestrator Usage (Scientific Edition)

This guide provides copy-paste templates to run the skill as a scientific paper writer + forensic auditor.

## 1) Preflight (Main Agent)

- Close stale agents.
- Ensure `paper/` exists.
- Ensure `.gitignore` contains `paper/`.
- Ensure files exist:
  - `paper/01_architecture_deployment.md`
  - `paper/02_ingestion_orchestration.md`
  - `paper/03_detection_tracking.md`
  - `paper/04_pose_temporal_artifacts.md`
  - `paper/05_data_api_frontend.md`
  - `paper/06_observability_testing_science.md`
  - `paper/coverage_matrix.md`
- Ensure `paper/latex/` exists.

## 2) Base Constraints Block (prepend to every agent message)

```text
You are an evidence-first forensic scientific audit agent.
Zero hallucination.
Every substantive claim must include repository evidence references.
Allowed labels: IMPLEMENTED, PARTIAL, MISSING, INCONSISTENT, NOT EVIDENCED.
Do not infer behavior from comments alone.
Do not fabricate equations, figures, metrics, or statistics.
Use absolute repo paths and line references when possible.
Only write to your assigned output file.
```

## 3) Required Report Template (A1..A6)

```text
1. Executive Summary
2. Audit Scope
3. Implemented (Evidence-Backed)
4. Partially Implemented (Evidence-Backed)
5. Missing (Evidence-Backed)
6. Architecturally Incorrect / Inconsistent
7. Scientifically Weak / Unvalidated
8. Bottlenecks
9. Production Readiness Assessment
10. Equations Actually Used
11. Figures/Images Candidates from Repo
12. Contradictions (Code vs Docs vs Tests)
13. Open Questions / Unknowns
14. Subsystem Coverage Checklist
```

## 4) Spawn Templates (A1..A6)

Use `agent_type: worker`, `fork_context: true`, one message per agent.

### A1 — Architecture + Deployment

```text
[BASE CONSTRAINTS BLOCK]

Write lock: paper/01_architecture_deployment.md
You may edit only this file.

Scope:
- High-level architecture and subsystem boundaries
- Runtime modes and deployment topology
- Env configs, scripts, infra, Triton endpoint/profile policy
- Docs-vs-code consistency for architecture/deployment

Output: populate paper/01_architecture_deployment.md using the required 14-section template.
```

### A2 — Ingestion + Queueing + Orchestration

```text
[BASE CONSTRAINTS BLOCK]

Write lock: paper/02_ingestion_orchestration.md
You may edit only this file.

Scope:
- Upload/live ingestion, RTSP handling/reconnect
- Buffering, frame lifecycle, drops
- Celery routing/topology/retries/backpressure/starvation
- Runtime orchestration, timeout/failure behavior

Output: populate paper/02_ingestion_orchestration.md using the required 14-section template.
```

### A3 — Detection + Tracking

```text
[BASE CONSTRAINTS BLOCK]

Write lock: paper/03_detection_tracking.md
You may edit only this file.

Scope:
- Detection architecture, batching/cadence/reuse/interpolation
- Tracking lifecycle and ID persistence
- ReID, occlusion/reentry behavior
- Multi-camera assumptions and identity continuity risks

Output: populate paper/03_detection_tracking.md using the required 14-section template.
```

### A4 — Pose + Temporal + Artifacts

```text
[BASE CONSTRAINTS BLOCK]

Write lock: paper/04_pose_temporal_artifacts.md
You may edit only this file.

Scope:
- RTMPose integration
- ROI/crop mapping and Triton/ONNX/TensorRT execution path
- Pose confidence/smoothing/jitter/visibility logic
- Artifact generation/consumption
- Temporal memory reality check

Output: populate paper/04_pose_temporal_artifacts.md using the required 14-section template.
```

### A5 — Data + APIs + WS + Frontend

```text
[BASE CONSTRAINTS BLOCK]

Write lock: paper/05_data_api_frontend.md
You may edit only this file.

Scope:
- DB schema and JSON artifact strategy
- Persistence model and ML-readiness
- API contracts and websocket/event contracts
- Frontend overlays/timelines/debug telemetry UX

Output: populate paper/05_data_api_frontend.md using the required 14-section template.
```

### A6 — Observability + Testing + Scientific Rigor

```text
[BASE CONSTRAINTS BLOCK]

Write lock: paper/06_observability_testing_science.md
You may edit only this file.

Scope:
- Logging/metrics/telemetry integrity
- Benchmark validity and regression detectability
- Unit/integration/system/perf/resilience test realism
- Scientific soundness of claims/evaluation

Output: populate paper/06_observability_testing_science.md using the required 14-section template.
```

## 5) Merge Protocol (Main Agent)

After A1..A6 complete:

1. Read all six report files.
2. Build `paper/coverage_matrix.md` with:
   - subsystem/module
   - covered_by_report
   - status (`covered`|`partial`|`missing`)
   - evidence_refs
3. For each `partial`/`missing`, run follow-up pass only on responsible report.
4. Repeat until coverage is complete for currently implemented modules.

Pre-synthesis QA gates:

- All 14 sections exist in each report and in order.
- Each major claim has evidence refs.
- Each equation includes symbol definitions + mapping refs.
- Each figure candidate has concrete repo path + relevance.

## 6) Synthesis Agent Template (Scientific Paper)

```text
You are the synthesis agent for a scientific systems-audit paper.
Zero hallucination.
All major claims must be evidence-traceable.
Do not invent equations, figures, or statistics.
Only use repository-existing figure assets.

Write lock: paper/latex/** only.

Inputs:
- paper/01_architecture_deployment.md
- paper/02_ingestion_orchestration.md
- paper/03_detection_tracking.md
- paper/04_pose_temporal_artifacts.md
- paper/05_data_api_frontend.md
- paper/06_observability_testing_science.md
- paper/coverage_matrix.md

Tasks:
1) Create paper/latex/main.tex
2) Create paper/latex/sections/*.tex from the six reports
3) Enforce scientific structure:
   - Abstract
   - Introduction
   - System Context and Problem Statement
   - Methods (Forensic Audit Methodology)
   - Results
   - Discussion
   - Threats to Validity
   - Reproducibility and Artifact Availability
   - Related Work / Standards Alignment
   - Conclusion
   - References
4) Create paper/latex/figures_manifest.md
5) Create paper/latex/equations_manifest.md
6) Create paper/latex/traceability_matrix.md
7) Create paper/latex/reproducibility_manifest.md
8) Create paper/latex/limitations_and_validity.md
9) Resolve cross-section contradictions where possible; otherwise mark INCONSISTENT with evidence references.
10) Iterate until internally consistent and complete.
```

## 7) Statistical Reporting Rules

- Use only observed numbers from repo artifacts or in-session execution outputs.
- Do not claim inferential statistics unless supported by repeated-run data.
- If repeated-run evidence is missing, explicitly state descriptive-only evidence.

## 8) Final Quality Gate Checklist

Pass only if all are true:

- No hallucinated components/claims
- Every major claim has evidence refs
- Coverage matrix shows complete implemented-system coverage
- LaTeX reflects implemented system only
- Equation manifest maps each equation to real code/dependency usage
- Figures manifest maps each figure to real repo file
- Threats to validity documented
- Reproducibility manifest complete

## 9) Final Summary Format (Main Agent)

```text
- Agents launched/completed: ...
- Files generated/updated: ...
- Coverage status: ...
- Quality gates: pass/fail (+ reasons)
- Remaining unknowns: ...
- Validity limitations: ...
```
