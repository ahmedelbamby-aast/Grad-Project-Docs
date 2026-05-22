# Layers

Behavior-layer package for posture and gaze classification.

## Purpose

This package groups the four behavior classifiers used by the student behavior pyramid.

## Public API

- `PostureLayer`
- `HorizontalGazeLayer`
- `DepthGazeLayer`
- `VerticalGazeLayer`

## Dependencies

- `apps.pipeline.layers.base_behavior.BehaviorLayerBase`
- `apps.pipeline.config.PipelineConfig`
- `ultralytics`

## Usage

```python
from apps.pipeline.layers import PostureLayer

layer = PostureLayer()
```

## Related Documents

- [README.md](../README.md) - Pipeline package overview
- [../../../../docs/backend/apps/pipeline/layers/posture.md](../../../../docs/backend/apps/pipeline/layers/posture.md) - Posture layer docs
- [../../../../docs/backend/apps/pipeline/layers/horizontal_gaze.md](../../../../docs/backend/apps/pipeline/layers/horizontal_gaze.md) - Horizontal gaze docs
- [../../../../docs/backend/apps/pipeline/layers/depth_gaze.md](../../../../docs/backend/apps/pipeline/layers/depth_gaze.md) - Depth gaze docs
- [../../../../docs/backend/apps/pipeline/layers/vertical_gaze.md](../../../../docs/backend/apps/pipeline/layers/vertical_gaze.md) - Vertical gaze docs
- [../../../../.specify/memory/constitution.md](../../../../.specify/memory/constitution.md) - Project governance
