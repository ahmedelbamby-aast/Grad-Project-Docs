# Research: FastAPI Stack — Auth, Database, Theming

**Feature**: 001-exam-monitor-dashboard | **Date**: 2026-02-27
**Spec**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md)

These three research topics resolve technology decisions for the FastAPI/React stack
defined in `plan.md`. Each topic provides: Decision, Rationale, Alternatives Considered.

---

## Topic 1: Authentication Architecture

### Decision: **JWT (access + refresh tokens) in httpOnly cookies, using PyJWT + argon2-cffi**

### Rationale

FastAPI has no built-in authentication system (unlike Django's `contrib.auth`). A JWT-based
approach is the natural fit because:

1. **Stateless access tokens**: Short-lived access tokens (15–30 min) allow the backend to
   validate requests without a database lookup on every request. For a system handling up to
   2,400 WebSocket messages/second, eliminating per-request DB hits for auth is significant.

2. **httpOnly cookie transport**: Tokens are set as `httpOnly`, `Secure`, `SameSite=Lax`
   cookies — never exposed to JavaScript. This eliminates XSS token theft entirely. The React
   SPA sends cookies automatically on every `fetch()` and WebSocket handshake (`wss://`
   inherits cookies from the origin). No custom `Authorization` header management needed in
   the frontend.

3. **Refresh token rotation**: A long-lived refresh token (7 days) stored as a separate
   httpOnly cookie (`/api/v1/auth/refresh` path-scoped to limit exposure). On each refresh,
   the old refresh token is invalidated and a new pair is issued — this is "refresh token
   rotation" which limits the window of a stolen refresh token.

4. **Configurable inactivity timeout (default 30 min)**: The access token lifetime itself
   serves as the inactivity timeout. Each successful refresh resets the clock. If the user
   is inactive for >30 min, the access token expires and the frontend's silent refresh
   interceptor detects the 401, attempts a refresh, and if the refresh token is also expired
   or revoked, redirects to login. The 30-minute default is stored in Pydantic Settings
   (`ACCESS_TOKEN_EXPIRE_MINUTES=30`) and is configurable via `.env`.

5. **Session invalidation on logout**: On logout, the backend: (a) adds the current access
   token's `jti` (JWT ID) to a short-lived blocklist (Redis key with TTL = remaining access
   token lifetime), and (b) deletes the refresh token from the database. This provides
   instant invalidation without maintaining a full session store. The blocklist is tiny
   (only tokens for actively-logging-out users, auto-expiring) — not a scalability concern.

6. **WebSocket authentication**: On WebSocket connect, FastAPI reads the access token from
   the cookie header in the handshake request. The token is validated once at connection
   time. For long-lived WebSocket connections (monitoring sessions), a periodic re-validation
   (every 5 min) checks the blocklist to catch post-logout invalidation.

7. **Future admin role**: JWT `payload.role` field (`"instructor"` | `"admin"`) enables
   role-based access control without schema changes. FastAPI dependency functions check the
   role claim: `Depends(require_role("admin"))`.

### Token Structure

```
Access Token (httpOnly cookie, 30 min):
{
  "sub": "<instructor_uuid>",      // subject — user ID
  "email": "instructor@uni.edu",
  "role": "instructor",
  "jti": "<unique-token-id>",      // for blocklist on logout
  "exp": 1740700000,               // expiry timestamp
  "iat": 1740698200                // issued-at timestamp
}

Refresh Token (httpOnly cookie, 7 days, path-scoped):
{
  "sub": "<instructor_uuid>",
  "jti": "<unique-token-id>",
  "exp": 1741302800,
  "type": "refresh"
}
```

### Library Choices

#### PyJWT ≥ 2.9 (chosen) vs python-jose

