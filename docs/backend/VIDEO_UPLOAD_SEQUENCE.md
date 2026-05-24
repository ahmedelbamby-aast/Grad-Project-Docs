# Video Upload Sequence

## Purpose

Shows the end-to-end journey from user upload to stored overlays and playback-ready data.

## Walkthrough

Read left to right: the user uploads a file, the API creates the job, Celery processes frames, tracking assigns IDs/colors, and storage persists the outputs for later playback.

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
sequenceDiagram
    actor User
    participant UI as Web UI
    participant API as Django API
    participant Celery as Celery Worker
    participant Track as Tracking Helpers
    participant Export as Video Exporter
    participant DB as PostgreSQL
    participant Cache as Redis

    User->>UI: Select .mp4 and submit
    UI->>API: POST upload form
    API->>DB: Create VideoAnalysisJob
    API->>Cache: Cache queued status
    API->>Celery: Enqueue process_video_upload()
    Celery->>DB: Create Frame records
    Celery->>Track: Resolve tracker + colors
    Track->>DB: Persist Detection / StudentTrack / BoundingBox
    Track->>Export: Optionally render annotated mp4
    Export->>DB: Persist annotated_video_path/url
    Track->>Cache: Cache embeddings / visibility / status
    Celery->>UI: Broadcast frame/status events
```

## Key Takeaways

- Upload handling is asynchronous.
- Tracking and rendering are separate steps.
- Redis holds fast-changing state; PostgreSQL keeps the durable records.

## Related Documents

- [Job State Diagram](JOB_STATE_DIAGRAM.md)
- [Architecture](../ARCHITECTURE.md)
