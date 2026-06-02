# Infrastructure & Deployment Tuning

**Last updated:** 2026-05-29

## Production Readiness & Performance Optimization

**Date**: 2026-05-29  
**Target**: Linux RTX 5090 Production (`0.tcp.eu.ngrok.io`)  
**Status**: Operational Improvement Guide

---

## Executive Summary

**6 Infrastructure Areas** need tuning for production stability & performance:

| Area | Current State | Priority | Expected Gain |
|------|---------------|----------|---------------|
| **Celery Worker Tuning** | Default concurrency | 🔴 HIGH | 3–5× throughput |
| **Triton Server Config** | Baseline tuning | 🔴 HIGH | 2–3× throughput |
| **PostgreSQL Tuning** | Development defaults | 🟡 MEDIUM | 30–50% faster |
| **Redis Tuning** | Baseline memory | 🟡 MEDIUM | Better eviction |
| **Systemd Services** | Basic setup | 🟡 MEDIUM | Reliability |
| **Monitoring Alerts** | None | 🟢 LOW | Early warning |

---

## 1. CELERY WORKER TUNING (Linux Production)

### 🔴 HIGH PRIORITY

#### 1.1 Worker Pool Configuration

**Current** (`backend/config/celery.py`):
```python
configured_pool = os.getenv('CELERY_WORKER_POOL', default_pool)  # 'prefork' on Linux
configured_concurrency = int(os.getenv('CELERY_WORKER_CONCURRENCY', str(default_concurrency)))  # CPU count
```

**Problem**:
- Default: 6 workers on 6-core CPU (100% CPU allocation)
- GPU-bound tasks don't need all CPU threads (RTX 5090 limits inference)
- Offline video processing needs more CPU; live streaming needs less

**Solution: Multi-Queue with Adaptive Worker Pools**

```bash
# Start 3 separate worker processes on production

# Worker 1: Offline video processing (CPU-heavy, 4 concurrency)
celery -A config worker \
  -Q offline_control,offline_person,offline_pose,offline_behavior \
  -c 4 \
  -n offline_worker@%h \
  -l info \
  --max-tasks-per-child=100

# Worker 2: Live streaming (I/O-heavy, 2 concurrency, use threads)
celery -A config worker \
  -Q live_control,live_person,live_pose,live_behavior \
  -c 2 \
  -n live_worker@%h \
  -l info \
  --pool=prefork \
  --max-tasks-per-child=50

# Worker 3: Background tasks (health checks, cleanup, 2 concurrency)
celery -A config worker \
  -Q default \
  -c 2 \
  -n default_worker@%h \
  -l info \
  --max-tasks-per-child=200
```

**Implementation**:
```bash
# Create systemd services (agents.md line 101-145)
# /etc/systemd/user/celery-offline-worker.service
# /etc/systemd/user/celery-live-worker.service
# /etc/systemd/user/celery-default-worker.service

# Or use supervisor:
[program:celery-offline-worker]
command=celery -A config worker -Q offline_control,offline_person -c 4 -n offline_worker@%%h
autostart=true
autorestart=true
stderr_logfile=/home/bamby/logs/celery-offline.err.log
stdout_logfile=/home/bamby/logs/celery-offline.out.log

[program:celery-live-worker]
command=celery -A config worker -Q live_control,live_person -c 2 -n live_worker@%%h
autostart=true
autorestart=true
stderr_logfile=/home/bamby/logs/celery-live.err.log
stdout_logfile=/home/bamby/logs/celery-live.out.log
```

**Environment Configuration**:
```bash
# backend/.env (production)

# Queue routing
CELERY_DEFAULT_QUEUE=default
CELERY_VIDEO_INFERENCE_QUEUE=offline_control

# Worker concurrency tuning
CELERY_WORKER_POOL=prefork
CELERY_WORKER_CONCURRENCY=4  # Per offline worker
CELERY_GPU_CONCURRENCY_CAP=2  # Cap GPU-bound tasks

# Task prefetch and acknowledgment
CELERY_WORKER_PREFETCH_MULTIPLIER=1  # Get 1 task at a time (fairness)
CELERY_TASK_ACKS_LATE=true  # Acknowledge after completion
CELERY_TASK_ACKS_ON_FAILURE_OR_TIMEOUT=true

# Timeouts
CELERY_TASK_TIME_LIMIT=600  # Hard limit 10 min
CELERY_TASK_SOFT_TIME_LIMIT=300  # Soft limit 5 min
CELERY_TASK_TRACK_STARTED=true  # Report task start
```

