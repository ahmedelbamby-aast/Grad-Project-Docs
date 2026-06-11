# Production Benchmark Metrics

- Replay key: `cycle015-17-prod-20260611-baseline`
- Job ID: `449f1540-f586-4049-829b-a4dd46bf166f`
- Status: `completed`
- Pipeline mode: `crop_frame`

## Metrics

| Metric | Value |
|---|---:|
| Processed frames | `4541/4541` |
| DB completed elapsed | `2729.178 s` |
| DB completed FPS | `1.664` |
| Step 2 frame wall | `1607.724 s` |
| Step 2 through pose upload | `2501.534 s` |
| Sharded child count | `0` |
| Shard merge wall | `0.000 s` |
| Async dispatch total | `0.000 ms` |
| Async dispatch mean | `0.000 ms` |
| Async dispatch calls | `0` |
| GPU avg util | `4.672%` |
| GPU peak util | `49.000%` |
| Peak VRAM | `32099.000 MiB` |
| Worker peak RSS | `0.000 MiB` |
| Embedding created span | `164.401 s` |
| Cycle20 timeline enabled | `False` |
| Cycle20 persistence wall | `0.000 s` |
| Cycle20 embedding wall | `0.000 s` |
| Detection rows | `127117` |
| BBox rows | `127117` |
| Embedding rows | `126519` |
| Pose kinematics rows | `72931` |
| Pose override rows | `0` |
| Student tracks | `138` |

## Layer Throughput

| Layer | Count | Wall s | Throughput | Unit | Basis | Unavailable reason |
|---|---:|---:|---:|---|---|---|
| `source_video_nominal` | `4541` | `151.366667` | `30.000000` | `frames/s` | job.metadata.fps and VideoAnalysisJob.total_frames | `` |
| `job_db_completed` | `4541` | `2729.178000` | `1.663871` | `frames/s` | VideoAnalysisJob.processed_frames / (completed_at - created_at) | `` |
| `step1_primary_tracking` | `4541` | `` | `` | `frames/s` | step1.primary_tracking.start -> step1.primary_tracking.complete | `step1_audit_timestamps_missing_or_skipped` |
| `step2_frame_inference_loop` | `4541` | `1607.723744` | `2.824490` | `frames/s` | step2.triton_shadow.start -> step2.frame_stage_timings | `` |
| `step2_through_pose_tail` | `4541` | `2501.534326` | `1.815286` | `frames/s` | step2.triton_shadow.start -> step2.pose_upload | `` |
| `step2_pose_tail_only` | `4541` | `893.810582` | `5.080495` | `frames/s` | step2.frame_stage_timings -> step2.pose_upload | `` |
| `step2_stage_decode_s` | `4541` | `0.064742` | `70139.940070` | `frames/s` | summed Step 2 per-frame stage timing | `` |
| `step2_stage_inference_s` | `4541` | `2153.903369` | `2.108265` | `frames/s` | summed Step 2 per-frame stage timing | `` |
| `step2_stage_postprocess_s` | `4541` | `4897.851158` | `0.927141` | `frames/s` | summed Step 2 per-frame stage timing | `` |
| `step2_stage_preprocess_s` | `4541` | `110.130605` | `41.232862` | `frames/s` | summed Step 2 per-frame stage timing | `` |
| `step3_persistence` | `4541` | `32.497800` | `139.732536` | `frames/s` | step3.persistence.start -> step3.persistence.complete | `` |
| `cycle20_post_stage_persistence` | `0` | `` | `` | `packets/s` | first_frame_persisted_at -> all_frames_persisted_at | `cycle20_post_stage_timeline_missing` |
| `embedding_stage_frame_equivalent` | `4541` | `` | `` | `frames/s` | embedding wall over processed frame count | `embedding_wall_missing` |
| `embedding_stage_work_items` | `252901` | `` | `` | `embedding-items/s` | embedding created + reused + skipped + error rows over embedding wall | `embedding_summary_or_wall_missing` |
| `step4_render` | `4541` | `27.887174` | `162.834714` | `frames/s` | step4.render.start -> step4.render.complete | `` |
| `post_render_finalize` | `4541` | `0.047849` | `94902.714790` | `frames/s` | step4.render.complete -> run.complete | `` |
| `telemetry_model_calls` | `13610` | `2729.178000` | `4.986850` | `calls/s` | TelemetryModelCall count over DB completed elapsed | `` |
| `db_frame_rows` | `4541` | `2729.178000` | `1.663871` | `rows/s` | db_frame_rows count over DB completed elapsed | `` |
| `db_detection_rows` | `127117` | `2729.178000` | `46.577028` | `rows/s` | db_detection_rows count over DB completed elapsed | `` |
| `db_bbox_rows` | `127117` | `2729.178000` | `46.577028` | `rows/s` | db_bbox_rows count over DB completed elapsed | `` |
| `db_embedding_rows` | `126519` | `2729.178000` | `46.357914` | `rows/s` | db_embedding_rows count over DB completed elapsed | `` |
| `db_pose_kinematics_rows` | `72931` | `2729.178000` | `26.722698` | `rows/s` | db_pose_kinematics_rows count over DB completed elapsed | `` |
| `boxes_person_detector` | `0` | `2729.178000` | `` | `boxes/s` | BoundingBox rows for person_detector over DB completed elapsed | `no_bbox_rows` |
| `boxes_posture_model` | `0` | `2729.178000` | `` | `boxes/s` | BoundingBox rows for posture_model over DB completed elapsed | `no_bbox_rows` |
| `boxes_horizontal_gaze_model` | `0` | `2729.178000` | `` | `boxes/s` | BoundingBox rows for horizontal_gaze_model over DB completed elapsed | `no_bbox_rows` |
| `boxes_vertical_gaze_model` | `0` | `2729.178000` | `` | `boxes/s` | BoundingBox rows for vertical_gaze_model over DB completed elapsed | `no_bbox_rows` |
| `boxes_depth_gaze_model` | `0` | `2729.178000` | `` | `boxes/s` | BoundingBox rows for depth_gaze_model over DB completed elapsed | `no_bbox_rows` |

