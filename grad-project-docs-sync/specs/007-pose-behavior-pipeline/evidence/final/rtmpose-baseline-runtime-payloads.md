# RT003 Baseline Runtime Payloads

Date captured: 2026-05-17
Purpose: Preserve pre-change baseline runtime payload expectations.

## Baseline payload anchors
- Runtime persisted summary path: `backend/data/videos/{job_id}/results.json`
- Baseline export helper: `apps.pipeline.results.save_results_json`
- Existing summary serializer: `apps.pipeline.results.dump_job_summary`

## Baseline field sample (pre RT067-RT074)
```json
{
  "job_id": "<job_id>",
  "filename": "<video>",
  "status": "<status>",
  "metadata": {},
  "person_tracking_ids": [1, 2],
  "frames": [
    {
      "frame_number": 1,
      "timestamp_ms": 33,
      "image_path": "...",
      "detections": []
    }
  ]
}
```

## Notes
- No breaking schema mutation introduced by RT067-RT074.
- Added validation/scoring helpers are additive and reusable.