**Expected Improvement**:
- **Offline processing**: 3–5× better throughput (dedicated 4-worker pool)
- **Live streaming**: Stable latency (dedicated low-concurrency pool)
- **Background tasks**: No starvation (separate queue)

---

#### 1.2 Task Routing & Queue Isolation

**Problem**: All tasks compete on same queue; slow offline jobs starve live tasks.

**Solution: Explicit Task Routing**
```python
# backend/config/celery.py (update task_routes)

app.conf.task_routes = {
    # Offline (high CPU, low latency requirement)
    'apps.video_analysis.tasks.process_video_upload': {
        'queue': 'offline_control',
        'routing_key': 'offline.control',
    },
    'apps.pipeline.tasks.dispatch_offline_*': {
        'queue': 'offline_person',
        'routing_key': 'offline.person',
    },
    
    # Live (low CPU, high latency requirement)
    'apps.video_analysis.tasks.run_live_stream_inference': {
        'queue': 'live_control',
        'routing_key': 'live.control',
    },
    'apps.pipeline.tasks.dispatch_live_*': {
        'queue': 'live_person',
        'routing_key': 'live.person',
    },
    
    # Background (periodic, low priority)
    'apps.exports.tasks.cleanup_expired_exports': {
        'queue': 'default',
    },
    'apps.cameras.tasks.stream_health_check': {
        'queue': 'default',
    },
}
```

**Expected Improvement**: **Live stream latency stable** even during offline processing

---

#### 1.3 Task Time Limits & Soft Timeouts

**Current**:
```python
task_time_limit=300,  # 5 min hard timeout
task_soft_time_limit=300,  # Same as hard limit (bad)
```

**Problem**: No graceful shutdown opportunity; task dies abruptly.

**Solution: Staggered Timeouts**
```python
# backend/config/celery.py

# Global defaults
app.conf.task_time_limit = 600  # Hard limit: 10 min (kill task)
app.conf.task_soft_time_limit = 300  # Soft limit: 5 min (signal SoftTimeLimitExceeded)

# Per-task overrides
app.task(
    bind=True,
    time_limit=600,
    soft_time_limit=300,
    autoretry_for=(SoftTimeLimitExceeded,),
)
def long_running_task(self):
    try:
        # 5-minute warning
        do_expensive_work()
    except SoftTimeLimitExceeded:
        # Gracefully save state, exit
        logger.warning('Task exceeded soft timeout; saving progress and exiting')
        return {'status': 'partial', 'progress': current_progress}
```

**Expected Improvement**: **Graceful task shutdown**, no abrupt process kills

---

### 🟡 MEDIUM PRIORITY

#### 1.4 Dead Letter Queue (DLQ) Handling

**Problem**: Failed tasks are lost; no replay mechanism.

**Solution: Implement DLQ**
```python
# backend/config/celery.py

# Configure DLQ for each queue
app.conf.task_queues = (
    Queue('default'),
    Queue('offline_control'),
    Queue('offline_person'),
    Queue('default.dlq'),  # Dead letter
    Queue('offline_control.dlq'),
)

# Retry failed tasks to DLQ
@app.task(bind=True, autoretry_for=(Exception,), retry_kwargs={'max_retries': 3})
def process_video_upload(self):
    try:
        # ... processing
    except Exception as exc:
        if self.request.retries >= 3:
            # Send to DLQ for manual investigation
            logger.error(f'Task {self.request.id} exhausted retries; sending to DLQ')
            task_id = self.request.id
            # Re-queue to DLQ
            from config.celery import app
            app.send_task(
                'apps.video_analysis.tasks.handle_dlq_task',
                args=[task_id, str(exc)],
                queue='default.dlq',
            )
        raise
```

**Expected Improvement**: **Visibility into failures**, ability to replay failed tasks

---

## 2. TRITON SERVER OPTIMIZATION

### 🔴 HIGH PRIORITY

#### 2.1 Triton Model Repository Tuning

**Current** (`agents.md` line 207):
```bash
# Baseline Triton config
/home/bamby/grad_project/backend/models/triton_repository_cuda12/
```

**Problem**: Models load with default settings; no batching or optimization.

