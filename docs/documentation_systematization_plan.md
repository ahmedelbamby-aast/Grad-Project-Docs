# Documentation Systematization Program (DSP) — Master Plan

**Last updated:** 2026-06-02

**Status:** **DSP CYCLE 0 — PLAN APPROVED, NO ENTITY DOCS GENERATED YET.**

> **Purpose.** This document is the binding multi-cycle plan for the
> Documentation Systematization Program (DSP). It exists so the project's
> docs can be:
> 1. structured and sorted by date and **canonical reading order**;
> 2. exhaustively cover every System / Module / Phase / Script / Code with
>    one entity doc per item, describing internal calls, intra-entity
>    flow, external calls, and dependencies;
> 3. visualised with **consistent Mermaid themes and colour codes**
>    so that the same diagram type renders identically across every doc;
> 4. preserved historically — every superseded diagram stays, new versions
>    are added in additional sections so other developers can see the
>    maturity arc;
> 5. produced **without hallucination** — every entity doc must cite the
>    real files / lines / symbols / commits / job IDs it describes;
> 6. governed by AGENTS.md + `.specify/memory/constitution.md` (Section 19,
>    added by this program).
>
> **Hallucinations are FORBIDDEN.** A violation of the Section 19 rules
> below is treated as a CI-blocking regression, not a style nit.

## 0. Why this plan exists (the governing problem)

Three concrete pain points triggered this program:

1. **Reading-order ambiguity.** Until 2026-06-02, the canonical narrative
   path through 86+ project docs had to be reconstructed from
   alphabetical listings + `docs/INDEX.md` + tribal knowledge. The
   reading-order section now lives in `README.md`, but no governance
   pinned it.
2. **Per-entity coverage gaps.** Many systems, modules, phases, and
   scripts have no single doc explaining their inputs, outputs,
   callers, callees, side effects, and connection graph. Developers
   read source code to recover that mental model — fragile and slow.
3. **Diagram drift + theme inconsistency.** Mermaid diagrams across
   the repo use different themes, colour palettes, and node-label
   conventions. Old diagrams sometimes get *deleted* when a flow
   changes, so the maturity arc is lost.

DSP fixes all three through 9 sequential cycles (Cycle 0 through Cycle 8)
each with its own acceptance gate.

## 1. Cycle map

| DSP Cycle | Title | Status | Acceptance gate |
|---|---|---|---|
| **0** | Plan + governance hardening | **CLOSED 2026-06-02** (commit `a32aa0d7`) | Three planning docs land + constitution Section 19 ratified + AGENTS.md hardened + README updated |
| **1** | Inventory + reading-order refresh for ALL project markdown | **CLOSED 2026-06-02** | 132/132 in-scope narrative docs carry a date header; verifier covers 132 docs (was 95); README reading order extended to Phase 10; `.pytest_cache_local/README.md` + 4 `scripts/ci/artifacts/triton_stabilization_summary.md` files explicitly excluded per § 9 |
| **2** | System-level entity docs (top-level systems) | **CLOSED 2026-06-02** (6 of 6 landed) | 6 system docs under `docs/entity/systems/`: `offline_inference_pipeline`, `live_streaming_pipeline`, `triton_inference_plane`, `telemetry_pipeline`, `camera_streaming_bridge`, `frontend_spa`. Each follows the per-entity template and uses the Cycle 0 theme contract. Anti-hallucination protocol (§ 19.6) caught 2 wrong paths/symbols during this cycle and self-corrected in the same commit that delivered the truth. |
| **3** | Module-level entity docs (Django apps + frontend packages) | **IN PROGRESS** (13 of ~18 landed: `apps.video_analysis`, `apps.pipeline`, `apps.tracking`, `apps.telemetry`, `apps.cameras`, `apps.detections`, `apps.anomalies`, `apps.sessions`, `apps.health`, `apps.behavior`, `apps.contracts`, `apps.accounts`, `apps.audit`) | One doc per module in `docs/entity/modules/`; covers internal call graph, cross-module deps, DB tables, env vars, public APIs |
| 4 | Phase / optimization-cycle docs | PLANNED | Per accepted/rejected cycle: a single canonical entity doc in `docs/entity/cycles/` that folds investigation + results + status |
| 5 | Script + service + helper entity docs | PLANNED | Per script under `tools/prod/**`, `scripts/**`, `backend/scripts/**`: entity doc listing inputs, outputs, env reads, files touched, callers, callees, side effects |
| 6 | Code-level entity docs (hot files only) | PLANNED | Per high-traffic Python module: function-level doc with call graph, exceptions thrown, ContextVar reads, async boundaries |
| 7 | API surface entity docs | PLANNED | Per REST endpoint / WS channel / WHEP route / Celery queue / Triton model route / Triton ensemble: connection-graph doc showing producer, consumer, payload contract, error path |
| 8 | CI gate that proves coverage + theme compliance | PLANNED | Extended verifier asserts (a) every entity in the inventory has a doc; (b) every Mermaid diagram declares the theme initializer; (c) no diagram was deleted (verified via git history) |

