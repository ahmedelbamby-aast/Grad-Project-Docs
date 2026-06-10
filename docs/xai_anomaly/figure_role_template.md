# Figure Role Template

**Last updated:** 2026-06-10

## Source-of-truth references

| Kind | Reference |
|---|---|
| Doc | `specs/015-xai-anomaly-score/plan.md` |
| Doc | `specs/015-xai-anomaly-score/contracts/benchmark-evidence-contract.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |
| Doc | `.specify/memory/constitution.md` |

## Purpose

Use this template at the start of every Cycle 015 atomic cycle to declare
exactly one Figure Planner and exactly one Figure Implementer before any
benchmark-decision claim is made.

## Planner Template

```md
# Cycle 015.X Figure Plan

**Last updated:** YYYY-MM-DD
**Cycle:** `015.X`
**Role:** `Figure Planner`
**Owner:** `<agent or human owner>`
**Status:** `active`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `<raw artifact producer>` |
| File | `<generator>` |
| Doc | `specs/015-xai-anomaly-score/contracts/benchmark-evidence-contract.md` |
| Doc | `docs/xai_anomaly/cycle_015_X_investigation.md` |

## Planner decision

- one causal variable:
- decision state candidates:
- required raw artifacts:
- unavailable-metric policy:
- markdown embed targets:

## Required figures

| Figure | Purpose | Raw fields consumed |
|---|---|---|
| `<name>` | `<why>` | `<fields>` |
```

## Implementer Template

```md
# Cycle 015.X Figure Implementation

**Last updated:** YYYY-MM-DD
**Cycle:** `015.X`
**Role:** `Figure Implementer`
**Owner:** `<different owner when available>`
**Status:** `active`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `<generator>` |
| File | `<tests>` |
| Doc | `docs/xai_anomaly/cycle_015_X_figure_plan.md` |

## Ownership claim

- generator path:
- test paths:
- output directory:
- manifest schema:
- rollback expectations:

## Validation

- local generator tests:
- manifest/digest checks:
- unavailable-metric handling proof:
```

## Role-separation rule

- Preferred: different owners for planner and implementer.
- Allowed exception: one owner may cover both roles only when no second agent is
  available; the cycle docs must say so explicitly and still keep the plan and
  implementation records separate.
