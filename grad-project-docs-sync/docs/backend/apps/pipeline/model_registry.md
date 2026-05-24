# backend/apps/pipeline/model_registry.py

## Purpose

Defines registry-driven model descriptors used by unified runtime wiring.

## Public API

- `ModelDescriptor`: canonical runtime descriptor for a model family.
- `get_model_descriptor(model_name)`: resolve descriptor by model key.
- `iter_model_descriptors()`: list all descriptors in deterministic order.

## Notes

- Descriptor fields include `task_key`, `runtime_support`, `output_kind`, `student_scope`, `tracking_behavior`, and `ui_overlay_capability`.
- `apps.video_analysis.tasks` now resolves Triton task routing from this registry instead of hardcoded maps.

