# Research: FastAPI WebSocket Architecture for Real-Time Detection Push

**Feature**: 001-exam-monitor-dashboard | **Date**: 2026-02-27  
**Spec**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Parent Research**: [research.md](research.md)

This document researches the WebSocket architecture for pushing real-time YOLO detection
results (bounding boxes, tracking IDs, anomaly alerts) from a FastAPI backend to a React
frontend. It resolves the architectural gap between plan.md (FastAPI) and the existing
websocket-api.md contract (which assumed Django Channels).

---

## Scale Parameters

Before evaluating any decision, the concrete load envelope:

| Parameter | Value | Source |
|-----------|-------|--------|
| Cameras (system-wide) | Up to ~40 (20 instructors × avg 2, max 8 each) | spec.md |
| Detection FPS per camera | 15–30 | User requirement |
| Concurrent instructors | Up to 20 | spec.md |
| Cameras per instructor | 1–8 | spec.md |
| Shared cameras | Yes — multiple instructors can view the same camera | spec.md Q&A |
| Worst-case frame messages produced | ~40 cameras × 30 FPS = **1,200 msg/s** | Derived |
| Worst-case fan-out messages delivered | ~1,200 × avg 3 subscribers = **3,600 msg/s** | Derived (conservative) |
| Detection payload size (JSON) | ~300–800 bytes per frame (5–15 detections) | websocket-api.md |
| Anomaly alert rate | 0–5 per minute per camera (event-driven) | websocket-api.md |
| Health update rate | 1 message every 10 seconds (broadcast) | websocket-api.md |

**Conclusion**: This is a moderate-scale system. Total throughput ceiling is ~4,000
delivered messages/second — well within single-process capability with async I/O.
No distributed broker is required for correctness, though one may be chosen for
architectural cleanliness.

---

## Decision 1: WebSocket Message Serialization Format

### Recommendation: **JSON (primary) with MessagePack as an optional upgrade path**

### Analysis

#### Option A: JSON

| Dimension | Assessment |
|-----------|-----------|
| Encode/decode speed (Python) | `orjson` — 3–10× faster than stdlib `json`. Encodes a typical detection frame (~500 bytes) in **<5 µs**. At 1,200 msg/s = **<6 ms/s** total CPU. Negligible. |
| Encode/decode speed (JS) | Native `JSON.parse()` — V8-optimized C++ implementation. Sub-microsecond for small payloads. **Zero dependency.** |
| Payload size | A typical `detection.frame` message: **~600 bytes** as JSON. Field names like `"detection_class"`, `"confidence"`, `"tracking_id"` are repeated per detection but compressible. |
| WebSocket compression | `permessage-deflate` reduces JSON by 60–80% for repetitive structures → effective **~150–200 bytes**. Starlette/Uvicorn does NOT support `permessage-deflate` natively (as of 2026). Would require a reverse proxy (nginx) or manual compression. |
| Debuggability | **Excellent.** Browser DevTools WebSocket inspector shows messages as readable text. `wscat`/`websocat` CLI tools work out of the box. Critical for a grad project with active development. |
| Schema evolution | Add new fields freely — unknown fields ignored by consumers. No codegen required. |
| Ecosystem fit | FastAPI's `WebSocket.send_json()` is built-in. React/TypeScript — native. Pydantic models serialize to JSON natively via `model_dump()`. Zero friction. |

#### Option B: MessagePack

| Dimension | Assessment |
|-----------|-----------|
| Encode/decode speed (Python) | `msgpack` (C extension) — comparable to `orjson` for small payloads. Marginal difference at this scale. |
| Encode/decode speed (JS) | `@msgpack/msgpack` — ~2–5× slower than native `JSON.parse()` (JS implementation, no engine optimization). |
| Payload size | ~30–40% smaller than JSON for the same structure. Detection frame: **~400 bytes** vs ~600 bytes JSON. |
| WebSocket transport | Binary frames (`WebSocket.send_bytes()`). Works fine but requires explicit handling on both sides. |
| Debuggability | **Poor.** Browser DevTools shows binary blobs. Requires custom decode tooling. Slows development velocity significantly. |
| Ecosystem fit | Requires `msgpack` Python package + `@msgpack/msgpack` JS package. FastAPI has no built-in MessagePack support — manual `send_bytes()`. |

#### Option C: Protocol Buffers (Protobuf)

