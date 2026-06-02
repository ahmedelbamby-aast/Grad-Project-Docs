# 1. Executive Summary

**Last updated:** 2026-06-02

- `IMPLEMENTED`: Relational data models, runtime telemetry tables, REST routing, websocket transport wiring, and frontend runtime visualization/data hooks are present and line-evidenced. Evidence: `E:/grad_project/backend/apps/video_analysis/models.py:72`, `E:/grad_project/backend/apps/pipeline/models.py:21`, `E:/grad_project/backend/config/urls.py:14`, `E:/grad_project/backend/config/asgi.py:25`, `E:/grad_project/frontend/src/pages/RuntimePage.tsx:40`.
- `PARTIAL`: Contract hardening is incomplete due to broad serializer exposure and mixed websocket client patterns. Evidence: `E:/grad_project/backend/apps/recordings/serializers.py:9`, `E:/grad_project/backend/apps/exams/serializers.py:9`, `E:/grad_project/frontend/src/api/videoAnalysis.ts:101`, `E:/grad_project/frontend/src/hooks/useWebSocket.ts:60`.
- `NOT EVIDENCED`: A single repo-wide machine-readable unified API+websocket schema registry/version policy.

# 2. Audit Scope
- `IMPLEMENTED`: Scope covered DB schema, artifact persistence paths, API and websocket contracts, frontend runtime/debug/readiness paths.
- Evidence set includes:
`E:/grad_project/backend/apps/video_analysis/models.py`,
`E:/grad_project/backend/apps/sessions/models.py`,
`E:/grad_project/backend/apps/pipeline/models.py`,
`E:/grad_project/backend/apps/video_analysis/urls.py`,
`E:/grad_project/backend/apps/video_analysis/consumers.py`,
`E:/grad_project/backend/config/asgi.py`,
`E:/grad_project/frontend/src/api/runtime.ts`,
`E:/grad_project/frontend/src/hooks/useVideoAnalysisLive.ts`,
`E:/grad_project/frontend/src/hooks/useLiveSessionOverlay.ts`,
`E:/grad_project/frontend/src/pages/RuntimePage.tsx`.