**Each cycle's acceptance gate must pass on the production branch
(`feature/phase7a-crop-frame-mode` or its successor) and must be backed
by real evidence — file paths, line numbers, commit SHAs, or job IDs.
A doc that cannot cite real evidence is a hallucination and is rejected.**

## 2. Entity scope (what counts as an "entity")

These are the five kinds of things every DSP entity doc covers:

| Kind | Examples | Cycle |
|---|---|---|
| **System** | offline inference pipeline, live streaming pipeline, telemetry pipeline, frontend SPA, go2rtc bridge | 2 |
| **Module** | `apps.video_analysis`, `apps.pipeline`, `apps.tracking`, `apps.telemetry`, `apps.cameras`, `frontend/src/api`, `frontend/src/components` | 3 |
| **Phase / Cycle** | "Cycle 6 pose chunking", "Cycle 9 ensemble", "Cycle 11 input 320→256", "Phase 7a crop-frame", LPM Phase 1 | 4 |
| **Script** | `tools/prod/prod_run_parallel_flow_benchmark.sh`, `backend/scripts/build_tensorrt_engines.py`, `scripts/ci/verify_doc_dates_and_reading_order.py` | 5 |
| **Code unit** | `apps.video_analysis.tasks._run_triton_frame_level_inference`, `apps.pipeline.services.triton_client.TritonClient`, `apps.pipeline.services.ensemble_validator.validate_behavior_ensemble_repository` | 6 |
| **API surface** | `POST /api/v1/video-analysis/jobs/`, `ws/video-analysis`, `whep/{camera_id}/`, Celery queue `offline_control`, Triton model route `behavior_all`, Triton ensemble `behavior_ensemble_gaze_slice_topk` | 7 |

The directory layout under `docs/entity/` is normative:

```
docs/
  entity/
    systems/<system-name>.md
    modules/<app-or-package-name>.md
    cycles/<cycle-id>.md
    scripts/<path-with-slashes-replaced-by-double-underscore>.md
    code/<dotted-path-of-symbol>.md
    api/<surface-kind>__<surface-name>.md
```

A file's existence under `docs/entity/` is the canonical answer to
"is this entity documented?" The Cycle 8 CI gate enforces it.

## 3. The non-negotiables (these become constitution Section 19)

These rules become binding on the entire repo when this commit lands.
See [`.specify/memory/constitution.md`](../.specify/memory/constitution.md)
Section 19 for the constitutional text + Anti-Regression Enforcement
Matrix rows.

### 3.1 Reading-order authority

The reading order declared in `README.md` § "Documentation Reading
Order" is **authoritative**. If a doc's `**Last updated:**` header
conflicts with its position in the reading order, **the reading
order wins**. Dates only reflect when a doc was last touched, not
where it belongs in the narrative.

