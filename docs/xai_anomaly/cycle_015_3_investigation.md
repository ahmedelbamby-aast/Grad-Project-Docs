# Cycle 015.3 Investigation: Deterministic Fast Explainers For Every Active Model

**Created:** 2026-06-11
**Status:** `implemented_local; awaiting production benchmark`
**Streaming compatibility:** `stream-safe` (bounded per-observation computation;
no saliency, artifact writes, or unbounded queries on the fast path)

## Adapter coverage matrix

| fast_explainer_key | Adapter | Signals covered | Deep methods |
|---|---|---|---|
| fast.person_detector | adapters/person_detector.py | detector.* (5) | d_rise, eigen_cam_if_validated |
| fast.behavior_models | adapters/behavior_models.py | behavior.* (6) | crop_perturbation, IG-if-supported |
| fast.pose | adapters/pose.py | pose.* (4) | joint occlusion, heatmap-if-exposed |
| fast.reid_tracking | adapters/reid_tracking.py | reid.*, tracking.* (3) | governed exemplar comparison |
| fast.yoloe_scene | adapters/yoloe_scene.py | scene.* (2) | d_rise, mask deletion/insertion |
| fast.srvl | adapters/srvl.py | srvl.* (2) | none (deterministic) |
| fast.interaction_graph | adapters/srvl.py | interaction_graph.* (3) | none (deterministic) |
| fast.bsil_rules | adapters/bsil_rules.py | bsil.* (3) | none (deterministic) |
| fast.pattern_window | adapters/bsil_rules.py | pattern.* (3) | none (deterministic) |
| fast.runtime | adapters/bsil_rules.py | runtime.* (3) | none (deterministic) |

Coverage gate: `SignalRegistry.verify_fast_explainer_coverage()` returns zero
uncovered enabled signals (SC-001/FR-003); unregistered routes are refused.

## Contract decisions

- Fragments are deterministic: same envelope -> identical sha256 digest.
- Counterfactual thresholds come from the producer payload; absent ->
  explicit `threshold_not_provided_by_producer` (no hardcoded thresholds).
- Calibration attaches through the 015.2 lookup; unavailable states carry the
  governed reason (raw score preserved, never presented as certainty).
- ReID explanations expose distances/thresholds/lifecycle only - never
  embedding-dimension semantics (research Decision 2).
- T080: the placeholder note writer is RETIRED with an explicit refusal;
  `write_explainability_artifact` requires digest-bearing fragments (FR-030).

## Open acceptance gates

Production benchmark (post-015.17): per-adapter latency within budget,
deterministic replay on production media, output parity, no hidden fallback,
figures + ledger + rollback.