| Dimension | Assessment |
|-----------|-----------|
| Encode/decode speed | Fastest for large, schema-rigid payloads (~50+ fields). Overkill for 5–15 field messages. |
| Payload size | Smallest — ~50–60% of JSON. Detection frame: **~250 bytes**. |
| Schema management | `.proto` files require compilation (`protoc`) to Python + TypeScript stubs. Adds a build step. Schema changes require recompilation on both sides. |
| Debuggability | **Very poor.** Completely opaque binary. Requires `protoc --decode` or custom tooling. |
| Ecosystem fit | `protobuf` Python package (Google) + `protobufjs` or `ts-proto` (JS/TS). No FastAPI integration — fully manual. Significant impedance mismatch with Pydantic models. |
| Maintenance burden | `.proto` → Python stubs → Pydantic conversion → send. `.proto` → TS stubs → store update. Every schema change touches 4 artifacts. |

### Bandwidth Reality Check

| Format | Per-message | × 1,200 msg/s | × 3,600 delivered/s | Per second |
|--------|-------------|----------------|----------------------|------------|
| JSON | ~600 B | 720 KB/s produced | 2.16 MB/s delivered | **2.16 MB/s** |
| MessagePack | ~400 B | 480 KB/s | 1.44 MB/s | **1.44 MB/s** |
| Protobuf | ~250 B | 300 KB/s | 0.90 MB/s | **0.90 MB/s** |
| JSON + deflate | ~180 B | 216 KB/s | 0.65 MB/s | **0.65 MB/s** |

Even worst-case uncompressed JSON is **2.16 MB/s** — trivially handled by any
modern network and server process. The difference between JSON (2.16 MB/s) and
Protobuf (0.90 MB/s) is **1.26 MB/s** — irrelevant for a LAN-deployed university
system.

### Decision Rationale

1. **At this scale, serialization format is not the bottleneck.** CPU time for JSON
   encoding (via `orjson`) is <6 ms/s. Bandwidth is <3 MB/s. Neither warrants
   binary format complexity.

2. **Debuggability is a multiplier on development velocity.** Readable WebSocket
   messages in browser DevTools eliminate an entire class of debugging friction.
   For a grad project with iterative development, this is a significant factor.

3. **Zero-dependency path.** JSON requires no additional packages on the frontend
   and only `orjson` (optional, for speed) on the backend. Pydantic `.model_dump()`
   → `orjson.dumps()` → `ws.send_bytes()` is a two-step pipeline.

4. **Schema evolution is frictionless.** Adding a field to a detection message
   requires changing one Pydantic model and one TypeScript interface. No
   recompilation, no codegen, no proto sync.

5. **Upgrade path preserved.** If profiling ever shows serialization as a
   bottleneck (unlikely), switching to MessagePack is a localized change: replace
   `orjson.dumps()` with `msgpack.packb()` on the server, `JSON.parse()` with
   `msgpack.decode()` on the client. The message schema, WebSocket routing,
   and application logic remain identical. Design the message schemas now with
   compact field names to minimize JSON overhead.

### Optimization: Compact Field Names

To reduce JSON payload without sacrificing readability in the codebase, use
Pydantic `serialization_alias` to emit short wire names while keeping long
Python attribute names:

```
Python model: detection_class, confidence, bounding_box, tracking_id
Wire JSON:    dc, cf, bb, tid
```

This reduces a typical detection frame from ~600 bytes to ~350 bytes — approaching
MessagePack size while retaining JSON debuggability. The TypeScript interface maps
short names back to readable properties.

### Alternatives Rejected

| Alternative | Why Rejected |
|-------------|-------------|
| **MessagePack** as primary | Debuggability loss outweighs 30% size reduction at this scale. JS decode slower than native `JSON.parse()`. |
| **Protobuf** | Build-step burden (proto compilation), impedance mismatch with Pydantic/TypeScript, poor debuggability. Justified only at >100k msg/s or >100 fields. |
| **FlatBuffers** | Zero-copy read access is meaningless for WebSocket messages that must be fully deserialized anyway. Even more complex than Protobuf. |
| **CBOR** | Similar tradeoffs to MessagePack, smaller ecosystem, less tooling. |

---

## Decision 2: Channel/Room Pattern for Per-Camera Fan-Out

### Recommendation: **In-process `ConnectionManager` with camera-keyed room registry, Redis pub/sub reserved for horizontal scaling**

### Problem

The detection pipeline produces frames per camera. Multiple instructors may subscribe
to the same camera. The WebSocket layer must route each camera's detection stream
only to instructors who have subscribed to that camera — a classic pub/sub fan-out.

### Architecture

```
Detection Pipeline (per camera)
    │
    ▼
ConnectionManager.broadcast(camera_id, message)
    │
    ├──▶ Instructor A (subscribed to camera_1, camera_3)
    ├──▶ Instructor B (subscribed to camera_1)
    └──▶ Instructor C (subscribed to camera_2, camera_3)
```