### 3.2 Per-entity coverage

Every entity in § 2 above MUST have exactly one entity doc under
`docs/entity/` following [`docs/per_entity_doc_template.md`](per_entity_doc_template.md).
Removing the doc removes the entity from the supported surface; the
removal MUST be paired with the entity's deletion in the same commit.

### 3.3 Diagram theme contract

Every Mermaid diagram in a project-owned `.md` file MUST declare the
theme initializer specified in
[`docs/mermaid_theme_contract.md`](mermaid_theme_contract.md). The same
diagram **type** uses the same theme + palette across every doc.
A new diagram type MUST be added to the theme contract before it is
used in any doc.

### 3.4 Text-fitting rule (MANDATORY)

Mermaid node labels MUST fit inside their box. The contract enforces
this by:

- **Hard cap of 40 characters per logical line** in any node label.
- Multi-line labels use Mermaid's `<br/>` (or literal `\n`) breaks.
- Tables MUST NOT contain unbreakable strings wider than the column
  budget declared by the surrounding doc.
- The Cycle 8 CI gate fails on any node label exceeding the hard cap
  without a break.

### 3.5 Diagram preservation rule

If an existing diagram no longer reflects the current implementation,
**DO NOT DELETE IT.** Add a new H3 section titled
`## <feature> diagram — vYYYY-MM-DD (current)` immediately above the
old one (which gets renamed to `## <feature> diagram — vYYYY-MM-DD (historical)`).
Both diagrams stay so the maturity arc is visible.

### 3.6 Anti-hallucination protocol

Every entity doc MUST include a "Source-of-truth references" block
near the top listing **every** file path, symbol, commit SHA, or job
ID it cites. The Cycle 8 CI gate parses this block and verifies that
each referenced path / symbol exists in the working tree. A claim
that cannot be backed by a real reference is a hallucination and is
rejected at review.

Specifically forbidden:
- "Approximately N lines of code" without citing the file.
- "Around X ms" without citing the inference_audit JSON or bench summary.
- "The function does Y" without a `<file>:<line>` citation.
- Any quote from a doc that does not exist at the cited path.

### 3.7 Cycle discipline

Every entity doc in Cycles 2-7 follows the same 4-phase template:
investigate → write → cross-check against source → land. **No
mass-generation across multiple entities in a single commit** — each
entity ships in its own commit so review can verify the citations.
This is the strongest anti-hallucination guard.

### 3.8 Maximum work without cycle-overrun

A single commit MUST NOT advance two DSP cycles at once. If DSP Cycle 3
work is in progress, no DSP Cycle 4 commits may land until 3 closes.
This makes review tractable and keeps each cycle's evidence localised.

## 4. Cycle-0 deliverables (this commit)

This commit lands:

| # | File | Status | Notes |
|---|---|---|---|
| 1 | `docs/documentation_systematization_plan.md` (this file) | **NEW** | Master plan |
| 2 | `docs/mermaid_theme_contract.md` | **NEW** | Theme + palette + label rules |
| 3 | `docs/per_entity_doc_template.md` | **NEW** | The template every entity doc must use |
| 4 | `.specify/memory/constitution.md` | **AMENDED to v2.5.0** | New Section 19 + matrix rows in 14.25 |
| 5 | `AGENTS.md` | **AMENDED** | New top-level DSP section + anti-hallucination rules |
| 6 | `README.md` | **AMENDED** | Reading order Phase-8b row added; DSP planning entries |
| 7 | `scripts/ci/verify_doc_dates_and_reading_order.py` | **AMENDED** | New planning docs added to priority list |
| 8 | `.github/workflows/inference-parallelization.yml` | **AMENDED** | Path filter additions |

**This commit does NOT generate any entity doc.** DSP Cycle 1 starts in
the next commit.

## 5. Cycle-by-cycle acceptance criteria (binding)

