# tracking Class Diagram

## Purpose

Summarizes the helper types and service-style objects used by the tracking package.

## Walkthrough

Read left to right: tracker input is normalized, embeddings are generated, re-ID checks similarity, and rendering converts tracked state into stored overlay records.

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
classDiagram
    class TrackerAdapter {
        +resolve_tracker_yaml()
        +tracker_configuration()
        +is_reid_enabled()
        +track_frame(frame)
    }

    class ByteSortTracker {
        +load_model()
        +track_frame(frame)
        +resolve_tracker_yaml()
    }

    class ColorAssigner {
        +allocate_job_colors(job_id, tracking_ids)
        +assign_student_color(job_id, tracking_id)
    }

    class EmbeddingGenerator {
        +generate_embedding_vector(source)
        +normalize_vector(vector)
        +cache_embedding(track_id, vector)
        +persist_embedding(...)
    }

    class ReIDSimilarity {
        +cosine_similarity(left, right)
        +is_reid_match(candidate, reference, threshold)
        +find_best_reid_match(candidate_vector, reference_vectors, threshold)
        +prune_stale_reid_candidates(reference_vectors, max_gap_seconds)
    }

    class BoundingBoxRenderer {
        +create_bounding_box(...)
        +build_tracking_id_text(...)
    }

    class VideoExporter {
        +render_annotated_video(...)
    }

    TrackerAdapter <|-- ByteSortTracker
    ByteSortTracker --> ColorAssigner
    ByteSortTracker --> BoundingBoxRenderer
    ByteSortTracker --> VideoExporter
    EmbeddingGenerator --> ReIDSimilarity
    ReIDSimilarity --> TrackerAdapter
```

## Key Takeaways

- `ByteSortTracker` is the concrete adapter used in the pipeline.
- `TrackerAdapter` captures the resolved tracker configuration and allows future extension without changing callers.
- Re-ID operates on embeddings, while rendering operates on persisted overlay records.

## Related Documents

- [Tracking README](README.md)
- [Color Assignment Flowchart](color-assignment-flowchart.md)
- [Re-ID Flowchart](reid-flowchart.md)
