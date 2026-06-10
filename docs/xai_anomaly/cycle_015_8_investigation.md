# Cycle 015.8 Investigation — Offline And On-Demand Deep Vision XAI

**Last updated:** 2026-06-10
**Cycle:** `015.8`
**Status:** `staged_local_only`
**Streaming compatibility:** `offline-only`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `backend/apps/behavior/explainability/deep_contracts.py` |
| File | `backend/apps/behavior/models.py` |
| File | `backend/tests/unit/behavior/test_xai_deep_methods.py` |
| Doc | `specs/015-xai-anomaly-score/spec.md` |
| Doc | `specs/015-xai-anomaly-score/plan.md` |
| Doc | `specs/015-xai-anomaly-score/data-model.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |

## Goal

Cycle 015.8 introduces the deep-XAI request, artifact, and evaluation lifecycle
without yet enabling runtime execution. The local scope is the contract and
persistence boundary that later adapters and Celery isolation will use.

## Per-model method eligibility matrix

The initial eligibility contract is intentionally method-first and route-aware:

| Method key | Intended source family | Eligibility rule in this slice |
|---|---|---|
| `drise` | detector, segmentation | requires completed evidence ref, route snapshot, artifact budget, and allowlisted method key |
| `integrated_gradients` | differentiable behavior heads | requires completed evidence ref, route snapshot, gradient-capable family, and allowlisted method key |
| `cam` | differentiable conv heads | same gating as IG, but keeps method identity separate |
| `pose_joint_occlusion` | pose/kinematics | requires pose-capable family and explicit route snapshot |
| `reid_exemplar` | ReID / embedding comparison | requires embedding-capable family and authenticated artifact access boundary |

This cycle does not claim any method is already faithful. It only defines how a
later adapter must declare and persist its eligibility, request, artifact, and
evaluation state.

## Local implementation boundary

This local slice is complete only when:

- deep-XAI requests carry explicit method key, route snapshot, idempotency, and
  deadline/budget fields;
- artifacts and evaluations persist with append-only digests and lifecycle
  status;
- status enums are bounded and non-ambiguous; and
- unit tests prove payload serialization, allowlist gating, deadline/idempotency
  handling, and evaluation digest stability.
