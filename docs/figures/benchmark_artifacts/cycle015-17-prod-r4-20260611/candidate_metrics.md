# Production Benchmark Metrics

- Replay key: `cycle015-17-prod-r4-20260611-candidate`
- Job ID: `0d045329-6283-4cab-99c8-451cb200a71c`
- Status: `completed`
- Pipeline mode: `crop_frame`

## Metrics

| Metric | Value |
|---|---:|
| Processed frames | `4541/4541` |
| DB completed elapsed | `2659.457 s` |
| DB completed FPS | `1.707` |
| Step 2 frame wall | `1566.994 s` |
| Step 2 through pose upload | `2451.885 s` |
| Sharded child count | `0` |
| Shard merge wall | `0.000 s` |
| Async dispatch total | `0.000 ms` |
| Async dispatch mean | `0.000 ms` |
| Async dispatch calls | `0` |
| GPU avg util | `5.102%` |
| GPU peak util | `61.000%` |
| Peak VRAM | `16131.000 MiB` |
| Worker peak RSS | `0.000 MiB` |
| Embedding created span | `162.505 s` |
| Cycle20 timeline enabled | `False` |
| Cycle20 persistence wall | `0.000 s` |
| Cycle20 embedding wall | `0.000 s` |
| Detection rows | `127117` |
| BBox rows | `127117` |
| Embedding rows | `126519` |
| Pose kinematics rows | `72931` |
| Pose override rows | `0` |
| Student tracks | `139` |

## Layer Throughput

| Layer | Count | Wall s | Throughput | Unit | Basis | Unavailable reason |
|---|---:|---:|---:|---|---|---|
| `source_video_nominal` | `4541` | `151.366667` | `30.000000` | `frames/s` | job.metadata.fps and VideoAnalysisJob.total_frames | `` |
| `job_db_completed` | `4541` | `2659.457000` | `1.707491` | `frames/s` | VideoAnalysisJob.processed_frames / (completed_at - created_at) | `` |
| `step1_primary_tracking` | `4541` | `` | `` | `frames/s` | step1.primary_tracking.start -> step1.primary_tracking.complete | `step1_audit_timestamps_missing_or_skipped` |
| `step2_frame_inference_loop` | `4541` | `1566.993768` | `2.897906` | `frames/s` | step2.triton_shadow.start -> step2.frame_stage_timings | `` |
| `step2_through_pose_tail` | `4541` | `2451.884836` | `1.852045` | `frames/s` | step2.triton_shadow.start -> step2.pose_upload | `` |
| `step2_pose_tail_only` | `4541` | `884.891068` | `5.131705` | `frames/s` | step2.frame_stage_timings -> step2.pose_upload | `` |
| `step2_stage_decode_s` | `4541` | `0.068430` | `66359.783721` | `frames/s` | summed Step 2 per-frame stage timing | `` |
| `step2_stage_inference_s` | `4541` | `2068.375590` | `2.195443` | `frames/s` | summed Step 2 per-frame stage timing | `` |
| `step2_stage_postprocess_s` | `4541` | `4803.991660` | `0.945256` | `frames/s` | summed Step 2 per-frame stage timing | `` |
| `step2_stage_preprocess_s` | `4541` | `84.266997` | `53.888238` | `frames/s` | summed Step 2 per-frame stage timing | `` |
| `step3_persistence` | `4541` | `14.305064` | `317.440034` | `frames/s` | step3.persistence.start -> step3.persistence.complete | `` |
| `cycle20_post_stage_persistence` | `0` | `` | `` | `packets/s` | first_frame_persisted_at -> all_frames_persisted_at | `cycle20_post_stage_timeline_missing` |
| `embedding_stage_frame_equivalent` | `4541` | `` | `` | `frames/s` | embedding wall over processed frame count | `embedding_wall_missing` |
| `embedding_stage_work_items` | `252900` | `` | `` | `embedding-items/s` | embedding created + reused + skipped + error rows over embedding wall | `embedding_summary_or_wall_missing` |
| `step4_render` | `4541` | `27.844623` | `163.083551` | `frames/s` | step4.render.start -> step4.render.complete | `` |
| `post_render_finalize` | `4541` | `0.053176` | `85395.667218` | `frames/s` | step4.render.complete -> run.complete | `` |
| `telemetry_model_calls` | `13610` | `2659.457000` | `5.117586` | `calls/s` | TelemetryModelCall count over DB completed elapsed | `` |
| `db_frame_rows` | `4541` | `2659.457000` | `1.707491` | `rows/s` | db_frame_rows count over DB completed elapsed | `` |
| `db_detection_rows` | `127117` | `2659.457000` | `47.798103` | `rows/s` | db_detection_rows count over DB completed elapsed | `` |
| `db_bbox_rows` | `127117` | `2659.457000` | `47.798103` | `rows/s` | db_bbox_rows count over DB completed elapsed | `` |
| `db_embedding_rows` | `126519` | `2659.457000` | `47.573245` | `rows/s` | db_embedding_rows count over DB completed elapsed | `` |
| `db_pose_kinematics_rows` | `72931` | `2659.457000` | `27.423267` | `rows/s` | db_pose_kinematics_rows count over DB completed elapsed | `` |
| `boxes_person_detector` | `0` | `2659.457000` | `` | `boxes/s` | BoundingBox rows for person_detector over DB completed elapsed | `no_bbox_rows` |
| `boxes_posture_model` | `0` | `2659.457000` | `` | `boxes/s` | BoundingBox rows for posture_model over DB completed elapsed | `no_bbox_rows` |
| `boxes_horizontal_gaze_model` | `0` | `2659.457000` | `` | `boxes/s` | BoundingBox rows for horizontal_gaze_model over DB completed elapsed | `no_bbox_rows` |
| `boxes_vertical_gaze_model` | `0` | `2659.457000` | `` | `boxes/s` | BoundingBox rows for vertical_gaze_model over DB completed elapsed | `no_bbox_rows` |
| `boxes_depth_gaze_model` | `0` | `2659.457000` | `` | `boxes/s` | BoundingBox rows for depth_gaze_model over DB completed elapsed | `no_bbox_rows` |

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
| `decode_s` | `0.068430` | `0.001` | `66359.783721` | `0.015069` |
| `inference_s` | `2068.375590` | `29.732` | `2.195443` | `455.489009` |
| `postprocess_s` | `4803.991660` | `69.056` | `0.945256` | `1057.914922` |
| `preprocess_s` | `84.266997` | `1.211` | `53.888238` | `18.556925` |
| `stage_timing_sum` | `6956.702677` | `100.000` | `0.652752` | `1531.975925` |

