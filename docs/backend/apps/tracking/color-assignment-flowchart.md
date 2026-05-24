# Color Assignment Flowchart

## Purpose

Explains how a stable color is derived for each student track so the same student keeps the same color across all overlays.

## Walkthrough

The flow starts with a `job_id` and `tracking_id`, hashes them into a palette index, and then checks the per-job uniqueness constraint before finalizing the color.

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TD
    A[Start] --> B[Read job_id and tracking_id]
    B --> C[Hash job_id + tracking_id]
    C --> D[Map hash to palette index]
    D --> E[Select hex color from palette]
    E --> F{Color already used in job?}
    F -- No --> G[Persist StudentTrack.color_hex]
    F -- Yes --> H[Advance to next palette slot]
    H --> E
    G --> I[Return color]
```

## Key Takeaways

- The assignment is deterministic.
- The unique-per-job constraint prevents two students in the same video from sharing a color.
- The palette is large enough to support a crowded classroom without immediate collisions.

## Related Documents

- [Tracking README](README.md)
- [Class Diagram](class-diagram.md)