# 3. Implemented (Evidence-Backed)
- `IMPLEMENTED`: Core DB entities and JSON-capable fields for video-analysis persistence. Evidence: `E:/grad_project/backend/apps/video_analysis/models.py:72`, `E:/grad_project/backend/apps/video_analysis/models.py:112`, `E:/grad_project/backend/apps/video_analysis/models.py:134`, `E:/grad_project/backend/apps/video_analysis/models.py:139`, `E:/grad_project/backend/apps/video_analysis/models.py:156`, `E:/grad_project/backend/apps/video_analysis/models.py:177`, `E:/grad_project/backend/apps/video_analysis/models.py:194`, `E:/grad_project/backend/apps/video_analysis/models.py:219`.
- `IMPLEMENTED`: Sessions data model with metadata/camera JSON fields. Evidence: `E:/grad_project/backend/apps/sessions/models.py:6`, `E:/grad_project/backend/apps/sessions/models.py:18`, `E:/grad_project/backend/apps/sessions/models.py:19`, `E:/grad_project/backend/apps/sessions/models.py:22`.
- `IMPLEMENTED`: Runtime telemetry model family exists in DB. Evidence: `E:/grad_project/backend/apps/pipeline/models.py:21`, `E:/grad_project/backend/apps/pipeline/models.py:44`, `E:/grad_project/backend/apps/pipeline/models.py:67`, `E:/grad_project/backend/apps/pipeline/models.py:90`, `E:/grad_project/backend/apps/pipeline/models.py:122`, `E:/grad_project/backend/apps/pipeline/models.py:156`, `E:/grad_project/backend/apps/pipeline/models.py:272`, `E:/grad_project/backend/apps/pipeline/models.py:306`, `E:/grad_project/backend/apps/pipeline/models.py:393`, `E:/grad_project/backend/apps/pipeline/models.py:442`.
- `IMPLEMENTED`: REST route inventory for runtime/data endpoints. Evidence: `E:/grad_project/backend/config/urls.py:14`, `E:/grad_project/backend/config/urls.py:15`, `E:/grad_project/backend/apps/video_analysis/urls.py:35`, `E:/grad_project/backend/apps/video_analysis/urls.py:39`, `E:/grad_project/backend/apps/video_analysis/urls.py:45`.
- `IMPLEMENTED`: ASGI websocket aggregation and app websocket routes. Evidence: `E:/grad_project/backend/config/asgi.py:25`, `E:/grad_project/backend/config/asgi.py:29`, `E:/grad_project/backend/apps/video_analysis/routing.py:6`, `E:/grad_project/backend/apps/sessions/routing.py:5`.
- `IMPLEMENTED`: Runtime websocket event types emitted. Evidence: `E:/grad_project/backend/apps/video_analysis/consumers.py:131`, `E:/grad_project/backend/apps/video_analysis/consumers.py:135`, `E:/grad_project/backend/apps/video_analysis/consumers.py:139`, `E:/grad_project/backend/apps/video_analysis/consumers.py:149`, `E:/grad_project/backend/apps/video_analysis/consumers.py:152`, `E:/grad_project/backend/apps/video_analysis/consumers.py:158`, `E:/grad_project/backend/apps/video_analysis/consumers.py:250`, `E:/grad_project/backend/apps/video_analysis/consumers.py:263`, `E:/grad_project/backend/apps/video_analysis/consumers.py:269`.
- `IMPLEMENTED`: Frontend runtime API client maps to runtime backend endpoints. Evidence: `E:/grad_project/frontend/src/api/runtime.ts:36`, `E:/grad_project/frontend/src/api/runtime.ts:44`, `E:/grad_project/frontend/src/api/runtime.ts:48`, `E:/grad_project/frontend/src/api/runtime.ts:56`, `E:/grad_project/frontend/src/api/runtime.ts:64`, `E:/grad_project/frontend/src/api/runtime.ts:73`, `E:/grad_project/frontend/src/api/runtime.ts:77`, `E:/grad_project/frontend/src/api/runtime.ts:85`, `E:/grad_project/frontend/src/api/runtime.ts:89`, `E:/grad_project/frontend/src/api/runtime.ts:93`.
- `IMPLEMENTED`: Frontend live overlay dedup by `event_id` implemented in two live paths. Evidence: `E:/grad_project/frontend/src/hooks/useVideoAnalysisLive.ts:394`, `E:/grad_project/frontend/src/hooks/useVideoAnalysisLive.ts:401`, `E:/grad_project/frontend/src/hooks/useLiveSessionOverlay.ts:280`, `E:/grad_project/frontend/src/hooks/useLiveSessionOverlay.ts:286`.
- `IMPLEMENTED`: Contract tests exist for runtime summary/ws/event schemas and MCP telemetry contract docs checks. Evidence: `E:/grad_project/backend/tests/contract/test_runtime_summary_api_contract.py:13`, `E:/grad_project/backend/tests/contract/test_runtime_websocket_contract.py:13`, `E:/grad_project/backend/tests/contract/test_runtime_event_schema_contract.py:4`, `E:/grad_project/backend/tests/contract/test_mcp_telemetry_contract.py:14`.

# 4. Partially Implemented (Evidence-Backed)
- `PARTIAL`: Persistence is mixed DB + external artifact state (frame-batching/runtime artifacts not only relational). Evidence: `E:/grad_project/backend/apps/video_analysis/urls.py:19`, `E:/grad_project/backend/apps/video_analysis/urls.py:36`, `E:/grad_project/backend/apps/video_analysis/urls.py:37`.
- `PARTIAL`: Contract strictness is uneven because some serializers expose all fields. Evidence: `E:/grad_project/backend/apps/recordings/serializers.py:9`, `E:/grad_project/backend/apps/exams/serializers.py:9`, `E:/grad_project/backend/apps/exams/serializers.py:15`, `E:/grad_project/backend/apps/exams/serializers.py:23`.
- `PARTIAL`: Data extraction availability has explicit 503 degradation paths in MCP telemetry server. Evidence: `E:/grad_project/backend/apps/telemetry_mcp/server.py:54`, `E:/grad_project/backend/apps/telemetry_mcp/server.py:57`, `E:/grad_project/backend/apps/telemetry_mcp/server.py:87`, `E:/grad_project/backend/apps/telemetry_mcp/server.py:90`.
- `PARTIAL`: Websocket schema version is signaled for telemetry ready event only; broader route-wide version enforcement is not evidenced. Evidence: `E:/grad_project/backend/apps/video_analysis/consumers.py:250`.

