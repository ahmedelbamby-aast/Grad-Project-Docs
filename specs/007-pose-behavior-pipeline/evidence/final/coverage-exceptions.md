# T104 Coverage Exceptions

Date: 2026-05-15

Gate status:
- Coverage gate failed (62% vs required 100%).

Approved exception register (temporary):
1. Owner: Backend platform team
   Scope: High-integration runtime and Triton-dependent paths under `backend/apps/video_analysis`, `backend/apps/tracking`, and selected `pipeline/services`
   Reason: Large untested branches are environment/hardware/integration coupled; additional deterministic harnessing is required.
   Expiry: 2026-06-30
   Removal plan: Add deterministic mocks/harnesses for inference and rendering branches, then ratchet minimum to 80/90/100 in staged CI gates.

2. Owner: Frontend camera UX team
   Scope: Camera grid interaction tests relying on browser-only APIs.
   Reason: `ResizeObserver` test-environment gaps and selector drift in fullscreen/grid tests.
   Expiry: 2026-06-15
   Removal plan: Add global test polyfill + align selectors/assertions with current DOM contract.

3. Owner: QA/integration
   Scope: End-to-end live/offline fallback routes requiring running backend dependencies.
   Reason: ECONNREFUSED in e2e indicates integration environment not fully provisioned in the current run.
   Expiry: 2026-06-15
   Removal plan: Add CI service orchestration checks before e2e start and fail-fast with explicit dependency status.
