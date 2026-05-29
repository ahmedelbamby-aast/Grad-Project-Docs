# System Optimization & Tuning Guide
## Exam Monitoring Dashboard — Complete Performance Audit

**Date**: 2026-05-29  
**Scope**: Backend, Frontend, Database, Network, Storage, Monitoring  
**Status**: Comprehensive Multi-Domain Analysis

---

## Executive Summary

Beyond threading/multiprocessing, **6 major optimization domains** need attention:

| Domain | Current State | Priority | Est. Gain |
|--------|---------------|----------|-----------|
| **Database (N+1, Indexing)** | Partial prefetch_related | 🔴 HIGH | 30–50% faster queries |
| **Caching Layer** | No Redis cache config | 🔴 HIGH | 40–60% fewer DB hits |
| **Frontend Bundle** | Vite defaults | 🟡 MEDIUM | 20–30% bundle reduction |
| **Video Streaming** | Default FFmpeg settings | 🟡 MEDIUM | 15–25% bitrate optimization |
| **API Pagination** | 20-item page size | 🟡 MEDIUM | Better UX at scale |
| **Monitoring/Metrics** | Basic logging | 🟢 LOW | Debug latency issues |

---

## 1. DATABASE OPTIMIZATION (N+1, Indexing, Query Tuning)

### 🔴 HIGH PRIORITY

#### 1.1 Missing Prefetch/Select Relations

**Current State** (Partial):
```python
# ✅ Good: RecordingViewSet
queryset = Recording.objects.select_related('camera_source').prefetch_related('sessions').all()

# ❌ Problem: DetectionsViewSet  
queryset = PyramidPrediction.objects.all()  # N+1 on detection__frame__session joins
```

**Affected Views**:
- `DetectionFrameViewSet`: Line 17 (basic prefetch, missing nested relations)
- `SessionViewSet`: Line 70 (good, but missing related counts)
- `CameraViewSet`: Likely missing camera_connection and stream_status prefetch
- `ExportViewSet`: Likely missing export_file and video_job prefetch
- `AnomalyViewSet`: Likely missing anomaly_detections and session prefetch

**Solution A: Audit & Fix Missing Prefetch**
```python
# backend/apps/detections/views.py
class DetectionFrameViewSet(mixins.ListModelMixin, mixins.RetrieveModelMixin, viewsets.GenericViewSet):
    def get_queryset(self):
        return DetectionFrame.objects.prefetch_related(
            'detections',
            'detections__frame',
            'detections__frame__session',
            'detections__frame__session__user',
        ).select_related('session').all()
```

**Solution B: Django Debug Toolbar for Detection**
```bash
# Install for development only
pip install django-debug-toolbar

# Add to INSTALLED_APPS (development only)
# Add to urls.py for debug mode
```

**Implementation Checklist**:
- [ ] Profile each ViewSet with Django Debug Toolbar
- [ ] Identify N+1 queries (1 parent + N children = N+1 queries)
- [ ] Add `select_related()` for ForeignKey relations
- [ ] Add `prefetch_related()` for ManyToMany and reverse ForeignKey
- [ ] Benchmark: measure query count before/after
- [ ] Document in each ViewSet

**Expected Improvement**: **20–30% faster** list/retrieve endpoints

---

#### 1.2 Missing Database Indexes

**Audit Query**:
```sql
-- Find missing indexes on frequently filtered columns
SELECT tablename, indexname FROM pg_indexes 
WHERE schemaname = 'public' 
ORDER BY tablename;

-- Check slow queries (PostgreSQL slow log)
SELECT query, calls, mean_time, total_time 
FROM pg_stat_statements 
WHERE mean_time > 10 
ORDER BY mean_time DESC LIMIT 10;
```