## Cycle 20 Post-Stage Timeline

| Metric | Value |
|---|---:|
| Enabled | `False` |
| Streaming post-stages enabled | `False` |
| Profile | `` |
| Stream post-stage mode | `` |
| Authoritative write boundary | `` |
| Missing required fields | `["inference_started_at", "frame_inference_done_at", "first_persist_packet_ready_at", "first_frame_persisted_at", "all_frames_persisted_at", "first_embedding_eligible_at", "first_embedding_started_at", "embedding_done_at", "terminal_coordinator_done_at"]` |
| Unavailable fields | `{"cycle20_post_stage_timeline": {"reason": "metadata_missing"}}` |
| Persistence starts before inference done | `None` |
| Embedding starts before inference done | `None` |
| Persistence lag after inference done | `0.000 s` |
| Persistence wall | `0.000 s` |
| Embedding eligibility lag after persist | `0.000 s` |
| Embedding start lag after persist | `0.000 s` |
| Embedding start lag after inference done | `0.000 s` |
| Embedding wall | `0.000 s` |
| Terminal lag after embedding | `0.000 s` |
| Frame detection count | `0` |
| Persisted frame count | `0` |
| Streaming ready packets | `0` |
| Streaming DB-attempted packets | `0` |
| Final-stable persisted packets | `0` |
| Final-stable failed packets | `0` |
| Final-stable join wait | `0.000 ms` |
| Step 3 already-complete packets | `0` |
| Step 3 reconciled packets | `0` |
| Embedding stage outcome | `` |
| Timestamps | `{}` |

## Pose Kinematics

| Metric | Value |
|---|---:|
| Enabled | `True` |
| Current settings enabled | `True` |
| Records total | `72931` |
| Metadata record count | `72931` |
| Input record count | `72931` |
| Deduplicated record count | `72931` |
| Duplicate scope count | `0` |
| Duplicate record count | `0` |
| Override events total | `0` |
| Artifact refs | `72931` |
| Config digest count | `1` |
| History seconds setting | `5.0` |
| History max samples setting | `150` |
| Override margin setting | `0.15` |
| Override min frames setting | `3` |
| Min keypoint confidence setting | `0.3` |
| Max history window seconds | `5.000` |
| Avg history window seconds | `4.520` |
| Max history sample count | `150` |
| History bound violations | `0` |
| State counts | `{"degraded": 23961, "unavailable": 737, "valid": 48233}` |
| Quality counts | `{"good": 22135, "invalid": 737, "occluded": 1148, "partial_body": 35078, "weak": 13833}` |
| Override decision counts | `{}` |

## Step 2 Stage Timings

| Stage | Wall s | Share % | FPS equivalent | ms/frame |
|---|---:|---:|---:|---:|
| `decode_s` | `0.064742` | `0.001` | `70139.940070` | `0.014257` |
| `inference_s` | `2153.903369` | `30.074` | `2.108265` | `474.323578` |
| `postprocess_s` | `4897.851158` | `68.387` | `0.927141` | `1078.584267` |
| `preprocess_s` | `110.130605` | `1.538` | `41.232862` | `24.252501` |
| `stage_timing_sum` | `7161.949874` | `100.000` | `0.634045` | `1577.174603` |

