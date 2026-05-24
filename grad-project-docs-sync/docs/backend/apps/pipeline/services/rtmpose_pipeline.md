# backend/apps/pipeline/services/rtmpose_pipeline.py

## Source
- [backend/apps/pipeline/services/rtmpose_pipeline.py](../../../../../../backend/apps/pipeline/services/rtmpose_pipeline.py)

## Purpose

Low-level RTMPose preprocessing/postprocessing helpers for ROI clamp, crop-resize tensor preparation, source-space mapping, temporal smoothing, and geometric metrics.

## Key utilities

- `clamp_box`: bounds-safe integer ROI.
- `crop_resize_people`: produces CHW normalized tensors and `RoiTransform`.
- `map_keypoints_to_source`: reprojects keypoints from model input space.
- `smooth_keypoints_ema`, `jitter_metric`, `fusion_valid_ratio`, `group_by_centroid_distance`: temporal/quality signals used by higher pipeline stages.

## Processing pipeline

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TD
    Boxes["detector boxes"] --> Clamp["clamp_box"]
    Clamp --> Crop["crop_resize_people"]
    Crop --> Inference["pose model inference"]
    Inference --> Map["map_keypoints_to_source"]
    Map --> Temporal["EMA smoothing / jitter / fusion metrics"]
```

## Cross-links

- [pose_runtime.md](pose_runtime.md)
- [../../tracking/reid.md](../../tracking/reid.md)