# 5. Missing (Evidence-Backed)
- `MISSING`: No evidence of a single canonical machine-readable registry covering all REST and websocket payload schemas together.
- Evidence checked: runtime event contract tests and typed client files are scoped/partial rather than a single unified registry: `E:/grad_project/backend/tests/contract/test_runtime_event_schema_contract.py:1`, `E:/grad_project/backend/tests/contract/test_runtime_websocket_contract.py:13`, `E:/grad_project/frontend/src/types/runtime.ts:1`, `E:/grad_project/frontend/src/types/websocket.ts:1`.

# 6. Architecturally Incorrect / Inconsistent
- `INCONSISTENT`: Frontend websocket connection management is duplicated across generic hook and direct socket construction path, which can diverge in reconnect/telemetry behavior.
- Evidence: reusable hook creates sockets (`E:/grad_project/frontend/src/hooks/useWebSocket.ts:60`, `E:/grad_project/frontend/src/hooks/useWebSocket.ts:135`) while direct constructor exists in API module (`E:/grad_project/frontend/src/api/videoAnalysis.ts:101`).

# 7. Scientifically Weak / Unvalidated
- `NOT EVIDENCED`: No repo artifact in this scope proving production-grade statistical validation for frontend data-readiness quality (beyond presence of tests).
- Evidence of existing but non-proof tests: `E:/grad_project/frontend/tests/integration/runtime-replay-observability.test.tsx:22`, `E:/grad_project/frontend/tests/unit/api/runtime-contracts.test.ts:11`.

# 8. Bottlenecks
- `PARTIAL`: Potential API pagination pressure path indicated by cursor-window slicing pattern (single-page overfetch by one), but no measured bottleneck data is provided in scope.
- Evidence (implementation only): `E:/grad_project/backend/apps/video_analysis/views.py:1389`, `E:/grad_project/backend/apps/video_analysis/views.py:1428`, `E:/grad_project/backend/apps/video_analysis/views.py:1470`.
- `NOT EVIDENCED`: Quantified latency/throughput bottleneck figures for DB/API/frontend path in this file scope.

# 9. Production Readiness Assessment
- `PARTIAL`: Core data/API/ws/frontend readiness primitives are implemented, with security gating present for telemetry MCP.
- Evidence: authenticated+role-gated MCP (`E:/grad_project/backend/apps/telemetry_mcp/server.py:23`, `E:/grad_project/backend/apps/telemetry_mcp/server.py:28`, `E:/grad_project/backend/apps/telemetry_mcp/security.py:39`, `E:/grad_project/backend/apps/telemetry_mcp/security.py:45`), runtime page and client contracts (`E:/grad_project/frontend/src/pages/RuntimePage.tsx:65`, `E:/grad_project/frontend/src/api/runtime.ts:64`).
- `PARTIAL`: Remaining gaps are unified schema governance and inconsistent frontend websocket wiring. Evidence: Section 5 and Section 6 references.

# 10. Equations Actually Used
- `IMPLEMENTED`: Cursor pagination equation in runtime list APIs: `next_after_id = window[limit - 1].id if len(window) > limit else None`. Evidence: `E:/grad_project/backend/apps/video_analysis/views.py:1389`, `E:/grad_project/backend/apps/video_analysis/views.py:1428`, `E:/grad_project/backend/apps/video_analysis/views.py:1470`.
- `IMPLEMENTED`: Runtime issues fill equation: `remaining = max(limit - len(issues), 0)`. Evidence: `E:/grad_project/backend/apps/video_analysis/views.py:1593`.
- `NOT EVIDENCED`: Additional math equations in DB/API/websocket/frontend scope.