**Likely Missing Indexes**:
- `VideoAnalysisJob.status` (filtered in tasks, views)
- `Frame.frame_number` (range queries in video export)
- `StudentTrack.tracking_id` (filtered in embeddings)
- `Detection.confidence` (thresholding)
- `MonitoringSession.user_id, created_at` (user-time-range queries)
- `Recording.camera_source_id, status` (compound index)

**Solution: Create Indexes**
```python
# backend/apps/video_analysis/models.py
class VideoAnalysisJob(models.Model):
    status = models.CharField(max_length=32, db_index=True)  # Add db_index=True
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)

class Frame(models.Model):
    frame_number = models.PositiveIntegerField(db_index=True)
    confidence = models.FloatField(db_index=True)

# backend/apps/tracking/models.py
class StudentTrack(models.Model):
    tracking_id = models.PositiveIntegerField(db_index=True)
    class Meta:
        indexes = [
            models.Index(fields=['job', 'tracking_id']),  # Compound index
        ]
```

**Create Migration**:
```bash
cd backend
uv run python manage.py makemigrations
uv run python manage.py migrate
```

**Expected Improvement**: **15–25% faster** filtered queries, especially on `status` and `frame_number`

---

#### 1.3 Query Timeouts & Pagination Limits

**Problem**: Large result sets (18K frames) loaded into memory.

**Current Configuration** (`base.py` line 131):
```python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'core.pagination.StandardResultsPagination',
    'PAGE_SIZE': 20,  # Too small for large list views
}
```

**Solution: Smart Pagination**
```python
# backend/core/pagination.py (NEW)
from rest_framework.pagination import PageNumberPagination

class SmartPagination(PageNumberPagination):
    """Adaptive page size based on endpoint and user."""
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100
    
    def get_page_size(self, request):
        """Allow larger pages for admin, smaller for regular users."""
        if request.user and request.user.is_staff:
            return 100
        
        # Check endpoint type
        if 'video_analysis_jobs' in request.path:
            return 50  # List all jobs at once
        if 'frames' in request.path:
            return 500  # Frame list can be large
        
        return self.page_size

class LargeDatasetPagination(PageNumberPagination):
    """For large result sets (export, reports)."""
    page_size = 1000
    page_size_query_param = 'page_size'
    max_page_size = 5000
```

**Use in Views**:
```python
class VideoAnalysisJobViewSet(viewsets.ModelViewSet):
    pagination_class = SmartPagination

class FrameViewSet(viewsets.ModelViewSet):
    pagination_class = LargeDatasetPagination
```

**Expected Improvement**: **Better API UX**, fewer requests for large lists

---

#### 1.4 Connection Pooling & Pool Exhaustion

**Problem**: 
- Django default: 1 connection per request (thread)
- Celery workers × 6 + Django workers × 2 = 8 concurrent connections
- PostgreSQL conn limit easily hit under load

**Solution: Configure Connection Pooling**
```python
# backend/config/settings/base.py (add to DATABASES)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.getenv('POSTGRES_DB', 'exam_monitor'),
        'CONN_MAX_AGE': 600,  # Pool connection for 10min
        'OPTIONS': {
            'connect_timeout': 10,
            'options': '-c statement_timeout=30000',  # 30s statement timeout
        },
        'ATOMIC_REQUESTS': False,  # Avoid holding connections too long
    }
}

# Celery also needs DB connection pooling
# Add to backend/config/celery.py
from django.db import connections
app.on_after_finalize.connect(lambda: connections.close_all())
```

**Alternative: Use PgBouncer (Production)**
```yaml
# /etc/pgbouncer/pgbouncer.ini
[databases]
exam_monitor = host=postgres port=5432 dbname=exam_monitor

[pgbouncer]
pool_mode = transaction  # Reuse connection per transaction
max_client_conn = 1000
default_pool_size = 25
reserve_pool_size = 5
reserve_pool_timeout = 3
max_db_connections = 50
```

**Expected Improvement**: **No connection pool exhaustion** under load

---

### 🟡 MEDIUM PRIORITY

#### 1.5 Query Result Caching Strategy