## Per-Model Telemetry

| Model | Count | Calls/s | Mean ms | P50 ms | P95 ms | P99 ms | Max ms | Status | Shape |
|---|---:|---:|---:|---:|---:|---:|---:|---|---|
| `behavior_ensemble` | `3597` | `1.352532` | `101.711` | `139.063` | `168.198` | `178.395` | `199.572` | `ok:3597` | `[32, 3, 320, 320]` |
| `gaze_horizontal_model` | `1` | `0.000376` | `31.511` | `31.511` | `31.511` | `31.511` | `31.511` | `ok:1` | `[1, 3, 320, 320]` |
| `gaze_vertical_model` | `1` | `0.000376` | `30.421` | `30.421` | `30.421` | `30.421` | `30.421` | `ok:1` | `[1, 3, 320, 320]` |
| `person_detector` | `910` | `0.342175` | `16.771` | `14.027` | `44.019` | `50.220` | `57.628` | `ok:910` | `[1, 3, 640, 640]` |
| `posture_model` | `1` | `0.000376` | `33.060` | `33.060` | `33.060` | `33.060` | `33.060` | `ok:1` | `[1, 3, 320, 320]` |
| `rtmpose_model` | `4559` | `1.714260` | `43.529` | `43.306` | `47.342` | `51.674` | `90.047` | `ok:4559` | `[16, 3, 256, 192]` |
| `yoloe_scene_seg` | `4541` | `1.707491` | `25.241` | `20.330` | `45.046` | `55.322` | `76.144` | `ok:4541` | `[1, 3, 640, 640]` |

## Per-Model Box Throughput

> Metrics: boxes generated per frame, per second (job wall time), per ms, per model call, and per call-ms (box output rate vs per-call latency).
> Models with output_kind != bbox show unavailable_reason.