| Criterion | PyJWT | python-jose |
|-----------|-------|-------------|
| **Maintenance** | Actively maintained (70M+ downloads/month, regular releases through 2025-2026). Core focus: JWT encoding/decoding. | **Effectively unmaintained** — last PyPI release was 3.3.0 in 2021. Open issues and PRs are unaddressed. Depends on `ecdsa` or `pycryptodome` for crypto backends. |
| **Dependencies** | Minimal — only `cryptography` as optional backend for RSA/EC. For HS256 (HMAC), zero extra deps. | Pulls in `rsa`, `ecdsa`, or `pycryptodome` — heavier dependency tree with its own vulnerability surface. |
| **Algorithm support** | HS256, HS384, HS512, RS256, RS384, RS512, ES256, ES384, ES512, PS256, PS384, PS512, EdDSA. Full coverage. | Same set, but via fragmented crypto backends. |
| **API surface** | `jwt.encode()`, `jwt.decode()` — simple and explicit. Built-in claim validation (`exp`, `nbf`, `aud`, `iss`). | Similar API but wraps PyJWT internally in some code paths. |
| **FastAPI ecosystem** | FastAPI's own documentation (2025+) uses PyJWT in examples. `fastapi-users` library uses PyJWT. | `python-jose` was in early FastAPI tutorials (2020-2022) but has been replaced in newer docs due to maintenance concerns. |
| **Security advisories** | Responsive — CVEs patched within days. | Slow — known issues linger due to inactive maintenance. |

**Decision: PyJWT**. python-jose's unmaintained status is a liability. PyJWT is the de facto
standard with active security maintenance. The plan.md reference to python-jose should be
updated to PyJWT.

#### argon2-cffi ≥ 23.1 (chosen) vs passlib

| Criterion | argon2-cffi | passlib |
|-----------|-------------|---------|
| **Algorithm** | Argon2id — winner of the 2015 Password Hashing Competition. Memory-hard (resistant to GPU/ASIC cracking). Recommended by OWASP 2025 as the #1 choice. | Multi-algorithm wrapper supporting bcrypt, PBKDF2, scrypt, argon2. |
| **Maintenance** | Actively maintained (28M+ downloads/month). Pure Python + CFFI bindings to the reference C implementation. | **Effectively deprecated** — last release was 1.7.4 in 2020. No Python 3.12+ testing. Maintainer announced reduced activity. GitHub issues unanswered. |
| **Performance** | Argon2id with default params: ~250ms per hash on modern hardware. Tunable `time_cost`, `memory_cost`, `parallelism`. | Depends on the selected algorithm. bcrypt is ~300ms. |
| **API** | `argon2.PasswordHasher().hash(password)` / `.verify(hash, password)`. Automatic salt generation. Built-in parameter migration (re-hashes on verify if params changed). | `CryptContext(schemes=["bcrypt"]).hash(password)` / `.verify(password, hash)`. More boilerplate for multi-scheme support. |
| **FastAPI fit** | Single-purpose, lightweight. Integrates cleanly into a `core/security.py` module with 10 lines of code. | Heavier, multi-purpose. Overkill when only one algorithm is needed. |

**Decision: argon2-cffi**. Argon2id is the OWASP-recommended algorithm. passlib is
unmaintained and adds unnecessary abstraction when only one hashing algorithm is needed.

### Auth Flow Summary

```
Login:
  POST /api/v1/auth/login  { email, password }
    → Verify password with argon2-cffi
    → Generate access token (PyJWT, HS256, 30 min)
    → Generate refresh token (PyJWT, HS256, 7 days)
    → Store refresh token hash in DB (instructor_id, jti, expires_at)
    → Set both as httpOnly cookies
    → Return instructor profile JSON

Silent Refresh (frontend interceptor on 401):
  POST /api/v1/auth/refresh  (refresh token cookie sent automatically)
    → Validate refresh token
    → Check refresh token exists in DB and not revoked
    → Issue new access + refresh token pair (rotation)
    → Delete old refresh token from DB
    → Set new cookies

Logout:
  POST /api/v1/auth/logout  (access + refresh cookies sent)
    → Add access token jti to Redis blocklist (TTL = remaining lifetime)
    → Delete refresh token from DB
    → Clear both cookies

Every Authenticated Request:
  → Read access token from cookie
  → Validate signature + expiry with PyJWT
  → Check jti not in Redis blocklist
  → Extract instructor_id and role from claims
```

### Alternatives Considered