### Option A: In-Process `ConnectionManager` (Recommended for Initial Deployment)

A singleton `ConnectionManager` class maintains:

```
rooms: dict[str, set[WebSocket]]
    "camera:{camera_id}"  → {ws1, ws2, ...}
    "anomaly:{session_id}" → {ws3, ws4, ...}
    "health"               → {ws1, ws2, ws3, ws4, ...}
```

**Subscribe**: When a client sends `camera.subscribe {camera_id}`, the manager
adds the WebSocket to `rooms["camera:{camera_id}"]`.

**Broadcast**: When the detection pipeline produces a frame for `camera_id`, it
calls `manager.broadcast("camera:{camera_id}", message)` which iterates the set
and sends to each WebSocket concurrently via `asyncio.gather()`.

**Fan-out cost**: For a room with N subscribers, broadcast is N concurrent
`ws.send_text()` calls. At N=20 (worst case: all instructors watching one
camera), this is 20 async sends — trivial for asyncio.

#### Advantages
- **Zero external dependencies.** No Redis required for WebSocket routing.
- **Minimal latency.** In-process dict lookup + async send. No network hop to
  a broker.
- **Simple lifecycle.** `on_connect` adds to rooms, `on_disconnect` removes.
  Python `set` operations are O(1).
- **Debuggable.** Can inspect room membership via a health endpoint.

#### Limitations
- **Single-process only.** If the FastAPI app scales to multiple Uvicorn workers,
  each worker has its own `ConnectionManager` — a client connected to worker 1
  won't receive broadcasts from worker 2.
- **Mitigation**: For initial deployment (20 users), a single Uvicorn process
  with async concurrency is sufficient. FastAPI/Uvicorn handles thousands of
  concurrent WebSocket connections on a single process.

### Option B: Redis Pub/Sub Broker

Each Uvicorn worker subscribes to Redis channels (`camera:{camera_id}`). The
detection pipeline publishes to Redis. Each worker receives the message and
broadcasts to its local WebSocket connections.

#### Advantages
- **Multi-worker safe.** Any number of Uvicorn workers can share the same
  pub/sub topology.
- **Decouples producer/consumer.** The detection pipeline only knows about
  Redis, not about WebSocket connections.

#### Limitations
- **Added latency.** Publish → Redis → Subscribe → ws.send adds ~0.5–2ms
  network round-trip. Acceptable but unnecessary at this scale.
- **Added complexity.** Requires Redis dependency, `aioredis`/`redis.asyncio`
  subscription management, reconnection handling for the Redis subscriber.
- **Added failure mode.** Redis outage = total WebSocket blackout. In-process
  manager has no such dependency.

### Option C: Broadcaster Library (encode/broadcaster)

The `broadcaster` Python library provides a clean pub/sub abstraction over
Redis, PostgreSQL LISTEN/NOTIFY, or in-memory backends. It integrates with
Starlette's WebSocket lifecycle.

#### Assessment
- Clean API: `broadcast.subscribe("camera:123")` / `broadcast.publish()`.
- Supports backend swap (memory → Redis) via config.
- **Concern**: Small library (~500 GitHub stars), last major update 2023.
  Adds a dependency for functionality achievable with ~80 lines of
  application code. The abstraction it provides is thin.

### Decision Rationale

1. **Start with in-process `ConnectionManager`.** For 20 concurrent users on a
   university LAN, a single Uvicorn process with async concurrency easily handles
   the entire WebSocket load. A detection pipeline running in a separate process
   (or subprocess) communicates with the FastAPI process via an internal mechanism
   (e.g., `asyncio.Queue` fed by a thread reading from the pipeline, or a simple
   Redis publish if the pipeline is a separate service).

2. **Design the `ConnectionManager` interface to be swappable.** Define an
   abstract `BaseBroadcaster` with `subscribe(room, ws)`, `unsubscribe(room, ws)`,
   `broadcast(room, message)`. The in-process implementation is the default.
   A `RedisBroadcaster` implementation can be swapped in via config if horizontal
   scaling is ever needed. This follows the Open/Closed principle from the
   security standards.

3. **Room naming convention**: Use namespaced string keys:
   - `camera:{camera_id}` — detection frames for a specific camera
   - `session:{session_id}:anomalies` — anomaly alerts for a monitoring session
   - `health` — global health updates (all connected clients)

