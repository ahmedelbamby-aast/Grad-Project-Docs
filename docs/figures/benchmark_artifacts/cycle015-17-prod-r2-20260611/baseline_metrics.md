# Production Benchmark Metrics

- Replay key: `cycle015-17-prod-r2-20260611-baseline`
- Job ID: `e60e678a-8862-4f87-8177-54b979aaf884`
- Status: `completed`
- Pipeline mode: `crop_frame`

## Metrics

| Metric | Value |
|---|---:|
| Processed frames | `4541/4541` |
| DB completed elapsed | `2742.376 s` |
| DB completed FPS | `1.656` |
| Step 2 frame wall | `1626.407 s` |
| Step 2 through pose upload | `2516.251 s` |
| Sharded child count | `0` |
| Shard merge wall | `0.000 s` |
| Async dispatch total | `0.000 ms` |
| Async dispatch mean | `0.000 ms` |
| Async dispatch calls | `0` |
| GPU avg util | `4.666%` |
| GPU peak util | `51.000%` |
| Peak VRAM | `16131.000 MiB` |
| Worker peak RSS | `0.000 MiB` |
| Embedding created span | `162.660 s` |
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
| `job_db_completed` | `4541` | `2742.376000` | `1.655863` | `frames/s` | VideoAnalysisJob.processed_frames / (completed_at - created_at) | `` |
| `step1_primary_tracking` | `4541` | `` | `` | `frames/s` | step1.primary_tracking.start -> step1.primary_tracking.complete | `step1_audit_timestamps_missing_or_skipped` |
| `step2_frame_inference_loop` | `4541` | `1626.406689` | `2.792045` | `frames/s` | step2.triton_shadow.start -> step2.frame_stage_timings | `` |
| `step2_through_pose_tail` | `4541` | `2516.250912` | `1.804669` | `frames/s` | step2.triton_shadow.start -> step2.pose_upload | `` |
| `step2_pose_tail_only` | `4541` | `889.844223` | `5.103140` | `frames/s` | step2.frame_stage_timings -> step2.pose_upload | `` |
| `step2_stage_decode_s` | `4541` | `0.074422` | `61016.903604` | `frames/s` | summed Step 2 per-frame stage timing | `` |
| `step2_stage_inference_s` | `4541` | `2164.949078` | `2.097509` | `frames/s` | summed Step 2 per-frame stage timing | `` |
| `step2_stage_postprocess_s` | `4541` | `4979.907167` | `0.911864` | `frames/s` | summed Step 2 per-frame stage timing | `` |
| `step2_stage_preprocess_s` | `4541` | `101.319574` | `44.818586` | `frames/s` | summed Step 2 per-frame stage timing | `` |
| `step3_persistence` | `4541` | `32.771042` | `138.567458` | `frames/s` | step3.persistence.start -> step3.persistence.complete | `` |
| `cycle20_post_stage_persistence` | `0` | `` | `` | `packets/s` | first_frame_persisted_at -> all_frames_persisted_at | `cycle20_post_stage_timeline_missing` |
| `embedding_stage_frame_equivalent` | `4541` | `` | `` | `frames/s` | embedding wall over processed frame count | `embedding_wall_missing` |
| `embedding_stage_work_items` | `252901` | `` | `` | `embedding-items/s` | embedding created + reused + skipped + error rows over embedding wall | `embedding_summary_or_wall_missing` |
| `step4_render` | `4541` | `27.761486` | `163.571936` | `frames/s` | step4.render.start -> step4.render.complete | `` |
| `post_render_finalize` | `4541` | `0.052379` | `86695.049543` | `frames/s` | step4.render.complete -> run.complete | `` |
| `telemetry_model_calls` | `13610` | `2742.376000` | `4.962850` | `calls/s` | TelemetryModelCall count over DB completed elapsed | `` |
| `db_frame_rows` | `4541` | `2742.376000` | `1.655863` | `rows/s` | db_frame_rows count over DB completed elapsed | `` |
| `db_detection_rows` | `127117` | `2742.376000` | `46.352871` | `rows/s` | db_detection_rows count over DB completed elapsed | `` |
| `db_bbox_rows` | `127117` | `2742.376000` | `46.352871` | `rows/s` | db_bbox_rows count over DB completed elapsed | `` |
| `db_embedding_rows` | `126519` | `2742.376000` | `46.134812` | `rows/s` | db_embedding_rows count over DB completed elapsed | `` |
| `db_pose_kinematics_rows` | `72931` | `2742.376000` | `26.594092` | `rows/s` | db_pose_kinematics_rows count over DB completed elapsed | `` |
| `boxes_person_detector` | `0` | `2742.376000` | `` | `boxes/s` | BoundingBox rows for person_detector over DB completed elapsed | `no_bbox_rows` |
| `boxes_posture_model` | `0` | `2742.376000` | `` | `boxes/s` | BoundingBox rows for posture_model over DB completed elapsed | `no_bbox_rows` |
| `boxes_horizontal_gaze_model` | `0` | `2742.376000` | `` | `boxes/s` | BoundingBox rows for horizontal_gaze_model over DB completed elapsed | `no_bbox_rows` |
| `boxes_vertical_gaze_model` | `0` | `2742.376000` | `` | `boxes/s` | BoundingBox rows for vertical_gaze_model over DB completed elapsed | `no_bbox_rows` |
| `boxes_depth_gaze_model` | `0` | `2742.376000` | `` | `boxes/s` | BoundingBox rows for depth_gaze_model over DB completed elapsed | `no_bbox_rows` |

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
| `decode_s` | `0.074422` | `0.001` | `61016.903604` | `0.016389` |
| `inference_s` | `2164.949078` | `29.877` | `2.097509` | `476.756018` |
| `postprocess_s` | `4979.907167` | `68.724` | `0.911864` | `1096.654298` |
| `preprocess_s` | `101.319574` | `1.398` | `44.818586` | `22.312172` |
| `stage_timing_sum` | `7246.250241` | `100.000` | `0.626669` | `1595.738877` |