**Solution: Optimize Model Config**
```protobuf
# backend/models/triton_repository_cuda12/person_detection_onnx/config.pbtxt

name: "person_detection_onnx"
platform: "onnxruntime_onnx"
max_batch_size: 8  # Enable batching (2x throughput)

input {
  name: "images"
  data_type: TYPE_FP32
  dims: [640, 640, 3]
}

output {
  name: "output"
  data_type: TYPE_FP32
}

# Model optimization
optimization {
  priority: PRIORITY_DEFAULT
  execution_accelerators {
    gpu_execution_accelerator {
      # Empty: use default GPU acceleration
    }
  }
}

# Ensemble batching
dynamic_batching {
  preferred_batch_size: [4, 8]  # Batch requests into 4 or 8
  max_queue_delay_microseconds: 100000  # Wait up to 100ms for full batch
  default_queue_policy {
    timeouts {
      max: 60000000  # 60 sec timeout
    }
  }
}

# Instance groups (GPU placement)
instance_group [
  {
    name: "gpu_0"
    kind: KIND_GPU
    gpus: [0]  # Use GPU 0
    count: 2  # 2 instances per GPU (if GPU memory allows)
  }
]
```

**Expected Improvement**: **2–3× throughput** via dynamic batching

---

#### 2.2 Triton Inference Server Launch Tuning

**Current** (`agents.md` line 103):
```bash
tritonserver --backend-directory=/home/bamby/services/triton_build_r2502/tritonserver/install/backends
```

**Problem**: No optimization flags; default memory/timing settings.

**Solution: Optimized Launch**
```bash
# backend/infra/systemd/triton-server.service (update)

[Service]
ExecStart=/home/bamby/services/triton_build_r2502/tritonserver/install/bin/tritonserver \
  --backend-directory=/home/bamby/services/triton_build_r2502/tritonserver/install/backends \
  --model-repository=/home/bamby/grad_project/backend/models/triton_repository_cuda12 \
  --model-control-mode=explicit \
  --load-model=person_detection_onnx \
  --load-model=standing_sitting_onnx \
  --load-model=gaze_models \
  --grpc-port=39100 \
  --http-port=39101 \
  --metrics-port=39102 \
  --backend-config=onnxruntime,default-max-batch-size=8 \
  --cuda-memory=8000 \
  --cuda-compute-capability=8.9 \
  --cuda-graphs=true \
  --rate-limiter-config=rate_limiter.json \
  --log-verbose=false
```

**Rate Limiter Config** (`rate_limiter.json`):
```json
{
  "resources": [
    {
      "name": "person_detection_onnx",
      "global": {
        "count": 100
      }
    },
    {
      "name": "standing_sitting_onnx",
      "global": {
        "count": 100
      }
    }
  ]
}
```

**Expected Improvement**: **Better GPU memory utilization**, explicit model loading

---

#### 2.3 Triton Request Routing & Timeout Tuning

**Current** (`backend/apps/pipeline/inference_runtime.py`):
```python
TRITON_TIMEOUT_MS = 1500  # 1.5 sec per request
TRITON_MAX_TIMEOUT_MS = 5000  # Max 5 sec
```

**Problem**: Too aggressive for batched requests.

**Solution: Adaptive Timeout**
```python
# backend/core/configuration.py (update)

def resolve_triton_timeout_ms() -> int:
    """Adaptive Triton timeout based on batch size and GPU load."""
    batch_size = int(os.getenv('PYRAMID_INFERENCE_BATCH_SIZE', '1'))
    
    # Base timeout + 500ms per batch item
    base = 1500
    per_batch = 500
    timeout = base + (batch_size - 1) * per_batch
    
    # Cap at reasonable maximum
    return min(timeout, 10000)  # Max 10 sec
```

**Expected Improvement**: **Fewer timeouts**, better handling of batch requests

---

### 🟡 MEDIUM PRIORITY

#### 2.4 Triton Health Check Optimization

**Current** (`backend/apps/health/views.py` line 59):
```python
service = ModelServingHealthService(
    triton_url=triton_url,
    model_names=model_names,
    timeout_s=5.0,  # Per-model timeout
)
```

**Problem**: Health checks take 5s × N models; block monitoring.

**Solution: Cached Health Check**
```python
# backend/apps/health/services.py (update)
from django.core.cache import cache

class CachedModelServingHealthService:
    def check(self):
        """Check Triton health with caching."""
        cache_key = 'triton_health_check'
        cached_result = cache.get(cache_key)
        
        if cached_result:
            return cached_result
        
        # Do actual check (expensive)
        result = self._perform_check()
        
        # Cache for 10 seconds
        cache.set(cache_key, result, timeout=10)
        return result
```

**Expected Improvement**: **Health checks complete in <100ms** (cached)

---

## 3. POSTGRESQL TUNING

### 🟡 MEDIUM PRIORITY

#### 3.1 PostgreSQL Configuration

**Current**: Docker defaults or basic setup.

