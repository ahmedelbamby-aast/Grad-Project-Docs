# Offline-Video Real-Data Validation

## Result

Blocked in this local run.

## Evidence

Offline-video integration tests attempted raw-media paths and failed on missing `backend/data/videos/.../input.mp4`, TensorRT runtime gaps, and local test database contention. The frontend offline navigation path is covered by the full-baseline e2e, but real raw-video validation still requires the prepared media fixture and model runtime lane.
