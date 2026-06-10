# Cycle 015.7 Investigation — Cross-Model Fusion And Explanation Graph

**Last updated:** 2026-06-10
**Cycle:** `015.7`
**Status:** `staged_local_only`
**Streaming compatibility:** `stream-safe-with-config`

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `backend/apps/anomalies/scoring/fusion.py` |
| File | `backend/apps/behavior/explainability/contradictions.py` |
| File | `backend/apps/behavior/explainability/context.py` |
| File | `backend/apps/behavior/explainability/graph.py` |
| File | `backend/apps/behavior/explainability/composer.py` |
| File | `backend/apps/behavior/models.py` |
| File | `backend/apps/behavior/serializers.py` |
| File | `backend/apps/behavior/views.py` |
| File | `backend/tests/unit/behavior/test_xai_explanation_graph.py` |
| File | `backend/tests/integration/behavior/test_xai_context_identity.py` |
| Doc | `specs/015-xai-anomaly-score/spec.md` |
| Doc | `specs/015-xai-anomaly-score/plan.md` |
| Doc | `specs/015-xai-anomaly-score/tasks.md` |

## Goal

Cycle 015.7 adds source-preserving fusion, identity-gated peer context, and a
reconstructable explanation graph so reviewers can inspect how contributions,
context, and contradictions relate without forcing identity or behavioral
conclusions.

## Identity-gated fusion protocol

- peer-context evidence stays source-scoped;
- unresolved or weak-identity context is preserved as suppressed/unresolved, not
  coerced into valid support;
- scene and SRVL context may support a student-local explanation only when tied
  to the producing student or explicitly marked non-peer;
- contradiction evidence remains append-only and visible even when fusion
  suppresses a source.

## Local implementation boundary

This local slice is complete only when:

- fusion summaries preserve every source and contradiction reference;
- peer context can be suppressed on unresolved identity continuity;
- explanation graph nodes and edges are deterministic and reconstructable; and
- an explanation record can be persisted and read without mutating the source
  score record.