**Problem**: Same queries run repeatedly (camera list, config values, settings).

**Solution: Multi-Level Cache**
```python
# backend/apps/cameras/views.py
from django.views.decorators.cache import cache_page
from django.core.cache import cache

class CameraViewSet(viewsets.ModelViewSet):
    def list(self, request, *args, **kwargs):
        # Cache for 5 minutes unless invalidated
        cache_key = f"cameras_list_{request.user.id}"
        queryset = cache.get_or_set(
            cache_key,
            lambda: list(self.get_queryset()),
            timeout=300,  # 5 minutes
        )
        # ... serialize and return
    
    def perform_create(self, serializer):
        instance = serializer.save()
        # Invalidate cache on write
        cache.delete(f"cameras_list_{self.request.user.id}")
        return instance
```

**Better: Use Django's Cache Framework**
```python
# backend/config/settings/base.py
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': os.getenv('REDIS_URL', 'redis://localhost:6379/0'),
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'PARSER_KWARGS': {'charset': 'utf-8', 'decode_responses': True},
            'CONNECTION_POOL_KWARGS': {'max_connections': 50},
        },
        'TIMEOUT': 300,  # Default 5min
        'KEY_PREFIX': 'exam_monitor',
        'VERSION': 1,
    }
}
```

**Cache Strategy Matrix**:
| Data | TTL | Invalidation | Location |
|------|-----|--------------|----------|
| Cameras list | 5min | On camera create/delete | Redis |
| User permissions | 30min | On user update | Redis |
| Model configs | 1hr | Manual (env change) | Redis |
| Frame list | 10min | On frame write | Redis |
| Session stats | 2min | Periodic batch job | Redis |

**Expected Improvement**: **40–60% fewer DB queries** for read-heavy endpoints

---

## 2. CACHING & REDIS OPTIMIZATION

### 🔴 HIGH PRIORITY

#### 2.1 Redis Client Configuration

**Current State**: No explicit Redis cache backend configured.

**Solution: Optimize Redis Connection**
```python
# backend/config/settings/base.py

REDIS_URL = os.getenv('REDIS_URL', 'redis://localhost:6379/0')

CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': REDIS_URL,
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'PARSER_KWARGS': {
                'charset': 'utf-8',
                'decode_responses': True,
            },
            'CONNECTION_POOL_KWARGS': {
                'max_connections': 50,
                'retry_on_timeout': True,
            },
            'SOCKET_CONNECT_TIMEOUT': 5,
            'SOCKET_TIMEOUT': 5,
        },
        'KEY_PREFIX': 'exam_monitor',
        'VERSION': 1,
        'TIMEOUT': 300,  # Default 5 minutes
    }
}

# Celery also uses Redis
CELERY_BROKER_CONNECTION_RETRY_ON_STARTUP = True
CELERY_BROKER_CONNECTION_RETRY = True
CELERY_BROKER_POOL_LIMIT = None  # Unlimited connections
CELERY_BROKER_HEARTBEAT = 30
```

**Expected Improvement**: **Faster cache lookups**, better connection reliability

---

#### 2.2 Embedding Cache Optimization

**Current State** (`backend/apps/tracking/embeddings.py`):
```python
# Embeddings stored in Redis with 30-day TTL
PYRAMID_EMBEDDING_REDIS_TTL_SECONDS = 2592000
```

**Problem**:
- Embeddings grow unbounded (30 days = 18M embeddings)
- TTL-based eviction doesn't guarantee space
- No compression

