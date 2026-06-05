# Quickstart: Human Pose Kinematics Layer

**Last updated:** 2026-06-05

## Preconditions

- PostgreSQL is the only accepted database backend.
- RTMPose remains the authoritative pose keypoint source.
- The feature is on by default after the 2026-06-05 initial production
  enablement exception for the offline-proven profile.
- All thresholds are read from `.env`/Django settings; no service logic may
  hardcode override gates.

## Configuration

These keys are now read in `backend/config/settings/base.py` and consumed
through `PoseKinematicsConfig`:

```bash
POSE_KINEMATICS_ENABLED=1
POSE_KINEMATICS_HISTORY_SECONDS=5
POSE_KINEMATICS_OVERRIDE_MARGIN=0.15
POSE_KINEMATICS_OVERRIDE_MIN_FRAMES=3
POSE_KINEMATICS_ARTIFACTS_ENABLED=1
POSE_KINEMATICS_TELEMETRY_ENABLED=1
```

`POSE_KINEMATICS_ENABLED` is `1` by default. Production validation wrappers can
still force `0` for baseline-disabled and rollback attempts.

## Local Validation Shape

Run focused PostgreSQL-backed tests first:

```powershell
cd E:\grad_project
.\backend\.venv\Scripts\python.exe -m pytest `
  backend\tests\unit\config\test_pose_kinematics_settings.py `
  backend\tests\unit\pipeline\test_pose_kinematics_quality.py `
  backend\tests\unit\pipeline\test_pose_kinematics_geometry.py `
  backend\tests\unit\pipeline\test_pose_kinematics_orientation.py `
  backend\tests\unit\pipeline\test_pose_kinematics_posture_gesture.py `
  backend\tests\unit\pipeline\test_pose_kinematics_constraints.py `
  backend\tests\unit\pipeline\test_pose_kinematics_history.py `
  backend\tests\unit\pipeline\test_pose_kinematics_temporal.py `
  backend\tests\unit\pipeline\test_pose_kinematics_override_gate.py `
  backend\tests\unit\pipeline\test_pose_kinematics_fusion.py `
  backend\tests\unit\video_analysis\test_pose_kinematics_models.py `
  backend\tests\unit\video_analysis\test_pose_kinematics_override_events.py `
  backend\tests\contract\test_pose_kinematics_contract.py `
  backend\tests\contract\test_pose_kinematics_summary_api.py `
  backend\tests\contract\test_pose_kinematics_telemetry_contract.py `
  backend\tests\contract\test_pose_kinematics_history_contract.py `
  backend\tests\contract\test_pose_kinematics_override_contract.py `
  backend\tests\integration\test_pose_kinematics_offline_replay.py `
  backend\tests\integration\test_pose_kinematics_live_streaming.py `
  backend\tests\integration\test_pose_kinematics_fusion_flow.py `
  backend\tests\integration\test_pose_kinematics_reviewer_labels.py `
  -q --tb=short
```

Run the static hardening gate:

```powershell
cd E:\grad_project
.\backend\.venv\Scripts\python.exe scripts\ci\verify_pose_kinematics_requirements_gates.py
```

## Production Offline Benchmark Matrix

Use the benchmark wrapper so baseline-disabled and candidate-enabled attempts
share the same video, replay policy, metrics collector, and evidence directory:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_run_pose_kinematics_benchmark.sh \
  --video "/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4" \
  --pipeline-mode crop_frame \
  --profile per-frame-signals \
  --timeout 7200 \
  --tag pose-kinematics-prod-$(date -u +%Y%m%dT%H%M%SZ)
```

Watch the active run from another shell:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_watch_benchmark_metrics.sh --latest --interval 10 --clear
```

## Live Validation

Run a real-media live profile when a stream is available:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_run_pose_kinematics_live_validation.sh \
  --stream-url "rtsp://<redacted-real-media-source>" \
  --actor runtime_validator \
  --timeout 900
```

If a prior real-media run has already produced a governed manifest, verify it:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_run_pose_kinematics_live_validation.sh \
  --live-capture-manifest backend/logs/<tag>/live_validation_manifest.json
```

## Reconciliation and Label Agreement

```bash
cd /home/bamby/grad_project
python tools/prod/prod_check_pose_kinematics_reconciliation.py \
  --replay-key "<candidate-replay-key>" \
  --metrics-json backend/logs/<tag>/candidate_enabled_metrics.json \
  --output backend/logs/<tag>/candidate_reconciliation.json \
  --markdown-output backend/logs/<tag>/candidate_reconciliation.md

python tools/prod/prod_collect_pose_kinematics_label_agreement.py \
  --manifest backend/logs/<tag>/reviewer_label_manifest.json \
  --output backend/logs/<tag>/reviewer_label_agreement.json \
  --markdown-output backend/logs/<tag>/reviewer_label_agreement.md
```

## Rollback Verification

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_verify_pose_kinematics_rollback.sh \
  --video "/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4" \
  --timeout 7200
```

## Acceptance Reminder

Do not mark the feature accepted because code exists. Acceptance requires:

- completed production Linux RTX 5090 run;
- baseline vs candidate comparison;
- DB parity and artifact completeness;
- reviewer-labeled accuracy for posture, hand raising, torso/head orientation,
  and attention support;
- kinematics latency and throughput metrics;
- contradiction/override monitoring evidence;
- live compatibility proof for bounded history and append-only evidence.
