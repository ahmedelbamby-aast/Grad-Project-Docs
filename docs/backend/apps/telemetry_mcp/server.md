# backend/apps/telemetry_mcp/server.py

## Source
- [backend/apps/telemetry_mcp/server.py](../../../../../backend/apps/telemetry_mcp/server.py)

## Purpose
Read-only MCP telemetry HTTP surface exposing bounded resource reads and a restricted query tool.

## Main structures and functions
- `RESOURCE_DESCRIPTORS`: allowlisted read-only resources mapped to models.
- `_BaseMCPView`: authenticated view base enforcing MCP role policy.
- `MCPResourceListView`: lists supported read-only resources.
- `MCPResourceDetailView`: resource reader with bounds parsing and audit outcomes.
- `MCPToolQueryView`: tool endpoint allowing only `query_resource` action.
- `_query_resource(...)`: bounded query executor for allowlisted resource.
- `_serialize_row(...)`: output serializer with redaction pass.

## Security and reliability behavior
- Enforces role checks before resource/tool access.
- Rejects unknown resources and non-read-only tool actions.
- Logs allow, deny, and failure outcomes through telemetry MCP audit helpers.
- Returns safe 503 responses for database or unexpected failures.

## Cross-links
- [security.md](security.md)
- [audit.md](audit.md)
- [../video_analysis/views.md](../video_analysis/views.md)