4. **Detection pipeline → WebSocket bridge**: The YOLO detection pipeline runs
   CPU-bound inference (likely in a separate process via `multiprocessing` per
   the security standards §5). It must cross a process boundary to reach the
   async WebSocket layer. Two clean approaches:

   - **Option A (same host)**: Pipeline writes to a `multiprocessing.Queue` or
     `asyncio.Queue` (via `janus` bridge library). A background `asyncio.Task`
     in the FastAPI process reads from the queue and calls
     `manager.broadcast()`. Lowest latency (~0 ms overhead).

   - **Option B (separate service)**: Pipeline publishes to Redis pub/sub. The
     FastAPI process subscribes. Cleaner process separation but adds Redis
     as a runtime dependency for the WebSocket path.

   **Recommendation**: Option A for single-host deployment (grad project scope),
   with the `BaseBroadcaster` abstraction allowing Option B as a config switch.

### Subscription Flow

```
Client connects → ws://{host}/ws/detections/{session_id}
    │
    ▼
FastAPI WebSocket endpoint authenticates via JWT token (query param or first message)
    │
    ▼
Client sends: { "type": "camera.subscribe", "camera_id": "uuid-1" }
    │
    ▼
ConnectionManager.subscribe("camera:uuid-1", websocket)
    │
    ▼
Client starts receiving detection.frame messages for camera uuid-1
    │
    ...
    │
Client sends: { "type": "camera.unsubscribe", "camera_id": "uuid-1" }
    │
    ▼
ConnectionManager.unsubscribe("camera:uuid-1", websocket)
```

### Fan-Out Performance Estimate

| Scenario | Rooms | Subscribers/Room | Broadcasts/sec | Total sends/sec |
|----------|-------|-------------------|-----------------|------------------|
| Low (5 instructors, 2 cams each, 5 unique cams) | 5 | avg 2 | 5×15=75 | 150 |
| Medium (10 instructors, 4 cams, 15 unique) | 15 | avg 2.7 | 15×15=225 | 607 |
| High (20 instructors, 8 cams, 30 unique) | 30 | avg 5.3 | 30×30=900 | 4,800 |
| Extreme (20 instructors ALL watch same camera) | 1 | 20 | 1×30=30 | 600 |

Even the "high" scenario at 4,800 async sends/second is well within a single
asyncio event loop's capacity (benchmarks show >50,000 small WebSocket sends/sec
on modern hardware).

---

## Decision 3: Connection Lifecycle — Reconnection, Heartbeat, Stale Cleanup

### Recommendation: **Application-level ping/pong heartbeat + exponential backoff reconnection + server-side stale reaper**

### 3.1 Heartbeat / Keep-Alive

#### Problem

WebSocket connections can silently die — network changes, NAT timeout, laptop
sleep/wake, browser tab suspension. Without active probing, the server holds
stale connections and the client shows stale data.

#### Approach: Dual-Layer Heartbeat

**Layer 1 — WebSocket protocol ping/pong (transport level)**

Starlette's WebSocket implementation supports the WebSocket protocol-level ping
frames. Uvicorn can be configured with `--ws-ping-interval` and
`--ws-ping-timeout` (defaults: 20s interval, 20s timeout). These are handled by
the WebSocket library (not application code) and detect transport-level failures.

- **Uvicorn config**: `--ws-ping-interval 20 --ws-ping-timeout 20`
- If the client doesn't respond to a ping within 20s, Uvicorn closes the
  connection and the FastAPI endpoint receives a `WebSocketDisconnect` exception.

**Layer 2 — Application-level heartbeat (application level)**

The server sends a `heartbeat` message every 30 seconds on the WebSocket. The
client responds with a `heartbeat_ack`. This serves two purposes:

1. **Client-side stale detection**: If the client doesn't receive a heartbeat
   for >45 seconds, it assumes the connection is dead and initiates reconnection
   (even if the TCP socket hasn't timed out yet).

2. **Server-side stale detection**: If the server doesn't receive a
   `heartbeat_ack` within 10 seconds of sending a `heartbeat`, it marks the
   connection as stale and closes it.

```
Server → Client: { "type": "heartbeat", "ts": 1709042400000 }
Client → Server: { "type": "heartbeat_ack", "ts": 1709042400000 }
```

#### Why Both Layers

- Transport-level pings detect TCP failures but NOT application-level freezes
  (e.g., event loop blocked, serialization bug).
- Application-level heartbeats detect end-to-end liveness including application
  health.
- Some corporate proxies/firewalls strip WebSocket ping frames but pass
  application messages. The application heartbeat works through these.

### 3.2 Reconnection Strategy (Client-Side)

The React `useWebSocket` hook implements automatic reconnection:

| Step | Behavior |
|------|----------|
| 1 | On unexpected disconnect (not user-initiated close), wait **1 second**. |
| 2 | Attempt reconnection. If it fails, **exponential backoff**: 1s → 2s → 4s → 8s → 16s → 30s (cap). |
| 3 | Add **±20% jitter** to each backoff interval to prevent thundering herd when many clients reconnect simultaneously (e.g., after server restart). |
| 4 | After **10 consecutive failures**, stop auto-reconnecting. Show a persistent "Connection Lost" banner with a manual "Retry" button. |
| 5 | On successful reconnect, **re-subscribe** to all previously subscribed camera rooms by replaying `camera.subscribe` messages. |
| 6 | On successful reconnect, **request state snapshot** — the server sends the latest detection frame for each subscribed camera so the UI doesn't show stale bounding boxes while waiting for the next pipeline frame. |

#### Backoff Timing Table

| Attempt | Base Delay | With Jitter (±20%) | Cumulative Wait |
|---------|------------|---------------------|-----------------|
| 1 | 1s | 0.8–1.2s | ~1s |
| 2 | 2s | 1.6–2.4s | ~3s |
| 3 | 4s | 3.2–4.8s | ~7s |
| 4 | 8s | 6.4–9.6s | ~15s |
| 5 | 16s | 12.8–19.2s | ~31s |
| 6–10 | 30s | 24–36s | ~2.5–4.5 min |

This matches the existing websocket-api.md reconnection spec but adds jitter
(missing from the original spec) and a state snapshot on reconnect.

### 3.3 Stale Connection Cleanup (Server-Side)

A background `asyncio.Task` runs every 30 seconds in the FastAPI process:

1. **Iterate all connections** in the `ConnectionManager`.
2. **Check last `heartbeat_ack` timestamp** for each connection.
3. If a connection hasn't responded in >60 seconds, **force-close** it
   (`await ws.close(code=4008, reason="Stale connection")`).
4. **Remove from all rooms** and log the event.

This prevents resource leaks from half-open connections where the TCP socket
hasn't timed out but the client is gone (mobile browser backgrounded, laptop
lid closed, etc.).

### 3.4 Authentication on Connect

```
Client → Server: wss://{host}/ws/detections/{session_id}?token={jwt}
```

1. FastAPI WebSocket endpoint extracts the JWT from the query parameter.
2. Validates token signature, expiration, and instructor identity.
3. Verifies the instructor owns or has access to the session.
4. If valid: accepts the WebSocket (`await ws.accept()`).
5. If invalid: closes with `4001` (auth failed) or `4003` (forbidden).

**Why query param, not header**: Browser `WebSocket()` API does not support
custom headers. The standard workaround is JWT in query string or first-message
auth. Query param is simpler (auth happens before `accept()`, rejecting
unauthenticated connections immediately). The JWT is short-lived (15 min) and
transmitted over WSS (TLS).

### 3.5 Graceful Shutdown

When the FastAPI server shuts down (`SIGTERM`/`SIGINT`):

1. Stop accepting new WebSocket connections.
2. Send `{ "type": "server.shutdown", "reason": "Server restarting" }` to all
   connected clients.
3. Close all WebSocket connections with code `1001` (Going Away).
4. Wait up to 5 seconds for in-flight broadcasts to complete.
5. Clients receive the close frame and begin reconnection with backoff.

FastAPI's `@app.on_event("shutdown")` or lifespan context manager handles this.

---

## Decision 4: Backpressure Handling

### Recommendation: **Frame decimation with per-client send queues and drop-oldest policy**

### Problem

The detection pipeline produces frames at 15–30 FPS per camera. A "slow client"
(poor network, overloaded browser tab, mobile device on Wi-Fi) may not be able
to consume messages at the production rate. Without backpressure handling:

- Server-side send buffers grow unbounded → memory exhaustion.
- Client receives a burst of stale frames when it catches up → visual jitter,
  wasted bandwidth on obsolete data.
- `await ws.send_text()` blocks the event loop if the OS TCP send buffer is
  full → other clients' broadcasts are delayed.

### Strategy: Multi-Layer Backpressure

#### Layer 1: Per-Client Async Send Queue with Bounded Buffer

Instead of calling `ws.send_text()` directly in the broadcast loop, each
WebSocket connection gets an `asyncio.Queue(maxsize=N)` (where N = 2–5 per
camera subscription). A dedicated per-client sender task drains the queue:

```
broadcast("camera:X", msg)
    │
    for each ws in room:
        try:
            ws.send_queue.put_nowait(msg)
        except QueueFull:
            # Drop oldest message in queue, insert new one
            ws.send_queue.get_nowait()  # discard oldest
            ws.send_queue.put_nowait(msg)  # insert newest
```

**Drop-oldest policy**: When the queue is full, the oldest (most stale) frame
is discarded and the newest frame is enqueued. The client always receives the
most recent data available, even if intermediate frames are lost. For a
real-time monitoring UI, showing the latest bounding boxes is always preferable
to delivering a complete sequence with delay.

