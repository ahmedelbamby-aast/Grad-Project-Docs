# Job State Diagram

## Purpose

Captures the lifecycle of a video analysis job, including the clarified partial-success path.

## Walkthrough

The job starts queued, moves into processing and embedding, and then finishes as either completed, completed_partial, or failed.

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
stateDiagram-v2
    [*] --> queued
    queued --> processing
    processing --> embedding
    embedding --> completed
    embedding --> completed_partial
    processing --> failed
    embedding --> failed
    completed --> [*]
    completed_partial --> [*]
    failed --> [*]
    failed --> queued: retry endpoint
    completed_partial --> queued: retry endpoint
```

## Key Takeaways

- `completed_partial` is a first-class terminal state.
- `failed` is reserved for unrecoverable processing errors.
- The state flow is intentionally simple so the UI can present progress unambiguously.

## Related Documents

- [Video Analysis App](apps/video_analysis/README.md)
- [API README](api/README.md)