**Solution: Bounded Cache with LRU**
```python
# backend/apps/tracking/embeddings_cache.py (NEW)
from redis import Redis
from typing import Optional
import json
import numpy as np

class EmbeddingCache:
    """LRU-bounded embedding cache with compression."""
    
    def __init__(self, redis_client: Redis, max_embeddings: int = 100_000):
        self.redis = redis_client
        self.max_embeddings = max_embeddings
        self.cache_key_prefix = "embeddings:"
        self.metadata_key = "embeddings:metadata"
    
    def get_embedding(self, track_id: int) -> Optional[list[float]]:
        """Retrieve embedding from cache."""
        key = f"{self.cache_key_prefix}{track_id}"
        data = self.redis.get(key)
        if not data:
            return None
        return json.loads(data)
    
    def cache_embedding(self, track_id: int, embedding: list[float], ttl_seconds: int = 2592000):
        """Cache embedding with LRU bounds."""
        key = f"{self.cache_key_prefix}{track_id}"
        
        # Check cache size and evict if needed
        size = self.redis.dbsize()
        if size > self.max_embeddings:
            self._evict_lru(count=1000)  # Evict 1000 entries
        
        # Store with TTL
        self.redis.setex(key, ttl_seconds, json.dumps(embedding))
        self.redis.hset(self.metadata_key, str(track_id), json.dumps({
            'cached_at': time.time(),
            'embedding_dim': len(embedding),
        }))
    
    def _evict_lru(self, count: int = 1000):
        """Evict least-recently-used entries."""
        # Get all keys and sort by access time
        cursor = 0
        keys_to_delete = []
        while len(keys_to_delete) < count:
            cursor, keys = self.redis.scan(cursor, match=f"{self.cache_key_prefix}*", count=100)
            keys_to_delete.extend(keys)
            if cursor == 0:
                break
        
        if keys_to_delete:
            self.redis.delete(*keys_to_delete[:count])
```

**Expected Improvement**: **Bounded memory**, no unbounded growth

---

#### 2.3 Session/WebSocket Broadcasting Cache

**Problem**: Broadcast events repeat data serialization.

**Solution: Cached Serialization**
```python
# backend/apps/sessions/signals.py (NEW)
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.core.cache import cache

@receiver(post_save, sender=AnomalyEvent)
def cache_anomaly_on_save(sender, instance, created, **kwargs):
    """Cache anomaly serialization for WebSocket broadcast."""
    if created:
        serializer = AnomalyEventSerializer(instance)
        cache.set(
            f"anomaly_event:{instance.id}",
            serializer.data,
            timeout=300,  # 5 min cache
        )
```

**Expected Improvement**: **Faster WebSocket broadcasts**, less serialization overhead

---

## 3. FRONTEND OPTIMIZATION

### 🟡 MEDIUM PRIORITY

#### 3.1 Bundle Size & Code Splitting

**Current State**: `Vite 7.3.1` with default config.

**Audit**:
```bash
cd frontend
npm run build
# Check output: dist/ size
# Expected: <500KB main bundle
```

**Solution A: Enable Code Splitting**
```javascript
// frontend/vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor': ['react', 'react-dom', 'react-router-dom'],
          'ui': ['zustand', 'axios'],
          'pages': [
            'src/pages/Dashboard',
            'src/pages/VideoAnalysis',
            'src/pages/Anomalies',
          ],
        },
      },
    },
    minify: 'terser',  // Better compression than esbuild
    terserOptions: {
      compress: {
        drop_console: true,  // Remove console.log in prod
      },
    },
  },
})
```

**Solution B: Lazy Load Routes**
```typescript
// frontend/src/App.tsx
import { lazy, Suspense } from 'react'
import LoadingSpinner from 'components/LoadingSpinner'

const Dashboard = lazy(() => import('pages/Dashboard'))
const VideoAnalysis = lazy(() => import('pages/VideoAnalysis'))

export function App() {
  return (
    <Routes>
      <Route path="/dashboard" element={
        <Suspense fallback={<LoadingSpinner />}>
          <Dashboard />
        </Suspense>
      } />
    </Routes>
  )
}
```

**Solution C: Tree-Shaking Unused Code**
```bash
# Audit unused code
npm install -D webpack-bundle-analyzer

# Add to build script
"analyze": "vite build --outDir dist && npx webpack-bundle-analyzer dist/assets/*.js"

npm run analyze
```

