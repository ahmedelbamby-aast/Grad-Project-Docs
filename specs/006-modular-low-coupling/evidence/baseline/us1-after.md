# US1 Post-Refactor Baseline Evidence

## Scope

This evidence pack records the post-refactor baseline for the live-stream, offline-video, and non-video dashboard workflows after publishing frontend API, store, and page boundary contracts.

## Implemented Evidence Points

| Scenario | Evidence | Result |
| --- | --- | --- |
| Live stream | `CAMERA_FEED_PAGE_INTERACTION_CONTRACT`, `CAMERA_STORE_TRANSITION_CONTRACT`, `selectCameraWorkflowState`, backend live equivalence tests | Preserved camera/session loading, connected camera filtering, per-camera loading, grid columns, and visible live controls |
| Offline video | `VIDEO_ANALYSIS_PAGE_INTERACTION_CONTRACT`, `RECORDING_DETAIL_PAGE_INTERACTION_CONTRACT`, `UPLOAD_STORE_TRANSITION_CONTRACT`, `selectUploadWorkflowState`, backend offline equivalence tests | Preserved upload jobs, selected job state, playback/review surface, anomaly timeline, and export access |
| Non-video dashboard | `DASHBOARD_PAGE_INTERACTION_CONTRACT`, `HEALTH_PAGE_INTERACTION_CONTRACT`, `HEALTH_STORE_TRANSITION_CONTRACT`, `selectHealthWorkflowState` | Preserved dashboard counts/navigation, health status, storage warning state, and page-level isolation |

## Verification Commands

| Command | Result |
| --- | --- |
| `npm run type-check` from `frontend/` | Passed |
| `npm test -- tests/integration/modular-performance.test.ts tests/unit/api/modular_contracts.test.ts --run` from `frontend/` | Passed: 2 files, 4 tests |
| Backend focused US1 tests from prior baseline slice | Passed: 8 tests |
| Frontend full-baseline e2e from prior baseline slice | Passed: 1 test, with Vite WebSocket close logs but zero exit status |

## Notes

The post-refactor contracts are additive and do not change existing rendered UI behavior. They expose the workflow boundaries needed for future dependency-direction and documentation-alignment checks while keeping delivered live, offline, dashboard, and health interactions on their existing API/store paths.