## Per-Model Telemetry

| Model | Count | Calls/s | Mean ms | P50 ms | P95 ms | P99 ms | Max ms | Status | Shape |
|---|---:|---:|---:|---:|---:|---:|---:|---|---|
| `behavior_ensemble` | `3597` | `1.317979` | `113.006` | `158.545` | `180.136` | `190.369` | `1027.778` | `ok:3597` | `[32, 3, 320, 320]` |
| `gaze_horizontal_model` | `1` | `0.000366` | `28.230` | `28.230` | `28.230` | `28.230` | `28.230` | `ok:1` | `[1, 3, 320, 320]` |
| `gaze_vertical_model` | `1` | `0.000366` | `27.269` | `27.269` | `27.269` | `27.269` | `27.269` | `ok:1` | `[1, 3, 320, 320]` |
| `person_detector` | `910` | `0.333434` | `17.372` | `13.899` | `45.858` | `51.931` | `110.843` | `ok:910` | `[1, 3, 640, 640]` |
| `posture_model` | `1` | `0.000366` | `29.391` | `29.391` | `29.391` | `29.391` | `29.391` | `ok:1` | `[1, 3, 320, 320]` |
| `rtmpose_model` | `4559` | `1.670467` | `42.617` | `42.762` | `45.336` | `52.853` | `87.848` | `ok:4559` | `[16, 3, 256, 192]` |
| `yoloe_scene_seg` | `4541` | `1.663871` | `25.495` | `21.077` | `45.584` | `58.555` | `584.476` | `ok:4541` | `[1, 3, 640, 640]` |

## Per-Model Box Throughput

> Metrics: boxes generated per frame, per second (job wall time), per ms, per model call, and per call-ms (box output rate vs per-call latency).
> Models with output_kind != bbox show unavailable_reason.

| Model | Output | Boxes total | /frame | /s | /ms | /call | /call-ms | Calls | Mean RTT ms | P95 RTT ms | Unavailable reason |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---|
| `depth_gaze_model` | `bbox` | `0` | `0.000` | `0.000` | `0.000000` | `` | `` | `0` | `0.000` | `0.000` | `no_bbox_rows` |
| `horizontal_gaze_model` | `bbox` | `0` | `0.000` | `0.000` | `0.000000` | `` | `` | `0` | `0.000` | `0.000` | `no_bbox_rows` |
| `person_detector` | `bbox` | `0` | `0.000` | `0.000` | `0.000000` | `0.000` | `0.000000` | `910` | `17.372` | `45.858` | `no_bbox_rows` |
| `posture_model` | `bbox` | `0` | `0.000` | `0.000` | `0.000000` | `0.000` | `0.000000` | `1` | `29.391` | `29.391` | `no_bbox_rows` |
| `rtmpose_model` | `keypoints` | `0` | `0.000` | `0.000` | `0.000000` | `0.000` | `0.000000` | `4559` | `42.617` | `45.336` | `output_kind_is_keypoints` |
| `vertical_gaze_model` | `bbox` | `0` | `0.000` | `0.000` | `0.000000` | `` | `` | `0` | `0.000` | `0.000` | `no_bbox_rows` |
| `yoloe_scene_seg` | `segmentation` | `0` | `0.000` | `0.000` | `0.000000` | `0.000` | `0.000000` | `4541` | `25.495` | `45.584` | `output_kind_is_segmentation` |


## Embedding Stage

| Metric | Value |
|---|---:|
| Created rows | `126519` |
| Reused track rows | `126382` |
| Skipped existing rows | `0` |
| Error rows | `0` |
| Stage outcome | `completed` |
| Created-at span | `164.401 s` |

## Cycle 17 Redis Stream

| Metric | Value |
|---|---:|
| Enabled | `False` |
| Stream key | `bench:cycle015-17-prod-20260611-baseline:events` |
| Expected MAXLEN | `1000` |
| Expected TTL seconds | `86400` |
| Watch enabled | `False` |
| Watch DB full poll every N | `1` |
| XLen | `0` |
| Redis TTL seconds | `-2` |
| Memory usage bytes | `0` |
| Write count estimate | `0` |
| Write attempts | `0` |
| Writes | `0` |
| Redis unavailable | `0` |
| Write errors | `0` |
| Fallback count | `0` |
| DB polls avoided estimate | `0` |
| Read errors | `0` |
| First event ID | `` |
| Last event ID | `` |


## Async Dispatch

| Boundary | Calls | Total ms | Mean ms | Max ms |
|---|---:|---:|---:|---:|
| `ALL` | `0` | `0.000` | `0.000` | `0.000` |
