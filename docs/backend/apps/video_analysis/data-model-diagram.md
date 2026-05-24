# video_analysis Data Model

## Purpose

Shows how uploads, frames, detections, student tracks, embeddings, and bounding boxes connect across the analysis pipeline.

## Walkthrough

Read top to bottom: a `VideoAnalysisJob` owns frames and student tracks; frames hold detections; detections can produce embeddings; student tracks own the stable identity and color used by all overlays.

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
erDiagram
    VideoAnalysisJob ||--o{ Frame : contains
    VideoAnalysisJob ||--o{ StudentTrack : produces
    Frame ||--o{ Detection : has
    Detection ||--o{ FrameEmbedding : generates
    StudentTrack ||--o{ FrameEmbedding : uses
    Detection ||--o{ BoundingBox : renders
    StudentTrack ||--o{ BoundingBox : labels

    VideoAnalysisJob {
        UUID job_id PK
        string filename
        string file_path
        string pipeline_mode
        string annotated_video_path
        string status
        datetime created_at
        datetime started_at
        datetime completed_at
        float progress_percent
        int total_frames
        int processed_frames
        string error_message
        json component_failures
        json metadata
    }

    Frame {
        UUID frame_id PK
        UUID job_id FK
        int frame_number
        int timestamp_ms
        string image_path
        float fps
    }

    Detection {
        UUID detection_id PK
        UUID frame_id FK
        string class_name
        float confidence
        json bbox_xyxy
        string crop_image_path
    }

    FrameEmbedding {
        UUID embedding_id PK
        UUID detection_id FK
        UUID student_track_id FK
        string model_name
        vector[768] embedding_vector
        datetime created_at
        int embedding_age_seconds
    }

    StudentTrack {
        UUID track_id PK
        UUID job_id FK
        int tracking_id
        string color_hex
        datetime first_seen
        datetime last_seen
        int total_frames_visible
        json track_metadata
    }

    BoundingBox {
        UUID bbox_id PK
        UUID frame_id FK
        UUID detection_id FK
        UUID student_track_id FK
        json xyxy
        string color_hex
        string tracking_id_text
    }
```

## Key Takeaways

- `VideoAnalysisJob` is the lifecycle root.
- `StudentTrack.color_hex` is the identity anchor for every overlay rendered for that student.
- `BoundingBox` is a derived render record; visibility is applied at retrieval time, not stored as the source of truth.

## Related Documents

- [App README](README.md)
- [API README](../../api/README.md)
- [Tracking README](../tracking/README.md)
