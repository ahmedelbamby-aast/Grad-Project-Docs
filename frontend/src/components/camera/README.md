# Camera Components

**Last updated:** 2026-03-05

## Purpose
- UI components for camera tiles, grid behavior, overlays, and connection status.

## Key Components
- `CameraFeed`, `CameraGrid`, `AddCameraDialog`, `ClassFilterPanel`
- `CameraStatusBadge`, `ConnectionProgress`, `ErrorBanner`

## Notes
- Camera feed relies on `useWhepClient`.
- Status/progress components consume `cameraStore` state.
- `CameraFeed` derives `isReconnecting` from both `whepStatus === 'connecting'` / `'reconnecting'` and `camera.status === 'reconnecting'` to cover WHEP retries and backend-driven reconnects.