| Alternative | Why Rejected |
|---|---|
| **Server-side sessions (DB/Redis store)** | Requires a database or Redis lookup on **every** request (including all WebSocket messages). At 2,400 msg/s peak for detection frames, this adds unnecessary load. JWT's stateless validation (signature check only) is more efficient. Additionally, FastAPI has no built-in session framework — implementing server-side sessions requires building the same infrastructure (cookie management, session store, expiry, invalidation) manually, negating the simplicity advantage sessions have in Django. |
| **JWT in localStorage + Authorization header** | Exposes tokens to XSS attacks. Requires manual header management on every fetch call and custom WebSocket auth (tokens can't be sent as cookies on WS handshake if using headers). httpOnly cookies are strictly more secure for browser-based SPAs. |
| **JWT in memory (React state) + refresh cookie** | Access token lost on page refresh, requiring a refresh call on every reload. Adds complexity and a flash of unauthenticated state. Cookie-based access token survives page refreshes naturally. |
| **OAuth2 / OpenID Connect** | Massive overengineering for a single-tenant university app with one identity provider (the app itself). No external IdP to federate with. Adds PKCE flow complexity, token endpoint configuration, and discovery metadata — all unnecessary. |
| **passlib[bcrypt]** | passlib is unmaintained (last release 2020). bcrypt is a solid algorithm but Argon2id is the 2025 OWASP #1 recommendation due to memory-hardness. |
| **python-jose** | Unmaintained since 2021. Fragmented crypto backends. PyJWT is actively maintained with 70M+ monthly downloads. |

### Risks & Mitigations

| Risk | Severity | Mitigation |
|------|----------|------------|
| JWT cannot be instantly revoked (stateless) | Medium | Redis blocklist for logout (TTL = access token remaining life). Blocklist is tiny and auto-expiring. Periodic WebSocket re-validation every 5 min catches invalidated tokens on long connections. |
| Token theft via cookie | Low | `httpOnly` (no JS access), `Secure` (HTTPS only), `SameSite=Lax` (no cross-origin sends). Refresh token path-scoped to `/api/v1/auth/refresh`. |
| HS256 key compromise | Low | Secret key loaded from `.env` via Pydantic Settings, stored in `SecurityVault` per constitution. Key rotation supported by checking `kid` (Key ID) header claim. |
| argon2 timing on slow hardware | Low | Tune `time_cost` and `memory_cost` to target ~250ms. argon2-cffi uses C bindings — consistent performance. Login is infrequent (once per session). |
| Refresh token database table growth | Low | Expired refresh tokens pruned by a periodic background task (APScheduler or FastAPI startup task). Rotation means at most 1 active refresh token per user. |

---

## Topic 2: Database & Recording Storage

### Decision: **PostgreSQL 16 + SQLAlchemy 2.0 async (asyncpg driver) + date-based partitioning for detection frames + filesystem recordings organized by date/camera**

### Rationale

#### PostgreSQL 16 over SQLite

| Criterion | PostgreSQL 16 | SQLite |
|-----------|---------------|--------|
| **Concurrent writes** | MVCC — 20 concurrent instructors writing simultaneously with zero contention. Connection pooling via asyncpg (or PgBouncer for advanced). | **Single-writer lock** — only one write transaction at a time. 20 instructors inserting detection frames simultaneously would serialize to a single queue, creating a bottleneck. WAL mode improves read concurrency but writes remain single-threaded. |
| **Write throughput** | Handles 2,400 inserts/second (20 instructors × 8 cameras × 15 FPS) with batch inserts. Tested to 100k+ inserts/sec with proper indexing. | ~50k inserts/sec for simple rows **but only if single-writer**. Under 20 concurrent writers with WAL, effective throughput drops significantly due to lock contention. |
| **Async support** | `asyncpg` — native async PostgreSQL driver, purpose-built for asyncio. 3× faster than psycopg3 async in benchmarks. SQLAlchemy 2.0 `create_async_engine(url, pool_size=20)` works natively. | `aiosqlite` — async wrapper around synchronous sqlite3 module. Not truly async at the driver level (uses thread pool internally). |
| **JSON support** | `JSONB` column type with GIN indexes. Efficient queries on prediction metadata, anomaly snapshots, session configs. `WHERE metadata @> '{"severity": "high"}'` uses index. | `JSON` type exists but no indexing. Every JSON query requires full-column scan or extract + index on virtual columns (fragile). |
| **Partitioning** | Native declarative partitioning (RANGE on timestamp). Critical for detection frames table which can reach millions of rows. Partitioned by date for efficient retention cleanup (`DROP PARTITION` is instant vs row-by-row DELETE). | No partitioning support. Large tables require manual sharding logic or periodic VACUUM after bulk deletes. |
| **Full-text search** | Built-in `tsvector` for searching anomaly notes and dismissed reasons without Elasticsearch. | FTS5 extension exists but less capable, no ranking, no multilingual stemming. |
| **Connection limit** | Default 100 connections. asyncpg pool of 20-40 connections handles the load. | N/A — file-level locking. |
| **Deployment** | Docker Compose: `postgres:16-alpine` (~80MB image). One-command setup. | Zero-config file — simpler deployment. But the concurrency limitation makes it unsuitable. |

**Decision: PostgreSQL 16**. SQLite's single-writer limitation is a hard blocker for 20
concurrent instructors writing detection frames at 15-30 FPS. This is not a premature
optimization — it's a fundamental architectural constraint.

#### SQLAlchemy 2.0 Async

| Decision | Detail |
|----------|--------|
| **Engine** | `create_async_engine("postgresql+asyncpg://...", pool_size=20, max_overflow=10)` — asyncpg is the fastest Python async PostgreSQL driver. |
| **Session** | `async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)` — sessions injected via FastAPI `Depends()`. |
| **Models** | Declarative base with `mapped_column()` (SQLAlchemy 2.0 style). Type hints on all columns. |
| **Migrations** | Alembic with async support (`alembic.ini` → `sqlalchemy.url` from Pydantic Settings). `alembic revision --autogenerate` for schema changes. |
| **Relationships** | `selectinload()` / `joinedload()` for eager loading (avoids N+1). `lazy="raise"` as default to catch accidental lazy loads in async context (which would raise `MissingGreenlet`). |
| **Bulk operations** | `session.execute(insert(DetectionFrame).values(batch))` for batch inserts of detection frames. 500-row batches balance memory and round-trip efficiency. |

#### Time-Series Partitioning for Detection Frames

The `DetectionFrame` table is the highest-volume table: up to 2,400 rows/second at peak
(20 instructors × 8 cameras × 15 FPS). Over a 3-hour exam session with 20 concurrent
instructors, this produces ~26 million rows. Over a semester, potentially billions of rows.

**Strategy: PostgreSQL declarative RANGE partitioning on `timestamp` by day.**

```
detection_frames (parent table — never queried directly)
├── detection_frames_2026_02_27  (Feb 27 partition)
├── detection_frames_2026_02_28  (Feb 28 partition)
├── detection_frames_2026_03_01  (Mar 1 partition)
└── ...auto-created by pg_partman or pre-created script
```

Benefits:
1. **Query performance**: Queries for a specific session (bounded by `started_at` /
   `ended_at`) automatically scan only relevant partitions (partition pruning).
2. **Retention cleanup**: `DROP TABLE detection_frames_2026_02_27` is instant (O(1)) vs
   `DELETE FROM detection_frames WHERE timestamp < '...'` which locks rows, generates WAL,
   and requires VACUUM. This directly supports the configurable per-term retention policy.
3. **Maintenance**: `VACUUM` and `ANALYZE` run per-partition, reducing lock duration.
4. **Index efficiency**: Each partition has its own B-tree indexes — smaller, faster index
   lookups per partition.

Partition management: A scheduled background task (APScheduler in FastAPI, or a cron job)
creates next-day partitions in advance and drops partitions past the retention window. Use
`pg_partman` extension if available, else a simple SQL function.

The `Detection` and `PyramidPrediction` tables reference `DetectionFrame.id`. These child
tables are NOT partitioned — they use standard B-tree indexes on `frame_id`. Since queries
always go through `DetectionFrame` first (filtered by session timestamp), the join is
efficient: partition pruning narrows the frame set, then child table lookups use indexed FKs.

#### Filesystem Layout for Recordings

Raw video files are stored on the local filesystem (not in the database). The database
`Recording` table stores metadata including the file path.

**Directory structure: `{MEDIA_ROOT}/recordings/{YYYY}/{MM}/{DD}/{camera_id}/`**

```
media/
└── recordings/
    └── 2026/
        └── 02/
            └── 27/
                ├── a1b2c3d4-cam-uuid/
                │   ├── session_e5f6g7h8.mp4       # raw video, no overlays
                │   └── session_e5f6g7h8.meta.json  # quick-access metadata cache
                └── i9j0k1l2-cam-uuid/
                    └── session_m3n4o5p6.mp4
```

Rationale:
1. **Date-first hierarchy**: Enables efficient retention cleanup — `rm -rf media/recordings/2025/`
   deletes an entire year. Matches the partition strategy for detection frames.
2. **Camera subdirectory**: Prevents filename collisions. Groups related recordings for a
   physical location.
3. **No session nesting**: Sessions are identified by filename, not an extra directory level.
   This avoids deeply nested paths which cause issues on Windows (260-char path limit without
   long path support).
4. **Flat within camera**: Simple `os.listdir()` for a camera's recordings on a given date.
5. **`.meta.json` sidecar** (optional optimization): Caches file size, duration, codec info
   to avoid repeated `ffprobe` calls. Regenerable from DB if deleted.

File naming: `session_{session_uuid_short}.mp4` — the first 8 chars of the session UUID for
human readability, with the full UUID stored in the database `Recording.video_file_path`.

#### Storage Capacity Warnings (80% / 95%)

A scheduled background task monitors disk usage on the `MEDIA_ROOT` volume:

| Threshold | Action |
|-----------|--------|
| **<80%** | Normal operation. Health endpoint reports `storage_status: "ok"`. |
| **≥80%** | Warning. Health endpoint reports `storage_status: "warning"`. WebSocket `health.update` pushes alert to all connected Health Dashboard clients. Audit log entry created. |
| **≥95%** | Critical. Health endpoint reports `storage_status: "critical"`. New recording starts are blocked (return 507 Insufficient Storage). WebSocket alert pushed. Audit log entry. |
| **≥98%** | Emergency. Same as 95% plus: oldest recordings past retention window are auto-purged (if auto-purge is enabled in config). |

Implementation: `shutil.disk_usage(MEDIA_ROOT)` called every 60 seconds by a background
task (FastAPI `BackgroundTasks` or APScheduler). Thresholds are configurable via Pydantic
Settings.

### Alternatives Considered

| Alternative | Why Rejected |
|---|---|
| **SQLite** | Single-writer lock is a hard blocker for 20 concurrent instructors. Not a scaling concern — it's a fundamental concurrency limitation. WAL mode improves reads but writes remain serialized. `aiosqlite` is a threadpool wrapper, not truly async. No partitioning support for detection frame retention. |
| **MySQL 8** | Viable but weaker ecosystem for Python async (no equivalent to asyncpg). JSONB support (`JSON` type) exists but GIN indexing is less mature. SQLAlchemy async works via `aiomysql` but it's less battle-tested than asyncpg. No significant advantage over PostgreSQL for this workload. |
| **MongoDB** | The data model is deeply relational (Instructor → Session → Frame → Detection → Prediction). MongoDB would require denormalization or `$lookup` for every query chain. No transactional integrity across collections without explicit multi-document transactions (verbose, fragile). |
| **TimescaleDB (extension)** | Adds hypertable abstraction for time-series, but PostgreSQL 16's native declarative partitioning is sufficient. TimescaleDB's continuous aggregates and compression are valuable for IoT telemetry (millions of sensors), but detection frames don't need aggregation — they're queried by session range and then discarded via retention. Adding an extension increases deployment complexity without proportional benefit. |
| **Object storage (S3/MinIO) for recordings** | Correct for production-scale multi-server deployments, but overengineering for a single-server graduation project. The plan already calls for a `BaseStorageBackend` abstraction — local filesystem is the default implementation, S3 can be added later without code changes. |
| **Storing recordings as BLOBs in PostgreSQL** | PostgreSQL `BYTEA` or Large Objects would massively bloat the database. A 1-hour 720p recording is ~500MB-2GB. This makes backups enormous, VACUUM expensive, and replication impractical. Filesystem is the industry standard for video storage with DB metadata pointers. |

### Risks & Mitigations

| Risk | Severity | Mitigation |
|------|----------|------------|
| Detection frame volume overwhelms DB | Medium | Batch inserts (500-row batches), date-based partitioning, retention auto-purge. Monitor insert latency via health endpoint. |
| Disk space exhaustion mid-exam | High | 80/95% warnings, 95% blocks new recordings, 98% auto-purge. `shutil.disk_usage()` checked every 60s. |
| asyncpg connection pool exhaustion | Medium | `pool_size=20, max_overflow=10` allows 30 concurrent connections. Pool timeout = 30s with clear error. Monitor active connections via health endpoint. |
| Alembic migration conflicts in team dev | Low | Linear migration chain. `alembic heads` check in CI. Only one migration branch at a time. |
| Windows long path issue for recordings | Low | Date-first hierarchy keeps paths short (~80 chars). Session UUID truncated to 8 chars in filename. Full UUID only in database. |
| Orphaned recording files (DB row deleted, file remains) | Low | Recording deletion is a two-step service method: delete DB row, then delete file. Background task scans for orphans weekly. |

---

## Topic 3: Theme System Architecture

### Decision: **CSS Custom Properties (variables) with `data-theme` attribute on `<html>`, theme tokens in dedicated `.css` files, synchronized via Zustand + localStorage**

### Rationale

#### Why CSS Custom Properties

1. **Instant switching (<300ms requirement)**: Changing `document.documentElement.dataset.theme`
   triggers a single CSS cascade recalculation. Measured at ~5-50ms in modern browsers — well
   under the 300ms requirement. No JavaScript recalculation, no component re-renders, no
   style object recomputation. The browser's native CSS engine handles the transition in a
   single style recalc + repaint pass.

2. **Zero JavaScript runtime overhead**: CSS custom properties are resolved by the browser's
   style engine, not by JavaScript. There is no per-component style computation (unlike
   CSS-in-JS which runs JS on every render to produce styles). For a real-time monitoring app
   rendering 8 video feeds with Canvas overlays at 15+ FPS, minimizing JS work is critical.

3. **Canvas/bounding box color integration**: Bounding box colors are read from CSS variables
   via `getComputedStyle()` at Canvas initialization and cached. When the theme changes, a
   single event listener re-reads the CSS variables and updates the cached color values. The
   Canvas drawing loop uses these cached values — no per-frame CSS lookups.

   ```
   Theme switch → getComputedStyle(root).getPropertyValue('--color-bbox-student')
                → Cache in Zustand detectionStore
                → Canvas draw loop uses cached values
   ```

4. **WCAG 2.1 AA verification**: Theme tokens are defined as a structured set of CSS
   variables with semantic names (`--color-text-primary`, `--color-bg-primary`, etc.).
   Each theme file is a self-contained contract of exactly these variables. Contrast ratios
   can be verified at design time by running automated checks (e.g., `color-contrast` PostCSS
   plugin or a custom build-time script) against each theme file's variable pairs. This is
   harder with CSS-in-JS where colors are scattered across component files.

5. **No FOUC (Flash of Unstyled Content)**: A synchronous `<script>` in `index.html` `<head>`
   reads `localStorage.getItem("theme")` and sets `data-theme` before the first paint. Since
   this runs before any CSS is applied, the correct theme is active from the first frame.
   This is unique to SPAs (no SSR mismatch risk).

#### Theme Token System

Three theme files define identical variable sets:

| Token | Purpose | Black+Purple | Full Black | White |
|-------|---------|-------------|------------|-------|
| `--color-bg-primary` | Page/card background | `#0D0D0D` | `#000000` | `#FFFFFF` |
| `--color-bg-secondary` | Elevated surfaces | `#1A1A2E` | `#0A0A0A` | `#F3F4F6` |
| `--color-bg-tertiary` | Deeply nested elements | `#16213E` | `#141414` | `#E5E7EB` |
| `--color-accent` | Primary accent (buttons, links) | `#7C3AED` | `#A78BFA` | `#6D28D9` |
| `--color-accent-hover` | Accent interaction state | `#6D28D9` | `#8B5CF6` | `#5B21B6` |
| `--color-text-primary` | Body text | `#F5F5F5` | `#E5E5E5` | `#111827` |
| `--color-text-secondary` | Muted text, labels | `#A1A1AA` | `#A1A1AA` | `#6B7280` |
| `--color-text-on-accent` | Text on accent backgrounds | `#FFFFFF` | `#FFFFFF` | `#FFFFFF` |
| `--color-border` | Card/panel borders | `#2D2D3F` | `#1F1F1F` | `#D1D5DB` |
| `--color-alert-high` | High severity anomaly | `#EF4444` | `#F87171` | `#DC2626` |
| `--color-alert-medium` | Medium severity anomaly | `#F59E0B` | `#FBBF24` | `#D97706` |
| `--color-alert-low` | Low severity anomaly | `#3B82F6` | `#60A5FA` | `#2563EB` |
| `--color-bbox-student` | Student bounding box stroke | `#22D3EE` (cyan) | `#22D3EE` (cyan) | `#0891B2` (teal) |
| `--color-bbox-teacher` | Teacher bounding box stroke | `#A78BFA` (purple) | `#C4B5FD` (light purple) | `#7C3AED` (purple) |
| `--color-bbox-anomaly` | Anomaly highlight stroke | `#F87171` (red) | `#FCA5A5` (light red) | `#EF4444` (red) |
| `--color-success` | Positive states | `#22C55E` | `#4ADE80` | `#16A34A` |
| `--shadow-card` | Card elevation | `0 2px 8px rgba(0,0,0,0.4)` | `none` | `0 1px 3px rgba(0,0,0,0.1)` |
| `--shadow-modal` | Modal elevation | `0 8px 32px rgba(0,0,0,0.6)` | `0 8px 32px rgba(0,0,0,0.8)` | `0 4px 16px rgba(0,0,0,0.15)` |
| `--radius-sm` | Small border radius | `4px` | `4px` | `6px` |
| `--radius-md` | Medium border radius | `8px` | `8px` | `8px` |
| `--radius-lg` | Large border radius | `12px` | `12px` | `12px` |

#### Bounding Box Color Distinguishability

Bounding box colors must remain distinguishable from each other AND from the video feed
background across all three themes. This is critical because the video feed content varies
(light exam rooms, dark rooms, etc.).

Design constraints:
1. Student (cyan family), Teacher (purple family), and Anomaly (red family) use three
   perceptually distinct hue regions (180°, 270°, 0°) — separated by ≥90° on the color
   wheel. This ensures distinguishability for both normal vision and the most common
   color vision deficiency (deuteranopia).
2. Stroke width of 2-3px with an inner shadow/outline ensures boxes are visible against
   any video background — a 1px dark outline around the colored stroke provides contrast on
   both light and dark video regions.
3. Tracking ID labels use a solid background pill (`--color-bg-primary` with 85% opacity)
   behind the text to guarantee readability regardless of video content.
4. All three themes use the same hue families (cyan, purple, red) but adjust lightness for
   the white theme (darker variants) to maintain WCAG AA contrast against the lighter UI
   chrome.

#### Implementation Architecture

```
index.html:
  <head>
    <script>
      // Synchronous — runs before first paint, prevents FOUC
      const theme = localStorage.getItem('theme') || 'black-purple';
      document.documentElement.dataset.theme = theme;
    </script>
    <link rel="stylesheet" href="/src/themes/variables.css">
  </head>

themes/variables.css:
  [data-theme="black-purple"] { --color-bg-primary: #0D0D0D; ... }
  [data-theme="full-black"]   { --color-bg-primary: #000000; ... }
  [data-theme="white"]        { --color-bg-primary: #FFFFFF; ... }

components/layout/ThemeToggle.tsx:
  → Reads current theme from Zustand themeStore
  → On switch: sets document.documentElement.dataset.theme
              + localStorage.setItem("theme", newTheme)
              + updates Zustand store (for React components that read theme name)
              + dispatches "themechange" CustomEvent (for Canvas color cache refresh)

components/camera/BoundingBoxOverlay.tsx:
  → On mount + on "themechange" event:
     reads --color-bbox-student, --color-bbox-teacher, --color-bbox-anomaly
     via getComputedStyle() and caches in local variables
  → requestAnimationFrame draw loop uses cached color strings
```

### Alternatives Considered

| Alternative | Why Rejected |
|---|---|
| **CSS-in-JS (styled-components / Emotion)** | Generates styles in JavaScript at runtime — every styled component runs JS to produce CSS on each render. For a dashboard rendering 8+ video feeds at 15 FPS, this adds unnecessary JavaScript execution competing with Canvas drawing and WebSocket processing. Theme switching requires re-running all style computations. Measured overhead: 5-15ms per component tree re-render for style generation vs ~0ms for CSS custom property swap. Also: styled-components' `ThemeProvider` passes theme via React Context, causing all themed components to re-render on theme change — exactly what we want to avoid. Bundle size: styled-components adds ~12KB gzipped, Emotion ~7KB — unnecessary when native CSS achieves the same result. |
| **Tailwind CSS with theme classes** | Tailwind uses utility classes (e.g., `bg-gray-900 dark:bg-white`). For 3 themes (not just light/dark), requires either: (a) custom variant plugins for each theme (`theme-bp:bg-gray-900 theme-fb:bg-black theme-w:bg-white`) which triples every utility class in the HTML and balloons markup, or (b) `@apply` with CSS variables under the hood — which is essentially CSS custom properties with extra steps and Tailwind's build pipeline. Tailwind is excellent for utility-first styling of layouts and components, but for **theming specifically** (the variable layer), CSS custom properties are more direct. **Note**: Tailwind CAN be used alongside CSS custom properties — Tailwind for utility layout classes, CSS variables for theme-dependent colors. This is a viable hybrid approach but adds Tailwind's build dependency (~3s HMR on large projects) for what CSS custom properties handle natively. |
| **CSS Modules with theme imports** | Each component imports a theme-specific module (e.g., `import styles from './Button.module.css'`). Theme switching requires swapping the imported module — this cannot be done at runtime without a page reload or complex dynamic import machinery. Does not meet the <300ms no-reload requirement natively. |
| **Prefers-color-scheme media query only** | Only supports 2 themes (light/dark) based on OS setting. Cannot implement 3 arbitrary themes. Cannot be overridden per-user without JavaScript — which brings us back to CSS custom properties anyway. |

### WCAG 2.1 AA Verification Strategy

Automated verification at build time:

1. **Theme contract file**: Each theme defines key foreground/background pairs:
   - `--color-text-primary` on `--color-bg-primary` → must be ≥4.5:1
   - `--color-text-secondary` on `--color-bg-primary` → must be ≥4.5:1
   - `--color-text-on-accent` on `--color-accent` → must be ≥4.5:1
   - `--color-alert-high` on `--color-bg-secondary` → must be ≥3:1 (large text / UI components)

2. **Build-time check**: A Node.js script (or Vitest test file) parses each theme's CSS
   variables, computes contrast ratios using the WCAG luminance formula, and fails the build
   if any pair is below threshold. This runs in CI on every PR.

3. **Runtime snapshot**: Playwright E2E tests capture screenshots of each theme and run
   `axe-core` accessibility scans — catching dynamic elements (Canvas overlays, modals)
   that static analysis misses.

### Risks & Mitigations

| Risk | Severity | Mitigation |
|------|----------|------------|
| Canvas colors not updating on theme switch | Medium | `CustomEvent("themechange")` listener in Canvas component. Re-reads CSS variables and re-caches on every switch. Unit test verifies color cache invalidation. |
| FOUC on first load | Low | Synchronous `<script>` in `<head>` sets `data-theme` before any CSS loads. Verified by Playwright test measuring time-to-correct-theme < 100ms. |
| Theme variable mismatch (missing variable in one theme) | Medium | TypeScript theme type defining all required tokens. Build-time script validates all three theme files define identical variable sets. CI gate. |
| Bounding box colors indistinguishable on certain video backgrounds | Medium | 2-3px stroke with 1px dark inner outline. Tested on sample exam footage across all themes. Instructor can adjust box opacity in settings (future feature). |
| WCAG regression on new components | Low | `axe-core` scan in Playwright E2E suite. Build-time contrast ratio check on all token pairs. Both block CI merge. |

---

## Summary of Decisions

| # | Topic | Decision | Key Rationale |
|---|-------|----------|---------------|
| T-1 | Auth Library (JWT) | PyJWT ≥ 2.9 | Actively maintained (70M+/month). python-jose unmaintained since 2021. |
| T-1 | Auth Library (Hashing) | argon2-cffi ≥ 23.1 | OWASP #1 recommendation (Argon2id). passlib unmaintained since 2020. |
| T-1 | Auth Strategy | JWT access+refresh in httpOnly cookies | Stateless validation (no DB hit per request), WebSocket-compatible via cookies, instant logout via Redis blocklist, configurable inactivity timeout via token expiry. |
| T-2 | Database | PostgreSQL 16 + asyncpg | Concurrent writes (MVCC), JSONB, native partitioning, true async driver. SQLite's single-writer lock is a hard blocker. |
| T-2 | ORM | SQLAlchemy 2.0 async | `AsyncSession` via FastAPI DI, `mapped_column()` typing, Alembic migrations, `lazy="raise"` default. |
| T-2 | Detection Frame Storage | Date-based partitioning (RANGE on timestamp) | Instant retention cleanup (`DROP PARTITION`), partition pruning on session queries, per-partition VACUUM. |
| T-2 | Recording Storage | Filesystem: `{MEDIA_ROOT}/recordings/{YYYY}/{MM}/{DD}/{camera_id}/` | Date-first for retention, camera-scoped for isolation, short paths for Windows compat. |
| T-2 | Storage Monitoring | 80%/95%/98% thresholds, checked every 60s | Warning → block new recordings → auto-purge. WebSocket health alerts. |
| T-3 | Theme Mechanism | CSS Custom Properties + `data-theme` attribute | ~5-50ms switch (native CSS cascade), zero JS runtime overhead, FOUC-free via `<head>` script, WCAG verifiable at build time. |
| T-3 | Bounding Box Colors | Three hue families (cyan/purple/red) with dark inner outline | ≥90° hue separation, distinguishable under deuteranopia, visible on any video background. |
| T-3 | WCAG Verification | Build-time contrast script + Playwright axe-core | Automated, CI-gated, catches both static and dynamic accessibility issues. |