**Solution: Production PostgreSQL Config**
```ini
# /etc/postgresql/16/main/postgresql.conf

# Memory
shared_buffers = 4GB              # 25% of RAM (8GB total)
effective_cache_size = 12GB       # 75% of RAM
work_mem = 100MB                  # Per-operation memory
maintenance_work_mem = 1GB        # For CREATE INDEX, VACUUM

# Connections
max_connections = 200
max_prepared_transactions = 100

# WAL (Write-Ahead Log)
wal_buffers = 16MB
checkpoint_timeout = 15min
checkpoint_completion_target = 0.9
wal_level = replica              # For backups

# Query Tuning
random_page_cost = 1.1            # SSD penalty (vs 4.0 default)
effective_io_concurrency = 200    # Concurrent I/O requests
default_statistics_target = 100   # Query planning accuracy

# Logging
log_statement = 'mod'             # Log DDL + DML
log_duration = on
log_min_duration_statement = 1000 # Log queries >1 sec
log_connections = on
log_disconnections = on

# Performance
jit = on                          # Just-in-time compilation
jit_above_cost = 100000
```

**Apply Changes**:
```bash
sudo systemctl restart postgresql
# Verify
psql -c "SHOW shared_buffers; SHOW effective_cache_size;"
```

**Expected Improvement**: **30–50% faster queries**, better plan selection

---

#### 3.2 Index Creation & Maintenance

**Solution: Automate Index Analysis**
```python
# backend/scripts/audit_db_indexes.py (NEW)
import os
import django
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings.production')
django.setup()

from django.db import connection

def find_missing_indexes():
    """Identify columns frequently filtered without indexes."""
    with connection.cursor() as cursor:
        cursor.execute("""
            SELECT
                schemaname, tablename, attname, n_distinct,
                null_frac, avg_width
            FROM pg_stats
            WHERE schemaname = 'public'
            AND n_distinct > 100  -- High cardinality
            AND null_frac < 0.5  -- Used regularly
            AND attname NOT IN (
                SELECT attname FROM pg_indexes
                WHERE pg_indexes.tablename = pg_stats.tablename
            )
            ORDER BY n_distinct DESC
            LIMIT 20;
        """)
        return cursor.fetchall()

if __name__ == '__main__':
    missing = find_missing_indexes()
    for table, col, cardinality, null_pct, width in missing:
        print(f"CREATE INDEX idx_{table}_{col} ON {table}({col});")
```

**Expected Improvement**: **Faster filtered queries**, automatic index discovery

---

## 4. REDIS OPTIMIZATION

### 🟡 MEDIUM PRIORITY

#### 4.1 Redis Memory Management

**Current**: No explicit memory limits or eviction policy.

**Solution: Configure Redis**
```bash
# /etc/redis/redis.conf (or /home/bamby/.redis/redis.conf)

# Memory
maxmemory 4gb                  # Cap at 4GB
maxmemory-policy allkeys-lru   # Evict oldest keys when full

# Persistence
save 900 1                     # Save after 900s if 1+ key changed
save 300 10                    # Save after 300s if 10+ keys changed
save 60 10000                  # Save after 60s if 10K+ keys changed

# Performance
appendonly no                  # Disable AOF (fast, less durable)
tcp-backlog 65535              # Handle bursts

# Connection pooling
timeout 0                      # Never timeout idle connections
```

**Expected Improvement**: **Bounded memory usage**, no unbounded growth

---

#### 4.2 Redis Monitoring

**Solution: Real-time Monitoring**
```bash
# Monitor Redis usage
redis-cli INFO memory
redis-cli INFO stats | grep hits

# Expected:
# keyspace_hits: high
# keyspace_misses: low
# hit_rate = hits / (hits + misses) should be >80%
```

**Expected Improvement**: **Visibility into cache performance**

---

## 5. SYSTEMD SERVICE TUNING

### 🟡 MEDIUM PRIORITY

#### 5.1 Service Dependency & Ordering

**Current**: Basic service definitions.

**Solution: Production-Ready Services**
```ini
# /etc/systemd/user/postgres.service (example)
[Unit]
Description=PostgreSQL Database Server
Wants=network-online.target
After=network-online.target

[Service]
Type=notify
ExecStart=/usr/lib/postgresql/16/bin/postgres ...
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=10

# Resource limits
MemoryMax=8G
CPUQuota=50%
IOWeight=500

[Install]
WantedBy=multi-user.target

# Similar for Redis, Triton, Django, Celery workers
```

**Service Startup Order**:
```
postgresql.service ↓
├─ redis.service ↓
├─ triton-server.service ↓
└─ django.service ↓
   └─ celery-workers.service
```