### DSP Cycle 1 — Inventory + reading-order

- [ ] Every project-owned `.md` in the repo (excluding `.venv/`, `.git/`,
      third-party tool installs) carries a `**Last updated:**` header
      verified by `scripts/audit_doc_dates.py`.
- [ ] `README.md` reading order has zero broken links AND covers every
      narrative doc found by the inventory pass.
- [ ] `docs/INDEX.md` cross-references the README reading order at the top.

### DSP Cycle 2 — System-level docs

- [ ] One doc per system identified in `docs/ARCHITECTURE.md`.
- [ ] Each doc cites real file paths in its source-of-truth block.
- [ ] Each doc carries a system-context + container + deployment diagram
      using the theme contract.

### DSP Cycle 3 — Module-level docs

- [ ] One doc per installed Django app (see README "Backend Surface").
- [ ] One doc per frontend top-level package.
- [ ] Each doc lists files in it (via `git ls-files`), public
      symbols (via grep), DB tables, env vars, public APIs.

### DSP Cycle 4 — Phase / cycle docs

- [ ] One doc per accepted/rejected/staged cycle in
      `docs/cycle_9_and_10_improvements_todo.md` § Z.
- [ ] Each doc consolidates investigation + results + status using
      the template, citing the original investigation/results docs.

### DSP Cycle 5 — Script docs

- [ ] One doc per script under `tools/prod/**`, `scripts/**`,
      `backend/scripts/**` listed by `git ls-files`.
- [ ] Each doc cites the script's argparse / getopts / env reads,
      callers (grep), callees (grep), and any prod-state mutations.

### DSP Cycle 6 — Code-level docs (hot files only)

- [ ] At minimum the following modules get a code-level doc:
  - `backend/apps/video_analysis/tasks.py`
  - `backend/apps/pipeline/services/triton_client.py`
  - `backend/apps/pipeline/services/ensemble_validator.py`
  - `backend/apps/pipeline/services/logical_path_matrix.py`
  - `backend/apps/pipeline/services/triton_ensemble_input_size.py`
  - `backend/apps/telemetry/models.py`
  - `backend/apps/telemetry/services/writer.py`
- [ ] Each doc has a function-level call graph and a Mermaid sequence
      diagram for the dominant call path.

### DSP Cycle 7 — API surface docs

- [ ] Per REST endpoint exposed by `backend/config/urls.py` and the
      `urls.py` of each app.
- [ ] Per WebSocket route declared in `apps.video_analysis.routing.py`.
- [ ] Per Celery queue listed in `backend/config/celery.py`.
- [ ] Per Triton model route in
      `apps.pipeline.services.model_route_service`.
- [ ] Per Triton ensemble in
      `backend/models/triton_repository_cuda12/**/config.pbtxt`.

### DSP Cycle 8 — CI gate

- [ ] `scripts/ci/verify_entity_docs_and_diagrams.py` (new) asserts
      coverage parity between the entity inventory and `docs/entity/**`.
- [ ] Theme-initializer regex check passes on every Mermaid block.
- [ ] Label-length check passes (≤ 40 chars per logical line without break).
- [ ] Diagram-deletion check passes (`git log` comparison shows no
      removed mermaid block without a sibling replacement in the same
      commit).

## 6. Anti-hallucination protocol (mechanical)

Every entity doc MUST start with a `## Source-of-truth references`
section in this exact shape:

```markdown
## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `backend/apps/pipeline/services/triton_client.py` |
| File | `backend/apps/pipeline/services/ensemble_validator.py` |
| Symbol | `apps.pipeline.services.triton_client.TritonClient.infer` |
| Commit | `9bc53d86` |
| Job | `be4ba9ee-4786-48e9-8334-28feb237a1fb` |
| Workflow | `.github/workflows/inference-parallelization.yml` |
```