| Model | Output | Boxes total | /frame | /s | /ms | /call | /call-ms | Calls | Mean RTT ms | P95 RTT ms | Unavailable reason |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---|
| `depth_gaze_model` | `bbox` | `0` | `0.000` | `0.000` | `0.000000` | `` | `` | `0` | `0.000` | `0.000` | `no_bbox_rows` |
| `horizontal_gaze_model` | `bbox` | `0` | `0.000` | `0.000` | `0.000000` | `` | `` | `0` | `0.000` | `0.000` | `no_bbox_rows` |
| `person_detector` | `bbox` | `0` | `0.000` | `0.000` | `0.000000` | `0.000` | `0.000000` | `910` | `16.771` | `44.019` | `no_bbox_rows` |
| `posture_model` | `bbox` | `0` | `0.000` | `0.000` | `0.000000` | `0.000` | `0.000000` | `1` | `33.060` | `33.060` | `no_bbox_rows` |
| `rtmpose_model` | `keypoints` | `0` | `0.000` | `0.000` | `0.000000` | `0.000` | `0.000000` | `4559` | `43.529` | `47.342` | `output_kind_is_keypoints` |
| `vertical_gaze_model` | `bbox` | `0` | `0.000` | `0.000` | `0.000000` | `` | `` | `0` | `0.000` | `0.000` | `no_bbox_rows` |
| `yoloe_scene_seg` | `segmentation` | `0` | `0.000` | `0.000` | `0.000000` | `0.000` | `0.000000` | `4541` | `25.241` | `45.046` | `output_kind_is_segmentation` |


## Embedding Stage

| Metric | Value |
|---|---:|
| Created rows | `126519` |
| Reused track rows | `126381` |
| Skipped existing rows | `0` |
| Error rows | `0` |
| Stage outcome | `completed` |
| Created-at span | `162.505 s` |

## Cycle 17 Redis Stream

| Metric | Value |
|---|---:|
| Enabled | `False` |
| Stream key | `bench:cycle015-17-prod-r4-20260611-candidate:events` |
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

## Baseline Comparison