**Expected Improvement**: **Reliable startup order**, resource isolation

---

#### 5.2 Service Restart & Recovery

**Solution: Automatic Recovery**
```ini
[Service]
Restart=always
RestartSec=5                  # Restart after 5 sec
StartLimitInterval=60         # Check limit in 60 sec window
StartLimitBurst=3            # Max 3 restarts in window

# If fails 3× in 60s, stop and alert
OnFailure=alert-restart@%i.service
```

**Expected Improvement**: **Self-healing services**, alerting on repeated failures

---

## 6. MONITORING & ALERTING

### 🟢 LOW PRIORITY (Enables Troubleshooting)

#### 6.1 Prometheus Metrics Export

**Solution: Instrument Services**
```python
# backend/core/metrics.py (add to existing)
from prometheus_client import Counter, Histogram, Gauge, generate_latest

# Custom metrics
JOB_DURATION = Histogram(
    'job_duration_seconds',
    'Video job duration',
    buckets=(10, 60, 300, 600),
)

ACTIVE_WORKERS = Gauge(
    'celery_active_workers',
    'Number of active Celery workers',
)

QUEUE_SIZE = Gauge(
    'celery_queue_size',
    'Messages in Celery queue',
    ['queue_name'],
)

# Expose via /metrics endpoint
from django.http import HttpResponse
def prometheus_metrics(request):
    return HttpResponse(generate_latest(), content_type='text/plain')
```

**Expected Benefit**: **Metrics for Prometheus/Grafana** dashboards

---

#### 6.2 Alert Rules

**Solution: Prometheus Alert Rules**
```yaml
# prometheus/rules/exam_monitor.yml
groups:
  - name: exam_monitor
    interval: 30s
    rules:
      - alert: CeleryQueueBacklog
        expr: celery_queue_size > 100
        for: 5m
        annotations:
          summary: "Celery queue backlog detected"
      
      - alert: TritonModelNotReady
        expr: triton_model_ready == 0
        for: 2m
        annotations:
          summary: "Triton model failed to load"
      
      - alert: PostgreSQLSlowQueries
        expr: rate(pg_stat_statements_mean_time[5m]) > 1000
        for: 5m
        annotations:
          summary: "Slow PostgreSQL queries detected"
```

**Expected Benefit**: **Early warning** before system degradation

---

## Implementation Roadmap

### Week 1: Celery & Task Routing
- [ ] Create 3 separate worker systemd services
- [ ] Implement task routing by queue
- [ ] Test offline/live queue isolation
- [ ] Benchmark throughput improvement

### Week 2: Triton Optimization
- [ ] Update model configs for batching
- [ ] Implement dynamic batching
- [ ] Optimize Triton launch script
- [ ] Benchmark inference throughput

### Week 3: Database Tuning
- [ ] Run PostgreSQL configuration audit
- [ ] Apply recommended settings
- [ ] Create missing indexes
- [ ] Benchmark query performance

### Week 4: Monitoring & Stability
- [ ] Set up Prometheus metrics
- [ ] Create alert rules
- [ ] Configure systemd service recovery
- [ ] Test failover scenarios

---

## Quick Production Wins

✅ **Immediate Actions** (Can start today):

1. **Reduce Triton timeout** (5 min)
   - Change `TRITON_TIMEOUT_MS` from 1500 to 3000
   - Impact: No timeout errors on batched requests

2. **Enable Triton dynamic batching** (15 min)
   - Update model configs
   - Impact: 2–3× throughput

3. **Separate offline/live workers** (30 min)
   - Create second Celery worker process
   - Impact: Live streams don't stall during uploads

4. **Add PostgreSQL indexes** (1 hour)
   - Add `db_index=True` to status, frame_number
   - Impact: 20–30% faster queries

5. **Configure Redis eviction** (10 min)
   - Set `maxmemory-policy allkeys-lru`
   - Impact: No Redis OOM, bounded memory

---

## Success Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| **Celery Queue Latency** | <100ms p50 | Celery task monitor |
| **Triton Inference Time** | <500ms p95 | Prometheus histogram |
| **PostgreSQL Query Time** | <100ms p95 | Slow query log |
| **Redis Hit Rate** | >80% | redis-cli INFO |
| **Worker Uptime** | >99.9% | systemd journal |

---

## References

- `agents.md`: Production deployment topology
- `backend/config/celery.py`: Current Celery config
- `backend/infra/systemd/`: Service definitions
- `backend/apps/pipeline/inference_runtime.py`: Triton client