## Per-Model Telemetry

| Model | Count | Calls/s | Mean ms | P50 ms | P95 ms | P99 ms | Max ms | Status | Shape |
|---|---:|---:|---:|---:|---:|---:|---:|---|---|
| `behavior_ensemble` | `3597` | `1.311636` | `111.791` | `155.474` | `182.639` | `192.551` | `214.331` | `ok:3597` | `[32, 3, 320, 320]` |
| `gaze_horizontal_model` | `1` | `0.000365` | `33.895` | `33.895` | `33.895` | `33.895` | `33.895` | `ok:1` | `[1, 3, 320, 320]` |
| `gaze_vertical_model` | `1` | `0.000365` | `31.930` | `31.930` | `31.930` | `31.930` | `31.930` | `ok:1` | `[1, 3, 320, 320]` |
| `person_detector` | `910` | `0.331829` | `16.627` | `13.794` | `42.971` | `51.338` | `56.616` | `ok:910` | `[1, 3, 640, 640]` |
| `posture_model` | `1` | `0.000365` | `34.694` | `34.694` | `34.694` | `34.694` | `34.694` | `ok:1` | `[1, 3, 320, 320]` |
| `rtmpose_model` | `4559` | `1.662427` | `42.568` | `42.672` | `45.269` | `73.726` | `86.880` | `ok:4559` | `[16, 3, 256, 192]` |
| `yoloe_scene_seg` | `4541` | `1.655863` | `25.115` | `19.766` | `45.175` | `60.088` | `93.057` | `ok:4541` | `[1, 3, 640, 640]` |

## Per-Model Box Throughput

> Metrics: boxes generated per frame, per second (job wall time), per ms, per model call, and per call-ms (box output rate vs per-call latency).
> Models with output_kind != bbox show unavailable_reason.

| Model | Output | Boxes total | /frame | /s | /ms | /call | /call-ms | Calls | Mean RTT ms | P95 RTT ms | Unavailable reason |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---|
| `depth_gaze_model` | `bbox` | `0` | `0.000` | `0.000` | `0.000000` | `` | `` | `0` | `0.000` | `0.000` | `no_bbox_rows` |
| `horizontal_gaze_model` | `bbox` | `0` | `0.000` | `0.000` | `0.000000` | `` | `` | `0` | `0.000` | `0.000` | `no_bbox_rows` |
| `person_detector` | `bbox` | `0` | `0.000` | `0.000` | `0.000000` | `0.000` | `0.000000` | `910` | `16.627` | `42.971` | `no_bbox_rows` |
| `posture_model` | `bbox` | `0` | `0.000` | `0.000` | `0.000000` | `0.000` | `0.000000` | `1` | `34.694` | `34.694` | `no_bbox_rows` |
| `rtmpose_model` | `keypoints` | `0` | `0.000` | `0.000` | `0.000000` | `0.000` | `0.000000` | `4559` | `42.568` | `45.269` | `output_kind_is_keypoints` |
| `vertical_gaze_model` | `bbox` | `0` | `0.000` | `0.000` | `0.000000` | `` | `` | `0` | `0.000` | `0.000` | `no_bbox_rows` |
| `yoloe_scene_seg` | `segmentation` | `0` | `0.000` | `0.000` | `0.000000` | `0.000` | `0.000000` | `4541` | `25.115` | `45.175` | `output_kind_is_segmentation` |


## Embedding Stage

| Metric | Value |
|---|---:|
| Created rows | `126519` |
| Reused track rows | `126382` |
| Skipped existing rows | `0` |
| Error rows | `0` |
| Stage outcome | `completed` |
| Created-at span | `162.660 s` |

## Cycle 17 Redis Stream

| Metric | Value |
|---|---:|
| Enabled | `False` |
| Stream key | `bench:cycle015-17-prod-r2-20260611-baseline:events` |
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
