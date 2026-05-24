# backend/apps/telemetry_mcp/security.py

## Source
- [backend/apps/telemetry_mcp/security.py](../../../../../backend/apps/telemetry_mcp/security.py)

## Purpose
Centralized RBAC, query bounds, and payload redaction rules for telemetry MCP.

## Main structures and functions
- `MCPPermissionDenied`: explicit authorization error.
- `Bounds`: immutable limit/offset contract.
- `enforce_role(user)`: allows only authenticated teacher/admin roles (or staff/superuser).
- `parse_bounds(query_params)`: clamps pagination to safe defaults and maximums.
- `apply_bounds(qs, bounds)`: applies bounded slicing.
- `redact_payload(value)`: recursively redacts PII/security-sensitive keys.

## Governing constants
- `MAX_LIMIT = 200`
- `DEFAULT_LIMIT = 50`
- `ALLOWED_ROLES = {"teacher", "admin"}`
- `PII_KEYS`: denylist used for recursive response redaction.

## Cross-links
- [server.md](server.md)
- [audit.md](audit.md)
