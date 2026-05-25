# RT074 Scientific Validation Method

Date: 2026-05-17

## Deterministic Metrics
- `temporal_stability_score(points)`
  - Input: ordered `(x, y)` keypoint positions.
  - Output: `[0,1]`, computed as `1 / (1 + mean_frame_delta)`.
- `temporal_continuity_score(timestamps_ms, max_gap_ms=120)`
  - Penalizes negative deltas and oversized gaps.
  - Output: `1 - (penalties / comparisons)`.
- `noise_reduction_score(raw_values, denoised_values)`
  - Variance-reduction ratio, clipped to `[0,1]`.

## Deterministic Processing Helpers
- `moving_average_denoise(values, window=3)` for stable, reproducible smoothing baseline.
- `aggregate_scores(scores)` for reusable `{mean,min,max,count}` summaries.
- `export_contrastive_tensor_sample(...)` for contrastive-ready temporal payload sanity.

## Thresholds Used in RT067-RT071
- Temporal stability: `>= 0.8` for low-motion synthetic track.
- Noise reduction: `> 0.2` improvement for deterministic noisy signal.
- Continuity robustness: clean stream `== 1.0`; degraded stream lower.
- Multi-person long-session continuity: minimum `>= 0.95`, mean `>= 0.98`.
- Tensor export: consistent `TJC_flat_xyc` layout and frame shape invariants.

## Reproducibility
- All tests are pure deterministic input/output checks.
- No network, camera, model runtime, or random seed dependency.
