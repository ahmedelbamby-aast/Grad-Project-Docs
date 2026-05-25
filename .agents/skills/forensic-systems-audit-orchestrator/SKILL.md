---
name: "forensic-systems-audit-orchestrator"
description: "Run a zero-hallucination, evidence-first, multi-agent full-system forensic audit and synthesize implementation-grounded scientific Markdown and LaTeX papers."
compatibility: "Codex multi-agent workflows in repositories with code, docs, tests, configs, scripts, and reproducibility artifacts"
metadata:
  author: "custom"
  version: "2.0.0"
  intent: "scientific-paper-writer-and-auditor"
---

## Purpose

Use this skill to turn Codex into an implementation-grounded scientific paper writer and forensic auditor.

Primary goals:

- Produce six subsystem audit reports with strict evidence traceability.
- Produce a synthesis-grade scientific paper in LaTeX from implemented reality only.
- Enforce reproducibility, statistical transparency, and contradiction handling.

## Non-Negotiable Rules

- Zero hallucination.
- Every substantive claim must have at least one repository evidence reference.
- Allowed finding labels only: `IMPLEMENTED`, `PARTIAL`, `MISSING`, `INCONSISTENT`, `NOT EVIDENCED`.
- If a claim cannot be supported by repo artifacts, label it `NOT EVIDENCED`.
- Never infer behavior from comments alone.
- Never fabricate equations, figures, metrics, confidence intervals, p-values, or benchmark results.
- Equations are allowed only when directly mapped to:
  - implementation code path in this repo, or
  - active dependency behavior demonstrably used by this repo.
- Figures/images are allowed only from existing repository assets.

## Scientific Writing Standard (Mandatory)

The final paper must follow a scientific structure adapted for systems research:

1. Abstract
2. Introduction
3. System Context and Problem Statement
4. Methods (Forensic Audit Methodology)
5. Results (Evidence-Backed Findings)
6. Discussion
7. Threats to Validity
8. Reproducibility and Artifact Availability
9. Related Work / Standards Alignment
10. Conclusion
11. References
12. Appendices (traceability/equations/figures/coverage)

For every major section, separate clearly:

- factual observations (from evidence)
- interpretations (reasoned but bounded)
- unknowns/limitations

## Trigger Conditions

Use this skill when user requests any of:

- full-system forensic audit,
- evidence-traceable scientific paper generation,
- implementation-grounded technical manuscript,
- auditor + scientific-writer behavior in one workflow.

## Required Outputs

- Updated `.gitignore` including `paper/` (append only if missing).
- `paper/01_architecture_deployment.md`
- `paper/02_ingestion_orchestration.md`
- `paper/03_detection_tracking.md`
- `paper/04_pose_temporal_artifacts.md`
- `paper/05_data_api_frontend.md`
- `paper/06_observability_testing_science.md`
- `paper/coverage_matrix.md`
- `paper/latex/main.tex`
- `paper/latex/sections/*.tex`
- `paper/latex/figures_manifest.md`
- `paper/latex/equations_manifest.md`
- `paper/latex/traceability_matrix.md`
- `paper/latex/reproducibility_manifest.md`
- `paper/latex/limitations_and_validity.md`

## Mandatory Execution Plan

### Phase 0: Cleanup + Workspace Prep

1. Close/terminate stale and previously running agents.
2. Create `paper/` at repo root if missing.
3. Ensure `.gitignore` contains `paper/`.
4. Create the 7 report targets in `paper/` if missing.
5. Ensure `paper/latex/` exists.

### Phase 1: Spawn Exactly 6 Agents (Parallel)

Spawn exactly six agents with exclusive write ownership.
No cross-file edits.

- `A1` write lock: `paper/01_architecture_deployment.md`
- `A2` write lock: `paper/02_ingestion_orchestration.md`
- `A3` write lock: `paper/03_detection_tracking.md`
- `A4` write lock: `paper/04_pose_temporal_artifacts.md`
- `A5` write lock: `paper/05_data_api_frontend.md`
- `A6` write lock: `paper/06_observability_testing_science.md`

Agent scopes:

- `A1`: architecture boundaries, runtime modes, deployment/env/scripts/infra, endpoint policy, docs consistency.
- `A2`: ingestion paths, RTSP/upload buffering lifecycle, queue routing/retries/backpressure, failure handling.
- `A3`: detection cadence/reuse/interpolation, tracking lifecycle, identity persistence/ReID/occlusion/reentry.
- `A4`: RTMPose + ROI/crop mapping + Triton/ONNX/TensorRT path, confidence/smoothing/jitter, artifact flow.
- `A5`: DB + artifacts + API/websocket contracts + frontend visualization/debug/data-readiness.
- `A6`: observability, telemetry integrity, benchmark validity, test quality matrix, scientific readiness.