**Queue size rationale**: `maxsize=3` per camera subscription means a client can
be up to 3 frames behind (~100–200ms at 30 FPS). Beyond that, frames are dropped
to keep the client near real-time.

#### Layer 2: Server-Side Frame Decimation (Adaptive Rate)

Not all clients need 30 FPS detection updates. Bounding box positions change
slowly (a student moves ~1–5 pixels per frame). The `ConnectionManager` can
track each client's effective consumption rate and decimate accordingly:

- **Subscription rate parameter**: When subscribing, the client can request
  a target rate: `{ "type": "camera.subscribe", "camera_id": "...", "max_fps": 10 }`
- **Server-side thinning**: The broadcast path keeps a per-room, per-client
  `last_sent` timestamp. If less than `1/max_fps` seconds have elapsed,
  the message is skipped (not even queued).
- **Default**: 15 FPS (every other frame at 30 FPS pipeline rate).

This is the most effective backpressure mechanism because it reduces load at the
source. A client viewing 8 cameras at 10 FPS each receives 80 msg/s instead of
240 msg/s — a 3× reduction with negligible visual impact.

#### Layer 3: Client-Side Frame Dropping

Even with server-side thinning, the client should be resilient to bursts:

- The `useWebSocket` `onmessage` handler writes incoming detection frames into
  a Zustand store keyed by `camera_id`.
- **Only the latest frame per camera is stored.** If two frames arrive between
  `requestAnimationFrame` ticks, only the latest is rendered.
- The Canvas render loop (`requestAnimationFrame`) reads the latest frame from
  the store and draws bounding boxes. It never queues frames — it always renders
  the freshest available data.

This means the client naturally skips frames it can't render in time. No explicit
drop logic needed — the store overwrite IS the drop.

#### Layer 4: Slow Client Detection and Degradation

If a client's send queue is consistently full (>90% full for >5 seconds), the
server takes graduated action:

| Level | Condition | Action |
|-------|-----------|--------|
| 1 | Queue >80% full for 3s | Reduce server-side rate to 10 FPS for this client |
| 2 | Queue >90% full for 5s | Reduce to 5 FPS |
| 3 | Queue >95% full for 10s | Reduce to 2 FPS + send `{ "type": "backpressure.warning", "message": "Slow connection detected. Reducing update rate." }` |
| 4 | Queue full + send fails for 30s | Disconnect with `4009` — `"Connection too slow"` |

This prevents a single slow client from degrading service for others while
giving the user feedback about their connection quality.

### Alternatives Considered

| Alternative | Why Rejected |
|-------------|-------------|
| **TCP backpressure only** (let OS buffer fill, `send()` blocks) | Blocks enitre broadcast for all clients in the room. One slow client degrades everyone. Unacceptable. |
| **Unbounded send queue** | Memory grows linearly with client slowness. 30 FPS × 600 bytes × 60 seconds = 1 MB/min per camera per slow client. With 8 cameras, a stuck client accumulates 8 MB/min. Eventually OOM. |
| **WebSocket per camera** (separate connection per camera) | Multiplies connection count: 20 instructors × 8 cameras = 160 WebSocket connections. Browser limit is typically 6 per domain (HTTP/1.1) or 100 (HTTP/2). Wastes server resources. Management complexity. |
| **Binary diff / delta encoding** | Only send changed detections since last frame. Reduces bandwidth but adds complexity (client must maintain state, handle missed deltas). Not worth it at this message size. |

---

## Decision 5: FastAPI WebSocket Patterns — Connection Manager vs Pub/Sub Broker

### Recommendation: **Custom `ConnectionManager` with abstract `BaseBroadcaster` interface, in-process implementation as default, Redis pub/sub as pluggable backend**

### Pattern Analysis

#### Pattern A: Starlette Raw WebSocket + Application ConnectionManager

FastAPI's WebSocket support is Starlette's `WebSocket` class. The standard
pattern from the FastAPI docs is a `ConnectionManager` class:

```python
# Conceptual structure (not implementation code)
class ConnectionManager:
    def __init__(self):
        self.active_connections: dict[str, set[WebSocket]] = {}

    async def subscribe(self, room: str, ws: WebSocket): ...
    async def unsubscribe(self, room: str, ws: WebSocket): ...
    async def broadcast(self, room: str, message: dict): ...
    async def disconnect(self, ws: WebSocket): ...
```

**Starlette's WebSocket lifecycle in FastAPI**:

```python
@app.websocket("/ws/detections/{session_id}")
async def detection_ws(websocket: WebSocket, session_id: str):
    await websocket.accept()
    manager.register(websocket)
    try:
        while True:
            data = await websocket.receive_json()
            # Handle subscribe/unsubscribe/heartbeat
    except WebSocketDisconnect:
        manager.disconnect(websocket)
```

**Key Starlette behaviors to account for**:
- `websocket.receive_*()` is the blocking read. It raises
  `WebSocketDisconnect` when the client disconnects.
- `websocket.send_*()` is non-blocking (writes to OS buffer) but can raise
  `RuntimeError` if the connection is closed.
- There is no built-in pub/sub, rooms, or broadcast — all application-level.
- Connection state is per-worker (not shared across Uvicorn workers).

#### Pattern B: Redis Pub/Sub as Backbone

```
Detection Pipeline → redis.publish("camera:{id}", msg)
    │
    ▼
FastAPI worker (subscriber) → redis.subscribe("camera:{id}")
    │
    ▼
ConnectionManager.broadcast(local WebSocket connections)
```

Each FastAPI worker runs an `asyncio.Task` that subscribes to relevant Redis
channels via `redis.asyncio`. When a message arrives from Redis, it broadcasts
to locally-connected WebSocket clients.

**Assessment**:
- Necessary when running multiple Uvicorn workers.
- Adds ~0.5–2 ms latency per message (Redis round-trip).
- Adds Redis as a runtime dependency for the WebSocket path.
- `redis.asyncio` `PubSub` object needs careful lifecycle management
  (subscribe, listen loop, reconnect on Redis failure).

#### Pattern C: Third-Party Libraries

| Library | Stars | Approach | Assessment |
|---------|-------|----------|-----------|
| `broadcaster` | ~800 | Abstract pub/sub over Redis/Postgres/memory | Thin wrapper. Last updated 2023. Adds dependency for ~80 lines of code. |
| `fastapi-websocket-pubsub` | ~400 | Built-in pub/sub with topics | Opinionated, adds RPC semantics we don't need. Extra dependency. |
| `socketio` (python-socketio) | ~4k | Socket.IO protocol over WebSocket | Heavy — adds its own protocol, rooms, namespaces. Requires `socket.io-client` on frontend. Overkill. |

**Assessment**: None of these libraries add enough value to justify the
dependency. The `ConnectionManager` pattern with rooms is ~100–150 lines of
straightforward async Python. Owning this code means full control over
backpressure, lifecycle, and debugging.

### Recommended Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    FastAPI Process                        │
│                                                          │
│  ┌────────────────┐    ┌──────────────────────────────┐  │
│  │ Detection       │    │ ConnectionManager             │  │
│  │ Pipeline        │───▶│                              │  │
│  │ (subprocess /   │    │  rooms:                      │  │
│  │  thread pool)   │    │    "camera:A" → {ws1, ws2}   │  │
│  │                 │    │    "camera:B" → {ws3}         │  │
│  │ Produces frames │    │    "health"   → {ws1,ws2,ws3} │  │
│  │ → Queue         │    │                              │  │
│  └────────────────┘    │  Per-client send queues:      │  │
│                        │    ws1 → Queue(maxsize=3)     │  │
│  ┌────────────────┐    │    ws2 → Queue(maxsize=3)     │  │
│  │ Queue Consumer  │───▶│    ws3 → Queue(maxsize=3)     │  │
│  │ (asyncio Task)  │    │                              │  │
│  │ reads from      │    │  Broadcast:                  │  │
│  │ pipeline queue,  │    │    put_nowait per subscriber │  │
│  │ calls broadcast │    │    Dedicated sender tasks     │  │
│  └────────────────┘    │    drain queues → ws.send()   │  │
│                        └──────────────────────────────┘  │
│                                                          │
│  ┌────────────────┐    ┌──────────────────────────────┐  │
│  │ WebSocket       │    │ Background Tasks              │  │
│  │ Endpoints       │    │                              │  │
│  │ /ws/detections/ │    │  - Heartbeat sender (30s)    │  │
│  │ /ws/anomalies/  │    │  - Stale reaper (30s)        │  │
│  │ /ws/health/     │    │  - Health publisher (10s)    │  │
│  └────────────────┘    └──────────────────────────────┘  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Interface Design (Open/Closed Principle)

```
BaseBroadcaster (abstract)
    ├── subscribe(room, ws) → None
    ├── unsubscribe(room, ws) → None
    ├── broadcast(room, message) → None
    ├── disconnect(ws) → None          # Remove from all rooms
    └── get_room_members(room) → set   # For health/debugging

InProcessBroadcaster(BaseBroadcaster)    ← Default, no dependencies
RedisBroadcaster(BaseBroadcaster)        ← Pluggable when multi-worker needed
```