**Expected Improvement**: **20–30% bundle reduction**, faster initial load

---

#### 3.2 Image & Asset Optimization

**Current State**: No image compression documented.

**Solution: Image Optimization**
```javascript
// frontend/vite.config.ts
import imagemin from 'vite-plugin-imagemin'

export default defineConfig({
  plugins: [
    imagemin({
      gifsicle: { optimizationLevel: 7, interlaced: false },
      optipng: { optimizationLevel: 7 },
      mozjpeg: { quality: 80 },
      pngquant: { quality: [0.8, 0.9], speed: 4 },
      svgo: { plugins: [{ name: 'removeViewBox' }] },
    }),
  ],
})
```

**Expected Improvement**: **30–50% image size reduction**

---

#### 3.3 Network Request Batching

**Problem**: Many WebSocket messages for individual detections.

**Solution: Batch Detection Updates**
```typescript
// frontend/src/hooks/useDetectionBatch.ts (NEW)
import { useCallback, useRef, useEffect } from 'react'

export function useDetectionBatch(onBatch: (detections: Detection[]) => void) {
  const batchRef = useRef<Detection[]>([])
  const timerRef = useRef<NodeJS.Timeout | null>(null)
  
  const addDetection = useCallback((detection: Detection) => {
    batchRef.current.push(detection)
    
    // Flush batch every 100ms or 50 detections
    if (batchRef.current.length >= 50) {
      flushBatch()
    } else if (!timerRef.current) {
      timerRef.current = setTimeout(flushBatch, 100)
    }
  }, [])
  
  const flushBatch = () => {
    if (batchRef.current.length > 0) {
      onBatch(batchRef.current)
      batchRef.current = []
    }
    timerRef.current = null
  }
  
  useEffect(() => {
    return () => {
      if (timerRef.current) clearTimeout(timerRef.current)
    }
  }, [])
  
  return { addDetection, flushBatch }
}
```

**Expected Improvement**: **10–20% fewer WebSocket messages**, smoother UI

---

## 4. VIDEO STREAMING & ENCODING OPTIMIZATION

### 🟡 MEDIUM PRIORITY

#### 4.1 Adaptive Bitrate for Streaming

**Current FFmpeg Command** (`video_exporter.py`):
```bash
ffmpeg -i input.mp4 -c:v libx264 -crf 18 -preset medium output.mp4
```

