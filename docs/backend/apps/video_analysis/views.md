# backend/apps/video_analysis/views.py

## Source
- [backend/apps/video_analysis/views.py](../../../../../backend/apps/video_analysis/views.py)

## Purpose
REST boundary for video-analysis workflows, runtime dashboards, and job-scoped telemetry projections.

## Key telemetry and reconciliation surface
- Reconciliation and actionable-error helpers:
  - `_normalize_reconciliation_run_summary(...)`
  - `_actionable_error_for_signal(...)`
  - `_collect_actionable_error_signals(...)`
  - `_build_actionable_errors(...)`
- Telemetry bounds/session helpers:
  - `_telemetry_bounds_from_request(...)`
  - `_resolve_session_id_for_job(...)`
- Dashboard telemetry endpoints:
  - `JobTelemetrySummaryView`
  - `JobTelemetryTimelineView`
  - `JobTelemetryArtifactsView`
  - `JobRuntimeAssignmentsView`
  - `JobBenchmarkComparisonView`
- Runtime telemetry endpoints:
  - `RuntimeSessionsView`
  - `RuntimeFrameEventsView`
  - `RuntimeModelEventsView`
  - `RuntimeSummaryView`
  - `RuntimeIssuesView`
  - `RuntimeFiltersView`
  - `RuntimeTimelineView`
  - `RuntimeModelMatrixView`
  - `RuntimeStudentTimelineView`
  - `RuntimeProfilesView`

## Architectural role
HTTP/API transport layer that validates request bounds and returns normalized, UI-safe telemetry and reconciliation data.

## Cross-links
- [../pipeline/runtime_ingestion.md](../pipeline/runtime_ingestion.md)
- [../telemetry_mcp/server.md](../telemetry_mcp/server.md)