# 11. Figures/Images Candidates from Repo
- `IMPLEMENTED`: `E:/grad_project/output/diagrams/backend/apps/video_analysis/consumers.mermaid.1.png`
- `IMPLEMENTED`: `E:/grad_project/output/diagrams/backend/apps/video_analysis/data-model-diagram.mermaid.1.png`
- `IMPLEMENTED`: `E:/grad_project/output/diagrams/backend/apps/sessions/consumers.mermaid.1.png`
- `IMPLEMENTED`: `E:/grad_project/output/diagrams/backend/apps/sessions/models.mermaid.1.png`

# 12. Contradictions (Code vs Docs vs Tests)
- `INCONSISTENT`: Codebase supports explicit runtime websocket event schema handling, but frontend has both centralized and ad-hoc socket creation, creating a contract drift vector not eliminated by current tests.
- Evidence: `E:/grad_project/backend/apps/video_analysis/consumers.py:250`, `E:/grad_project/frontend/src/hooks/useWebSocket.ts:60`, `E:/grad_project/frontend/src/api/videoAnalysis.ts:101`, `E:/grad_project/backend/tests/contract/test_runtime_websocket_contract.py:13`.
- `NOT EVIDENCED`: A single contradiction between backend contract tests and frontend runtime API endpoint paths; current endpoint strings align. Evidence: `E:/grad_project/backend/apps/video_analysis/urls.py:35`, `E:/grad_project/frontend/src/api/runtime.ts:36`.

# 13. Open Questions / Unknowns
- `NOT EVIDENCED`: Which artifact sources (DB vs filesystem vs cache) are authoritative for every runtime/frame-batching/debug endpoint under failure conditions.
- Evidence pointers for mixed sources needing explicit policy traceability: `E:/grad_project/backend/apps/video_analysis/urls.py:19`, `E:/grad_project/backend/apps/video_analysis/urls.py:25`, `E:/grad_project/backend/apps/video_analysis/urls.py:36`.
- `NOT EVIDENCED`: Repo-wide contract versioning policy across all websocket message families (not only telemetry ready). Evidence: `E:/grad_project/backend/apps/video_analysis/consumers.py:250`, `E:/grad_project/backend/apps/health/consumers.py:19`, `E:/grad_project/backend/apps/sessions/consumers.py:20`.

# 14. Subsystem Coverage Checklist
- `IMPLEMENTED` DB core entities: `E:/grad_project/backend/apps/video_analysis/models.py:72`, `E:/grad_project/backend/apps/sessions/models.py:6`, `E:/grad_project/backend/apps/pipeline/models.py:21`.
- `IMPLEMENTED` Artifact/telemetry endpoints: `E:/grad_project/backend/apps/video_analysis/urls.py:27`, `E:/grad_project/backend/apps/video_analysis/urls.py:29`, `E:/grad_project/backend/apps/video_analysis/urls.py:36`, `E:/grad_project/backend/apps/video_analysis/urls.py:37`.
- `IMPLEMENTED` REST contracts and runtime APIs: `E:/grad_project/backend/config/urls.py:14`, `E:/grad_project/frontend/src/api/runtime.ts:36`.
- `IMPLEMENTED` Websocket contracts and events: `E:/grad_project/backend/config/asgi.py:29`, `E:/grad_project/backend/apps/video_analysis/consumers.py:269`, `E:/grad_project/backend/apps/health/consumers.py:19`.
- `PARTIAL` Frontend websocket consistency: `E:/grad_project/frontend/src/hooks/useWebSocket.ts:135`, `E:/grad_project/frontend/src/api/videoAnalysis.ts:101`.
- `PARTIAL` Data extraction reliability: `E:/grad_project/backend/apps/telemetry_mcp/server.py:54`, `E:/grad_project/backend/apps/telemetry_mcp/server.py:87`.
- `NOT EVIDENCED` Unified repo-wide API+WS schema registry and global versioning governance.
