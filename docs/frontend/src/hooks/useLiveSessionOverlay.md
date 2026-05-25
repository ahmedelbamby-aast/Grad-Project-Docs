# frontend/src/hooks/useLiveSessionOverlay.ts

## Purpose

Consumes live overlay WebSocket messages and projects them into detection-store frames plus camera metrics for UI rendering.

## Backpressure and Coalescing

- Overlay updates are coalesced per animation frame.
- For each camera, only the newest pending `overlay.frame` is kept before flush.
- Flush writes one frame per camera per paint cycle and updates metrics in a batched state update.
- Older out-of-order frames are dropped once a newer frame index is already applied.

## Runtime Impact

- Reduces render pressure under bursty live streams.
- Keeps freshness-first UI behavior while backend continues full-fidelity persistence.

