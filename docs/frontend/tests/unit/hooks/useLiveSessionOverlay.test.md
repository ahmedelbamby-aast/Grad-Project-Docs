# frontend/tests/unit/hooks/useLiveSessionOverlay.test.tsx

## Coverage

- Verifies paint-cycle coalescing for rapid `overlay.frame` messages.
- Verifies newest-frame backpressure policy per camera before flush.
- Verifies detection-store receives only the latest frame for the coalesced cycle.

## Intent

Protect live overlay responsiveness by preventing per-message render/store churn during high-frequency frame bursts.

