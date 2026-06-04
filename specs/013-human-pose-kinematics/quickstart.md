# Quickstart: Human Pose Kinematics Layer

**Last updated:** 2026-06-04

## Preconditions

- PostgreSQL is the only accepted database backend.
- RTMPose remains the authoritative pose keypoint source.
- The feature is off by default until implementation, tests, and production
  benchmark evidence pass.
- All thresholds are read from `.env`/Django settings; no service logic may
  hardcode override gates.

## Planned Configuration

Add these keys to the relevant `.env` and config files during implementation:

```bash
POSE_KINEMATICS_ENABLED=0
POSE_KINEMATICS_HISTORY_SECONDS=5
POSE_KINEMATICS_OVERRIDE_MARGIN=0.15
POSE_KINEMATICS_OVERRIDE_MIN_FRAMES=3
POSE_KINEMATICS_ARTIFACTS_ENABLED=1
POSE_KINEMATICS_TELEMETRY_ENABLED=1
```

Implementation must add bounded parsing in `backend/config/settings/base.py`.
Future production enablement should change only env/config values, not service
logic.

## Local Validation Shape

Run narrow tests first after implementation tasks exist:

```powershell
cd E:\grad_project
.\backend\.venv\Scripts\python.exe -m pytest backend\tests\unit -q --tb=short
.\backend\.venv\Scripts\python.exe -m pytest backend\tests\contract -q --tb=short
```

Run integration tests against PostgreSQL semantics, not SQLite:

```powershell
cd E:\grad_project\backend
.\.venv\Scripts\python.exe -m pytest tests\integration -q --tb=short
```

## Production Benchmark Shape

After implementation and deployment, use the same production video used by the
optimization program:

```bash
cd /home/bamby/grad_project
bash tools/prod/prod_watch_benchmark_metrics.sh --latest --interval 10 --clear
```

The candidate replay should use:

```bash
cd /home/bamby/grad_project/backend
PYTHONPATH=/home/bamby/grad_project/backend \
DJANGO_SETTINGS_MODULE=config.settings \
APP_ENV=prod \
python manage.py runtime_ingest_video \
  --video "/home/bamby/grad_project/Raw Data/Diverse Classroom Enviroments/combined.mp4" \
  --actor-username bamby \
  --runtime-profile offline \
  --pipeline-mode crop_frame \
  --benchmark-evidence \
  --replay-key "<pose-kinematics-candidate-key>" \
  --replay-policy new-attempt \
  --wait \
  --timeout-seconds 7200
```

Collect benchmark metrics with the existing production collector and append the
comparison table required by [plan.md](plan.md).

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
