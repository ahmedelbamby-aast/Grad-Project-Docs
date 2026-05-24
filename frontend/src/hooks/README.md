# Hooks Module

## Purpose
- Reusable stateful logic for sockets, camera streaming, and session monitoring.

## Key Hooks
- `useWhepClient`
- `useCameraStatus`
- `useBulkFailure`
- `useWebSocket`, `useDetectionSocket`

## Notes
- Hooks are designed to be composable and side-effect scoped.
- `useWebSocket` auto-detects WS URL from `window.location` when `VITE_WS_BASE_URL` is not set (Vite proxy / nginx compatible).
- `useWhepClient` includes `[WHEP]`-prefixed debug logging and auto-retries on WebRTC ICE failure.