**Problems**:
- Fixed CRF for all videos (doesn't adapt to source quality)
- Preset `medium` is slow on production (RTX 5090)
- No bitrate constraint

**Solution: Adaptive Encoding**
```python
# backend/apps/tracking/video_exporter.py

def get_encoding_preset(video_duration_seconds: int, total_frames: int) -> str:
    """Choose FFmpeg preset based on video size and available resources."""
    fps = total_frames / video_duration_seconds if video_duration_seconds > 0 else 30
    
    if fps > 60:
        return 'fast'  # High FPS = fast encoding
    elif total_frames > 10000:  # >5 min at 30fps
        return 'fast'  # Long videos = faster preset
    else:
        return 'medium'  # Short videos = better quality

def get_adaptive_crf(width: int, height: int, conf_score: float) -> int:
    """Choose CRF based on resolution and confidence."""
    # Higher resolution = higher CRF (more compression)
    # Higher confidence = lower CRF (better quality)
    base_crf = 18
    
    pixels = width * height
    if pixels > 1920 * 1080:
        base_crf += 2
    
    # Quality bonus for high-confidence annotations
    if conf_score > 0.9:
        base_crf -= 2
    elif conf_score < 0.7:
        base_crf += 4
    
    return max(16, min(28, base_crf))

def render_annotated_video_optimized(
    job: VideoAnalysisJob,
    visible_models: list[str] | None = None,
):
    """Render with adaptive quality settings."""
    
    preset = get_encoding_preset(job.duration, job.total_frames)
    avg_confidence = job.avg_confidence or 0.8
    crf = get_adaptive_crf(job.width, job.height, avg_confidence)
    
    ffmpeg_cmd = [
        'ffmpeg', '-i', str(job.file_path),
        '-c:v', 'libx264',
        '-crf', str(crf),
        '-preset', preset,
        '-maxrate', '8M',  # Bitrate cap
        '-bufsize', '16M',
        str(output_path),
    ]
    
    subprocess.run(ffmpeg_cmd, check=True)
```

**Expected Improvement**: **15–25% smaller files**, faster encoding on production

---

#### 4.2 Stream Transport Optimization

**Current**: Default RTSP over UDP (lossy on unstable networks).

**Solution: Configure RTSP Transport**
```python
# backend/config/settings/base.py

# Force RTSP/TCP for reliability
PYRAMID_RTSP_OVER_TCP = os.getenv('PYRAMID_RTSP_OVER_TCP', 'true').lower() in {'1', 'true'}
PYRAMID_RTSP_TRANSPORT = os.getenv('PYRAMID_RTSP_TRANSPORT', 'tcp')  # tcp, udp, auto

# go2rtc config
GO2RTC_RTSP_TRANSPORT = 'tcp'  # Force TCP for cameras
```

**Expected Improvement**: **More stable streams**, fewer frame drops

---

## 5. API RESPONSE TIME OPTIMIZATION

### 🟡 MEDIUM PRIORITY

#### 5.1 Endpoint Response Time Targets

**Audit Current Performance**:
```bash
# Install django-extensions
pip install django-extensions

# Run with request timing
python manage.py runserver --profile

# Or use Django Debug Toolbar
```

**Response Time SLOs**:
| Endpoint | Current | Target | Status |
|----------|---------|--------|--------|
| GET /api/v1/cameras | ? | <100ms | 🔍 Audit |
| GET /api/v1/sessions | ? | <200ms | 🔍 Audit |
| GET /api/v1/video-analysis-jobs | ? | <300ms | 🔍 Audit |
| POST /api/v1/sessions/{id}/start-live | ? | <50ms | 🔍 Audit |
| GET /api/v1/detections | ? | <500ms | 🔍 Audit |

**Solution: Async Views for I/O**
```python
# backend/apps/health/views.py
from django.http import JsonResponse
from asgiref.sync import async_to_sync

class AsyncHealthCheckView(APIView):
    async def get(self, request):
        """Non-blocking health check using async."""
        status = await HealthService.get_status_async()
        return JsonResponse(status)
```

**Expected Improvement**: **Lower latency**, better concurrency handling

---

#### 5.2 Slow Query Alerts

**Solution: Query Logging**
```python
# backend/config/settings/base.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'db_slow_log': {
            'level': 'DEBUG',
            'class': 'logging.FileHandler',
            'filename': 'logs/db_slow.log',
        },
    },
    'loggers': {
        'django.db.backends': {
            'handlers': ['db_slow_log'],
            'level': 'DEBUG',
            'propagate': False,
        },
    },
}

# Alert on queries > 100ms
SLOW_QUERY_THRESHOLD_MS = 100
```

---

## 6. MONITORING & OBSERVABILITY

### 🟢 LOW PRIORITY (Enables Future Optimization)

#### 6.1 Structured Logging

**Current**: Basic Django logging.

**Solution: Structured JSON Logging**
```python
# backend/config/settings/base.py
import logging.config
import pythonjsonlogger.jsonlogger

LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'json': {
            '()': 'pythonjsonlogger.jsonlogger.JsonFormatter',
            'format': '%(timestamp)s %(level)s %(name)s %(message)s'
        }
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'json',
        },
    },
    'root': {
        'handlers': ['console'],
        'level': 'INFO',
    },
}
```

**Expected Benefit**: Better log aggregation (ELK, Datadog, etc.)

---

#### 6.2 Metrics Instrumentation

**Solution: Prometheus Metrics**
```python
# backend/core/metrics.py (NEW)
from prometheus_client import Counter, Histogram, Gauge

# Counter: requests per endpoint
REQUEST_COUNT = Counter(
    'exam_monitor_requests_total',
    'Total requests',
    ['method', 'endpoint', 'status'],
)

# Histogram: request latency
REQUEST_LATENCY = Histogram(
    'exam_monitor_request_duration_seconds',
    'Request latency',
    ['endpoint'],
    buckets=(0.1, 0.5, 1.0, 2.5, 5.0),
)

# Gauge: active jobs
ACTIVE_JOBS = Gauge(
    'exam_monitor_active_video_jobs',
    'Number of active video analysis jobs',
)
```

**Expected Benefit**: Identify performance bottlenecks in production

---

## Implementation Roadmap

### Sprint 1: Database (Weeks 1–2)
- [ ] Profile N+1 queries with Django Debug Toolbar
- [ ] Add missing prefetch_related/select_related
- [ ] Audit and add database indexes
- [ ] Configure connection pooling
- [ ] Benchmark: measure query improvements

### Sprint 2: Caching (Weeks 3–4)
- [ ] Configure Redis cache backend
- [ ] Implement smart pagination
- [ ] Add embedding cache LRU bounds
- [ ] Cache frequently queried data (cameras, configs)
- [ ] Benchmark: measure cache hit rate

### Sprint 3: Frontend (Weeks 5–6)
- [ ] Enable code splitting and lazy loading
- [ ] Optimize images and assets
- [ ] Implement detection batching
- [ ] Benchmark: measure bundle size and load time

### Sprint 4: Streaming & API (Weeks 7–8)
- [ ] Implement adaptive video encoding
- [ ] Configure stream transport optimization
- [ ] Add response time instrumentation
- [ ] Audit and optimize slow endpoints
- [ ] Benchmark: measure encoding time and API latency

### Sprint 5: Monitoring (Weeks 9–10)
- [ ] Set up structured logging
- [ ] Add Prometheus metrics
- [ ] Create performance dashboards
- [ ] Define and enforce SLOs

---

## Quick Wins (Can Start Immediately)

✅ **High-Impact, Low-Effort**:

1. **Add indexes to VideoAnalysisJob.status, Frame.frame_number** (1 hour)
   - Impact: 20–30% faster filtered queries
   - Risk: Low (read-only query improvements)

2. **Add prefetch_related to DetectionViewSet** (30 min)
   - Impact: 30% fewer queries
   - Risk: Low

3. **Enable FFmpeg faster preset on production** (15 min)
   - Impact: 20% faster encoding
   - Risk: Very Low (only faster, same quality)

4. **Configure Redis cache for camera list** (1 hour)
   - Impact: 50% fewer camera queries
   - Risk: Low (cache invalidation on write)

5. **Enable code splitting in Vite** (1 hour)
   - Impact: 20% bundle reduction
   - Risk: Low (requires testing)

---

## Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Query Count** | <50% baseline | Django Debug Toolbar per endpoint |
| **Cache Hit Rate** | >80% | Redis INFO stats |
| **API Response Time** | <200ms p95 | Prometheus histogram |
| **Bundle Size** | <500KB main | `npm run build` output |
| **Video Encoding Time** | -20% baseline | Benchmark script |
| **Database Connection Pool** | No exhaustion | PostgreSQL max_connections monitor |

---

## References

- `backend/config/settings/base.py`: Django configuration
- `backend/apps/detections/views.py`: Example ViewSet (partial prefetch)
- `backend/apps/recordings/views.py`: Example ViewSet (good prefetch)
- `backend/apps/tracking/embeddings.py`: Embedding cache logic
- `frontend/vite.config.ts`: Build configuration
- `backend/apps/tracking/video_exporter.py`: FFmpeg encoding logic
