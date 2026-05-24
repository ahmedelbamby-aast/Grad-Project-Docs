# Quickstart: Fix RTSP Camera Feed Connection

**Branch**: `002-fix-rtsp-camera-feed` | **Date**: 2026-03-04

This guide describes how to set up and verify the RTSP camera feed fix locally.

---

## Prerequisites

| Dependency | Version | Check Command |
|------------|---------|---------------|
| Python | 3.10+ | `python --version` |
| Node.js | 18+ | `node --version` |
| Docker + Compose | Latest | `docker compose version` |
| PostgreSQL | 16 (via Docker) | Runs in `docker compose` |
| Redis | 7 (via Docker) | Runs in `docker compose` |
| go2rtc | Latest (via Docker) | Runs in `docker compose` |

---

## 1. Environment Setup

### 1.1 Start Infrastructure Services

```bash
docker compose -f docker-compose.dev.yml up -d postgres redis go2rtc
```

Verify all three services are healthy:
```bash
docker compose -f docker-compose.dev.yml ps
```

### 1.2 Backend Setup

```bash
cd backend

# Create virtual environment (if not done)
python -m venv ../.venv
source ../.venv/bin/activate  # Linux/Mac
# or: ..\.venv\Scripts\Activate.ps1  # PowerShell

# Install dependencies
pip install -r requirements.txt
pip install httpx  # NEW: required for go2rtc client

# Environment variables — create/update .env file
cat >> .env << 'EOF'
FIELD_ENCRYPTION_KEY=<generate-with-command-below>
GO2RTC_API_URL=http://localhost:1984
EOF

# Generate encryption key
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
# Copy the output and paste into .env as FIELD_ENCRYPTION_KEY value

# Run migrations
python manage.py migrate

# Start Django dev server
python manage.py runserver
```

### 1.3 Start Celery Worker + Beat

```bash
# In a separate terminal (with venv activated)
cd backend
celery -A config worker -l info
```

```bash
# In another terminal
cd backend
celery -A config beat -l info
```

The `stream_health_check` task runs every 15 seconds and monitors all connected camera streams.

### 1.4 Frontend Setup

```bash
cd frontend
npm install
npm run dev
```

---

## 2. Verify the Fix

### 2.1 Test with RTSP Simulator (no real camera needed)

If you don't have a real RTSP camera, use an RTSP test stream:

```bash
# Option A: Use a public test stream
# (Note: may be slow or unavailable)
rtsp://rtsp.stream/pattern

# Option B: Run a local RTSP simulator with FFmpeg
docker run --rm -d --name rtsp-sim -p 8554:8554 \
  bluenviron/mediamtx:latest
# Then use: rtsp://host.docker.internal:8554/mystream
# (You'll need to push a test stream to it with ffmpeg)
```

### 2.2 Connect a Camera

1. Log in as an instructor at `http://localhost:5173`.
2. Navigate to **Camera List** page.
3. Click **Add Camera**.
4. Enter a name and the RTSP URL.
5. Click **Save**.
6. Click **Connect** on the camera card.
7. Watch for:
   - Multi-stage progress indicator: "Validating stream…" → "Registering…" → "Connecting video…" → "Live"
   - Status badge turns green: **Connected**
   - Success toast with "**View Live Feed →**" link
8. Click the toast link or navigate to **Camera Feed** page.
9. Verify the live video feed is visible.

### 2.3 Test Error Handling

- **Invalid URL**: Create a camera with `rtsp://192.168.1.254/stream` (unreachable). Click Connect. Expect: "Camera Unreachable" error banner on the tile + toast.
- **Bad credentials**: Use `rtsp://wrong:wrong@real-camera-ip/stream`. Click Connect. Expect: "Authentication Failed" error.
- **go2rtc down**: Stop go2rtc (`docker compose stop go2rtc`). Try connecting. Expect: "Streaming Service Unavailable" error.

### 2.4 Test Real-Time Status

1. Open the Camera Feed page in **two browser tabs**.
2. Connect a camera from Tab 1.
3. Verify Tab 2 updates the status badge within 2 seconds.
4. Disconnect from Tab 1 — Tab 2 should update immediately.

### 2.5 Test Auto-Reconnection