| Metric | Baseline | Candidate | Delta |
|---|---:|---:|---:|
| Status | `completed` | `completed` | `` |
| Processed frames | `4541` | `4541` | `+0.00%` |
| DB completed FPS | `1.654325` | `1.707491` | `+3.21%` |
| DB completed elapsed s | `2744.926` | `2659.457` | `-3.11%` |
| Step 2 frame wall s | `1621.705863` | `1566.993768` | `-3.37%` |
| Step 2 through pose upload s | `2506.292752` | `2451.884836` | `-2.17%` |
| Shard merge wall s | `None` | `None` | `` |
| Shard child count | `None` | `None` | `` |
| Pose tail drain after frame loop ms | `None` | `None` | `` |
| Pose tail runtime wall ms | `None` | `None` | `` |
| Pose tail provider async ms | `None` | `None` | `` |
| Pose tail artifact write ms | `None` | `None` | `` |
| Pose tail metadata save ms | `None` | `None` | `` |
| Async dispatch total ms | `0.0` | `0.0` | `n/a` |
| Async dispatch mean ms | `0.0` | `0.0` | `n/a` |
| Async dispatch calls | `0` | `0` | `n/a` |
| GPU avg util % | `4.247` | `5.102` | `+20.13%` |
| GPU peak util % | `49.0` | `61.0` | `+24.49%` |
| GPU peak mem MiB | `16131.0` | `16131.0` | `+0.00%` |
| Worker peak RSS MiB | `0.0` | `0.0` | `n/a` |
| Embedding created span s | `164.141` | `162.505` | `-1.00%` |
| Embedding created rows | `126519` | `126519` | `+0.00%` |
| Embedding reused rows | `126382` | `126381` | `-0.00%` |
| Embedding profile total ms | `None` | `None` | `` |
| Embedding DB flush ms | `None` | `None` | `` |
| Embedding Redis flush ms | `None` | `None` | `` |
| Embedding Redis read ms | `None` | `None` | `` |
| Embedding Redis estimated commands | `None` | `None` | `` |
| Embedding Redis pipeline executes | `None` | `None` | `` |
| Embedding Redis coalesced commands | `None` | `None` | `` |
| Embedding Redis coalesced pipelines | `None` | `None` | `` |
| Embedding Redis coalesced pipeline ms | `None` | `None` | `` |
| Embedding Redis coalesced errors | `None` | `None` | `` |
| Embedding Redis server command calls | `None` | `None` | `` |
| Embedding Redis server command ms | `None` | `None` | `` |
| Embedding Redis payload bytes | `None` | `None` | `` |
| Cycle20 timeline enabled | `False` | `False` | `n/a` |
| Cycle20 streaming post-stages enabled | `False` | `False` | `n/a` |
| Cycle20 persistence starts before inference done | `None` | `None` | `` |
| Cycle20 embedding starts before inference done | `None` | `None` | `` |
| Cycle20 persistence wall s | `None` | `None` | `` |
| Cycle20 embedding start lag after persist s | `None` | `None` | `` |
| Cycle20 embedding wall s | `None` | `None` | `` |
| Cycle20 terminal lag after embedding s | `None` | `None` | `` |
| Pose kinematics enabled | `True` | `True` | `+0.00%` |
| Pose kinematics records | `72931` | `72931` | `+0.00%` |
| Pose kinematics metadata records | `72931` | `72931` | `+0.00%` |
| Pose kinematics input records | `72931` | `72931` | `+0.00%` |
| Pose kinematics deduped records | `72931` | `72931` | `+0.00%` |
| Pose kinematics duplicate scopes | `0` | `0` | `n/a` |
| Pose kinematics duplicate records | `0` | `0` | `n/a` |
| Pose kinematics overrides | `0` | `0` | `n/a` |
| Pose kinematics artifacts | `72931` | `72931` | `+0.00%` |
| Pose kinematics config digests | `1` | `1` | `+0.00%` |
| Pose kinematics max history s | `5.0` | `5.0` | `+0.00%` |
| Pose kinematics avg history s | `4.520472912753168` | `4.520472912753168` | `+0.00%` |
| Pose kinematics max samples | `150` | `150` | `+0.00%` |
| Pose kinematics bound violations | `0` | `0` | `n/a` |
| Cycle17 stream events enabled | `False` | `False` | `n/a` |
| Cycle17 stream xlen | `0` | `0` | `n/a` |
| Cycle17 stream memory bytes | `0` | `0` | `n/a` |
| Cycle17 stream write attempts | `0` | `0` | `n/a` |
| Cycle17 stream writes | `0` | `0` | `n/a` |
| Cycle17 stream unavailable | `0` | `0` | `n/a` |
| Cycle17 stream write errors | `0` | `0` | `n/a` |
| Cycle17 stream read errors | `0` | `0` | `n/a` |
| Embedding exists check ms | `None` | `None` | `` |
| Embedding track lookup ms | `None` | `None` | `` |
| Embedding vector compute ms | `None` | `None` | `` |
| Embedding cv2 read ms | `None` | `None` | `` |
| Detection rows | `127117` | `127117` | `+0.00%` |
| BBox rows | `127117` | `127117` | `+0.00%` |
| Embedding rows | `126519` | `126519` | `+0.00%` |
| Pose kinematics rows | `72931` | `72931` | `+0.00%` |
| Pose override rows | `0` | `0` | `n/a` |
| Student tracks | `138` | `139` | `+0.72%` |
| Boxes/frame person_detector | `0.0` | `0.0` | `n/a` |
| Boxes/frame posture_model | `0.0` | `0.0` | `n/a` |
| Boxes/frame horizontal_gaze_model | `0.0` | `0.0` | `n/a` |
| Boxes/frame vertical_gaze_model | `0.0` | `0.0` | `n/a` |
| Boxes/frame depth_gaze_model | `0.0` | `0.0` | `n/a` |
| Boxes/s person_detector | `0.0` | `0.0` | `n/a` |
| Boxes/s posture_model | `0.0` | `0.0` | `n/a` |
| Boxes/s horizontal_gaze_model | `0.0` | `0.0` | `n/a` |
| Boxes/s vertical_gaze_model | `0.0` | `0.0` | `n/a` |
| Boxes/s depth_gaze_model | `0.0` | `0.0` | `n/a` |
| Boxes/ms person_detector | `0.0` | `0.0` | `n/a` |
| Boxes/ms posture_model | `0.0` | `0.0` | `n/a` |
| Boxes/ms horizontal_gaze_model | `0.0` | `0.0` | `n/a` |
| Boxes/ms vertical_gaze_model | `0.0` | `0.0` | `n/a` |
| Boxes/ms depth_gaze_model | `0.0` | `0.0` | `n/a` |
| Boxes/call-ms person_detector | `0.0` | `0.0` | `n/a` |
| Boxes/call-ms posture_model | `0.0` | `0.0` | `n/a` |
| Boxes/call-ms horizontal_gaze_model | `None` | `None` | `` |
| Boxes/call-ms vertical_gaze_model | `None` | `None` | `` |
| Boxes/call-ms depth_gaze_model | `None` | `None` | `` |