The CI gate parses this table and verifies:
- Every `File` row points to a path that exists in `git ls-files`.
- Every `Symbol` row matches a Python identifier under the cited file.
- Every `Commit` row matches `git cat-file -e <sha>` for a real commit.
- Every `Job` row matches a 36-char UUID present in some `inference_audit`
  / bench summary / cycle results doc.
- Every `Workflow` row points to a file under `.github/workflows/`.

**A doc that lacks the source-of-truth block fails the gate.**
**A doc with a broken reference fails the gate.**

## 7. Schedule (target dates)

| Cycle | Target close | Notes |
|---|---|---|
| 0 | 2026-06-02 (this commit) | Plan + governance |
| 1 | within 1 working day of 0 | Inventory + dates |
| 2 | within 3 working days of 1 | System docs (≤ 6 systems) |
| 3 | within 5 working days of 2 | Module docs (≈ 14 apps + frontend packages) |
| 4 | within 3 working days of 3 | Cycle docs (≈ 12 cycles) |
| 5 | within 5 working days of 4 | Script docs (≈ 50 scripts) |
| 6 | within 5 working days of 5 | Code docs (≤ 10 hot files) |
| 7 | within 5 working days of 6 | API surface docs |
| 8 | within 2 working days of 7 | CI gate + theme + deletion check |

Slippage is acceptable as long as the cycle is *visibly* in progress
(commits referring to it land at least once per working day). Silent
gaps trigger a §3.7 review.

## 8. Relation to the optimization-cycle program

This is a **documentation** program. It does not modify production
inference code, engines, env flags, or any other runtime state. It
runs in parallel with the optimization cycles (Cycles 1-13 of the
crop-frame plan). Optimization cycles continue to be governed by
[`docs/cycle_9_and_10_improvements_todo.md`](cycle_9_and_10_improvements_todo.md)
§ E hard rules; DSP cycles are governed by constitution Section 19
and this plan.

When the two intersect (e.g., DSP Cycle 4 documents Cycle 11), the
**optimization cycle docs are the source of truth** (`docs/cycle_11_input_size_results.md`,
`docs/cycle_11_input_size_investigation.md`). The DSP entity doc cites
them; it does NOT restate them or revise them.

## 9. Out of scope

Explicitly NOT covered by DSP (the verifier and reading-order also
skip these):

- Auto-generated mirror docs under `docs/backend/**`, `docs/frontend/**`,
  `docs/scripts/**` — those are produced by
  `scripts/ci/verify_docs_diagrams.py` and remain orthogonal.
- Generated diagram docs under `docs/diagrams/**` — produced by the
  diagram compiler.
- Point-in-time evidence reports under `ci_evidence/**` — preserved
  as-is; each one is dated within its own content.
- CI artifact summaries under
  `scripts/ci/artifacts/**/triton_stabilization_summary.md` — these are
  CI run outputs, generated, not narrative docs.
- Spec docs under `specs/**` — already follow Spec-Kit governance.
- Pytest cache READMEs (`**/.pytest_cache*/**`) — generated by pytest.
- Third-party READMEs in `.venv/`, `node_modules/`, tool-installed
  skill directories — never touched.
- The constitution at `.specify/memory/constitution.md` uses its own
  formal versioning header (`**Version**: X.Y.Z | **Ratified**: ... |
  **Last Amended**: ...`) — verifier regex accepts that as a valid
  date marker; no `**Last updated:**` header is added.

## 10. References

- [`docs/mermaid_theme_contract.md`](mermaid_theme_contract.md)
- [`docs/per_entity_doc_template.md`](per_entity_doc_template.md)
- [`AGENTS.md`](../AGENTS.md) DSP section
- [`.specify/memory/constitution.md`](../.specify/memory/constitution.md) Section 19
- [`README.md`](../README.md) Documentation Reading Order
- [`docs/INDEX.md`](INDEX.md)
- [`docs/cycle_9_and_10_improvements_todo.md`](cycle_9_and_10_improvements_todo.md) § Z (optimization-cycle map)