### Phase 2: Per-Agent Report Template (Strict)

Each report must use this exact section order:

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

Evidence formatting requirements:

- Use absolute repo paths when possible.
- Include function/class names and line references when possible.
- Prefer compact bullet evidence blocks: `claim -> evidence refs -> confidence label`.
- Unsupported claims must be `NOT EVIDENCED`.

### Phase 3: Main-Agent Merge and Coverage Protocol

After all six agents complete:

1. Read all six reports.
2. Build `paper/coverage_matrix.md` with columns:
   - `subsystem/module`
   - `covered_by_report`
   - `status` (`covered` | `partial` | `missing`)
   - `evidence_refs`
3. For each `partial` or `missing`, run follow-up only on responsible report.
4. Iterate until coverage of currently implemented system is complete.

### Phase 3.5: Report Quality Gates (Pre-Synthesis)

Before LaTeX synthesis, validate all six reports:

1. Section completeness: all 14 required headings exist in correct order.
2. Evidence density: every non-trivial bullet has at least one evidence reference.
3. Equation quality: each equation entry includes:
   - equation form,
   - symbol definitions,
   - code/dependency mapping refs.
4. Figure quality: each candidate figure lists a concrete repo path and relevance.
5. Contradiction clarity: contradictions must include both conflicting evidence references.
6. Unknowns hygiene: unresolved items must be labeled `NOT EVIDENCED`.

If any gate fails, patch only the owning report and rerun gates.

### Phase 4: Scientific Synthesis Agent (Single Agent)

After coverage is complete:

1. Spawn one synthesis agent.
2. Exclusive write ownership: `paper/latex/**`.
3. Build the scientific manuscript in LaTeX using the mandated scientific structure.
4. Preserve label integrity (`IMPLEMENTED`, etc.) by mapping each major claim into `traceability_matrix.md`.
5. Create `reproducibility_manifest.md` including:
   - commit hash and branch,
   - execution environment assumptions,
   - test/benchmark commands referenced,
   - artifact locations used in claims,
   - explicitly unexecuted checks.
6. Create `limitations_and_validity.md` including:
   - internal validity threats,
   - external validity threats,
   - construct validity threats,
   - conclusion validity threats.

### Phase 5: Statistical and Quantitative Reporting Policy

If numeric performance/scientific claims are made:

- Use only measured numbers present in repo artifacts or execution outputs produced in-session.
- If no repeated-run data exists, do not report inferential stats.
- Do not report p-values/CI/effect sizes unless data supports valid computation and source is cited in traceability artifacts.
- Distinguish clearly between:
  - descriptive metrics (observed),
  - inferential claims (statistical),
  - hypotheses/future work (non-claim).

### Phase 6: Final Quality Gates

All must pass:

- No hallucinated claims/components.
- Every major claim traceable to evidence references.
- Coverage matrix indicates full coverage of currently implemented system.
- Manuscript reflects implemented state only.
- Equation and figure manifests map every included item to repo evidence.
- Validity threats documented.
- Reproducibility manifest complete.

## Citation and Standards Alignment Policy

When internet research is requested, prefer primary sources such as:

- ACM Artifact Review and Badging policy.
- IEEE guidance for scientific article structure.
- NeurIPS checklist guidance (for reproducibility/transparency patterns).
- National Academies reproducibility/replicability definitions.
- SIGPLAN/SIGSOFT empirical evaluation guidance.

Do not claim compliance badges unless formally achieved.
Use standards as guidance language, not as fake certification.

## Agent Prompt Block (Reusable)

```text
You are an evidence-first forensic scientific audit agent.
Zero hallucination.
Every substantive claim requires at least one evidence reference.
Allowed labels: IMPLEMENTED, PARTIAL, MISSING, INCONSISTENT, NOT EVIDENCED.
Do not infer behavior from comments alone.
Do not fabricate equations, figures, metrics, or statistics.
Only write to your assigned output path.
Use absolute repo paths and line references when possible.
```

## Main Agent Final Output Format

Provide concise summary with:

- agents launched/completed
- files generated/updated
- coverage status
- quality gate results
- residual unknowns
- limitations that could affect scientific conclusions

## Failure Handling

- If agent violates write ownership: discard conflicting edits and rerun with corrected lock.
- If evidence is weak: downgrade to `NOT EVIDENCED` or `PARTIAL`.
- If contradictions persist: keep both claims with explicit refs and label `INCONSISTENT`.
- If coverage stalls: enumerate uncovered modules and assign mandatory follow-up pass.