1. Connect a camera successfully.
2. Restart go2rtc: `docker compose restart go2rtc`.
3. Wait up to 30 seconds (health-check interval + retry time).
4. Verify the camera auto-reconnects — status goes `reconnecting → connected`.

---

## 3. Run Tests

### Backend

```bash
cd backend
pytest tests/unit/cameras/ -v --tb=short
pytest tests/integration/cameras/ -v --tb=short
pytest -v --tb=short  # full suite
```

### Frontend

```bash
cd frontend
npx vitest run --reporter=verbose
npx playwright test tests/e2e/camera-connect.spec.ts
```

---

## 4. New Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `FIELD_ENCRYPTION_KEY` | Yes | — | Fernet encryption key for RTSP URLs. Generate with `Fernet.generate_key()`. |
| `GO2RTC_API_URL` | No | `http://localhost:1984` | go2rtc REST API base URL (Docker: `http://go2rtc:1984`). |

---

## 5. New Python Dependencies

| Package | Purpose |
|---------|---------|
| `httpx` | HTTP client for go2rtc REST API communication (sync + async support) |

`cryptography` is already transitively installed via `channels`.

---

## 6. Configuration Constants

Defined in `backend/apps/cameras/constants.py`:

| Constant | Default | Description |
|----------|---------|-------------|
| `RTSP_PROBE_TIMEOUT` | `10` (seconds) | Total timeout for the three-stage RTSP probe |
| `HEALTH_CHECK_INTERVAL` | `15` (seconds) | Celery beat interval for stream health monitoring |
| `MAX_RECONNECT_RETRIES` | `3` | Auto-reconnection retry limit |
| `RECONNECT_BASE_DELAY` | `2` (seconds) | Exponential backoff base (2s → 4s → 8s) |
| `CONNECTION_EVENT_RETENTION_DAYS` | `90` | Days to retain ConnectionEvent logs |
| `MAX_CAMERAS_PER_INSTRUCTOR` | `8` | Per-instructor camera limit |
| `BULK_FAILURE_WINDOW` | `10` (seconds) | Time window for consolidating failure notifications |
| `BULK_FAILURE_THRESHOLD` | `3` | Number of failures in window to trigger consolidated notification |

---

## Resetting a Stuck Celery Worker

Use this runbook any time a Celery worker becomes unresponsive, a task hangs
indefinitely, or camera health-check tasks stop firing.

### Step 1 — Diagnose

```powershell
# Is a worker running at all?
Get-Process python | Where-Object { $_.CommandLine -like "*celery*" }

# How many messages are queued in the broker?
.venv\Scripts\python.exe -c "
import redis; r = redis.Redis.from_url('redis://localhost:6379/1')
print('Queued tasks:', r.llen('celery'))
"
```

### Step 2 — Graceful restart (try first)

Stop the terminal running the worker with **Ctrl+C**, then restart:

```powershell
# In the backend directory, with the venv active:
.venv\Scripts\celery.exe -A config worker -l INFO --pool=solo
```

> **Windows note:** always use `--pool=solo`. The default `prefork` pool
> does not work on Windows and will silently fail to process tasks.

### Step 3 — Force-kill if Ctrl+C does not respond

```powershell
Get-Process python | Where-Object MainWindowTitle -eq "" | Stop-Process -Force
```

Then restart as in Step 2.

### Step 4 — Purge stale tasks from the broker queue

```powershell
cd backend
.venv\Scripts\celery.exe -A config purge -f
```

### Step 5 — Full system reset (nuclear option)

```powershell
cd backend
.venv\Scripts\python.exe manage.py flush_video_jobs --yes
```

Then restart the worker (Step 2) and clear browser localStorage:
open DevTools → Application → Local Storage → delete `exam_monitor_recent_jobs`.

### Quick-diagnosis table

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Job stays `queued` forever | Worker not running | Step 2 |
| Job stays `processing` at 0% | Worker crashed mid-task | Steps 3 → 2 → 5 |
| Job stays `processing` at N% | Task deadlock or DB lock | Step 3 → 2 |
| `IntegrityError` in worker log | Stale DB state from previous run | Step 5 |
| Browser still shows old jobs after DB reset | Stale localStorage | Clear `exam_monitor_recent_jobs` key |
| `redis.exceptions.ConnectionError` in worker | Redis not running | Start Redis, then Step 2 |

