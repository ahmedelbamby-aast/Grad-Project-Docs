# T090 Offline Metrics Validation

Date: 2026-05-15

Evidence inputs:
- Full backend run (T087) includes offline/video-analysis and end-to-end tests.

Validated status:
- Offline metric validation executed with real suite run, but gate is NOT green.

Observed blockers:
- Failing offline/workflow-related tests (e.g., upload/workflow and video export edge behavior) prevent acceptance of frame coverage and event consistency targets.

Conclusion:
- Offline frame coverage and event consistency require remediation of failing backend tests before this gate can pass.
