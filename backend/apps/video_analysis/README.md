# Video Analysis Module

## Boundary Summary

| Field | Value |
| --- | --- |
| Purpose | Offline upload, jobs, batch processing, and preview orchestration |
| Responsibilities | Manage offline jobs; publish processed video results |
| Public inputs | Raw video; job commands |
| Public outputs | Job status; stored results; annotated media |
| Consumers | Frontend offline video UI, exports, recordings |
| Dependencies | Pipeline, tracking, storage |
| Failure behavior | Job failures persist retry-safe error status |

## Offline Prediction Flow

```mermaid
flowchart TD
    V["Raw video"] --> J["Video analysis job"]
    J --> P["Pipeline inference contract"]
    P --> T["Tracking output contract"]
    T --> R["Annotated media and stored results"]
```

The flow keeps offline orchestration behind job and pipeline contracts.
