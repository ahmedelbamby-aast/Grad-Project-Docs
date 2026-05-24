# backend/apps/telemetry_mcp/audit.py

## Source
- [backend/apps/telemetry_mcp/audit.py](../../../../../backend/apps/telemetry_mcp/audit.py)

## Purpose
Audit adapter that records telemetry MCP allow/deny/failure outcomes without breaking response flow.

## Main structures and functions
- `MCP_AUDIT_ACTION`: audit action key used for telemetry MCP events.
- `log_mcp_access(...)`: writes allowed-operation audit events.
- `log_mcp_denied(...)`: writes denied-operation audit events.
- `log_mcp_failure(...)`: writes failure-operation audit events.
- `_safe_log(...)`: shared writer that swallows audit subsystem failures.

## Behavior notes
- Captures surface/resource/outcome with optional reason/tool/bounds details.
- Delegates persistence to `apps.audit.services.AuditService`.
- Intentionally non-fatal when audit persistence throws.

## Cross-links
- [server.md](server.md)
- [security.md](security.md)