Selected via config: `BROADCASTER_BACKEND=in_process` or `BROADCASTER_BACKEND=redis`.
Injected into the WebSocket endpoints via FastAPI's dependency injection.

### Single vs Multiple WebSocket Endpoints

**Option A: Single multiplexed endpoint** — `ws://{host}/ws/`

All message types (detections, anomalies, health) flow through one WebSocket
connection. Type-based routing on both sides.

**Option B: Multiple purpose-specific endpoints** — as defined in websocket-api.md

- `ws://{host}/ws/detections/{session_id}/`
- `ws://{host}/ws/anomalies/{session_id}/`
- `ws://{host}/ws/health/`

**Recommendation: Single multiplexed endpoint.**

Rationale:
1. **Fewer connections.** One WebSocket per client instead of 3. Simpler
   lifecycle, one reconnection path, one heartbeat.
2. **Atomic subscription management.** Subscribe/unsubscribe to cameras on
   the same connection that receives anomalies and health updates.
3. **Reduced server resource usage.** Each WebSocket connection consumes a
   file descriptor and memory for buffers. 20 clients × 1 connection = 20 fd
   vs 20 × 3 = 60 fd.
4. **Simpler client code.** One `useWebSocket` hook, one reconnection handler.
   Message dispatch by `type` field is trivial.
5. **The websocket-api.md contract's `type` field already enables multiplexing.**
   All messages have a `type` field — the routing infrastructure is already in
   the protocol.

Updated endpoint: `ws://{host}/ws/monitor/{session_id}/` — single connection
per monitoring session carrying all message types.

### Alternatives Rejected

| Alternative | Why Rejected |
|-------------|-------------|
| **Socket.IO** | Adds protocol overhead, requires matching client library, room semantics we can build simpler. Large dependency. |
| **Multiple endpoints per camera** | Connection explosion: 20 users × 8 cameras = 160 connections. Browser limits, management complexity. |
| **Server-Sent Events + WebSocket hybrid** | SSE for server→client (detections), WebSocket for client→server (subscribe, triage). Two transport mechanisms to maintain. No benefit. |
| **GraphQL subscriptions** | Massive complexity overhead (Apollo Server, schema definition) for simple pub/sub. WebSocket transport underneath anyway. |

---

## Summary of All Decisions

| # | Topic | Decision | Key Rationale |
|---|-------|----------|---------------|
| 1 | Message Format | **JSON** (via `orjson`) with compact field aliases | Debuggable, zero-dependency on frontend, <3 MB/s bandwidth at worst case. MessagePack upgrade path preserved. |
| 2 | Channel/Room Pattern | **In-process `ConnectionManager`** with camera-keyed rooms | Single-process sufficient for 20 users. Abstract `BaseBroadcaster` interface allows Redis swap. Zero external dependency. |
| 3 | Connection Lifecycle | **Dual heartbeat** (transport + app), **exponential backoff + jitter**, **server-side stale reaper** | Detects silent failures at both layers. Jitter prevents thundering herd. Stale reaper prevents resource leaks. |
| 4 | Backpressure | **Bounded per-client send queue** (drop-oldest) + **server-side frame decimation** + **client-side latest-only store** | Prevents slow clients from degrading others. Graceful degradation with user feedback. |
| 5 | FastAPI Pattern | **Custom `ConnectionManager`** with `BaseBroadcaster` abstraction, **single multiplexed endpoint** | Full control over lifecycle/backpressure. Interface segregation for future Redis swap. Single connection reduces complexity. |

---

## Impact on Existing Contracts

This research recommends changes to [websocket-api.md](contracts/websocket-api.md):

| Change | From (Current) | To (Recommended) |
|--------|-----------------|-------------------|
| Framework | Django Channels | FastAPI + Starlette WebSocket (aligns with plan.md) |
| Endpoints | 4 separate endpoints | Single multiplexed `ws://{host}/ws/monitor/{session_id}/` |
| Auth | Session cookie on handshake | JWT token in query parameter (`?token=...`) |
| Channel layer | Redis channel layer (mandatory) | In-process ConnectionManager (default), Redis (optional) |
| Heartbeat | Not specified | Dual: transport ping 20s + app heartbeat 30s |
| Backpressure | Not specified | Per-client bounded queue + frame decimation |
| Reconnect jitter | Not specified | ±20% jitter on backoff intervals |
| State snapshot | Not specified | Latest frame per camera on reconnect |
| Client rate control | Not specified | `max_fps` parameter on `camera.subscribe` |

These changes should be applied to websocket-api.md after review and approval.
