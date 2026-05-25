# Research: Three-Stage RTSP URL Validation Probe

**Feature**: 002-fix-rtsp-camera-feed | **Date**: 2026-03-04  
**Spec**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md)

This document researches the implementation of a three-stage RTSP URL validation probe for the Django backend. The probe validates that an RTSP camera is reachable, responds to the RTSP protocol, and actually produces decodable video data — before the stream is registered with the go2rtc media relay.

---

## Context & Constraints

| Parameter | Value | Source |
|-----------|-------|--------|
| Total probe timeout | ≤10 seconds (configurable) | spec.md FR-002 |
| Python version | 3.10+ | plan.md |
| Existing CV dependency | `ultralytics==8.3.61` (pulls in `opencv-python`) | requirements.txt |
| Execution context | Django/Celery (sync task or sync service call) | plan.md |
| Supported schemes | `rtsp://` and `rtsps://` | spec.md FR-012 |
| Error granularity | Distinct error per failed stage | spec.md FR-002, FR-003 |
| Credential handling | Encrypted at rest, masked in logs/UI | spec.md FR-015 |

---

## Stage Definitions

| Stage | Purpose | Input | Success Condition | Typical Duration |
|-------|---------|-------|-------------------|------------------|
| **1 — TCP Connect** | Verify host:port is reachable | Parsed host + port from URL | TCP socket connects within timeout | 50–500 ms (LAN), 1–3 s (WAN/timeout) |
| **2 — RTSP DESCRIBE** | Verify RTSP server responds with valid stream metadata | Full RTSP URL | HTTP-like 200 OK response with SDP body containing at least one `m=video` line | 100–500 ms |
| **3 — Frame Decode** | Confirm video data is actually being produced | Full RTSP URL | At least one video frame successfully decoded | 500–3000 ms |

**Total expected duration (happy path)**: 1–4 seconds.  
**Total expected duration (failure at stage 1)**: configurable connect timeout (default 3s).

---

## 1. Library Comparison

### Option A: Raw TCP Socket + Hand-Crafted RTSP DESCRIBE

**How it works**: Use Python's `socket` module for TCP connect. Build a raw RTSP `DESCRIBE` request string (`DESCRIBE rtsp://... RTSP/1.0\r\nCSeq: 1\r\n...`), send over the socket, parse the response status code and SDP body manually.

| Dimension | Assessment |
|-----------|-----------|
| Stage 1 (TCP) | **Native.** `socket.create_connection((host, port), timeout=N)`. Zero dependencies. |
| Stage 2 (DESCRIBE) | **Manual implementation.** Must construct RTSP request headers, handle `WWW-Authenticate` challenges (Digest auth is common for IP cameras), parse response codes, extract SDP. RTSP Digest auth requires implementing RFC 2617 — non-trivial. |
| Stage 3 (Frame) | **Not possible.** Raw socket cannot decode RTP/H.264/H.265 video frames. Would need a separate library for this stage anyway. |
| `rtsps://` support | Must wrap socket in `ssl.SSLContext` manually. Doable but adds complexity (cert verification, TLS version negotiation). |
| Auth support | Must implement RTSP Basic + Digest auth manually (RFC 2326 + RFC 2617). Digest auth requires MD5/SHA-256 challenge-response. This is the most complex part. |
| Dependencies | None (stdlib only) for stages 1–2. |
| Maintenance | High. RTSP is a stateful protocol with many edge cases (session management, interleaved RTP, digest auth nonces). IP camera firmware varies widely in compliance. |
| Reliability | **Low-Medium.** Camera firmware often deviates from the RTSP spec. A hand-rolled parser will hit edge cases that battle-tested libraries have already solved. |

**Verdict**: Good for Stage 1 only. Too fragile and complex for Stage 2 (auth), impossible for Stage 3.

---

### Option B: `ffprobe` Subprocess

**How it works**: Shell out to `ffprobe` (part of FFmpeg) with RTSP-specific options. Parse JSON/XML output for stream metadata.

```bash
ffprobe -v quiet -print_format json -show_streams -rtsp_transport tcp \
  -timeout 5000000 "rtsp://user:pass@192.168.1.10:554/stream"
```

| Dimension | Assessment |
|-----------|-----------|
| Stage 1 (TCP) | **Implicit.** `ffprobe` performs TCP connect internally. But failure mode is a generic timeout or connection error — not a distinct "TCP unreachable" error. Would need a separate TCP check if distinct Stage 1 errors are required. |
| Stage 2 (DESCRIBE) | **Implicit.** `ffprobe` sends DESCRIBE internally to get stream info. Returns stream metadata (codec, resolution, framerate) in structured JSON. Auth errors manifest as specific error strings in stderr. |
| Stage 3 (Frame) | **Requires `ffmpeg` instead of `ffprobe`.** `ffprobe` only inspects metadata, it does not decode frames. Could use `ffmpeg -frames:v 1` to decode one frame, but that's a separate subprocess call. |
| `rtsps://` support | **Full support.** FFmpeg handles `rtsps://` natively with TLS negotiation. Pass `-rtsp_transport tcp` for interleaved mode. |
| Auth support | **Full support.** FFmpeg handles Basic, Digest, and NTLM auth from credentials embedded in the URL. |
| Dependencies | Requires `ffprobe`/`ffmpeg` binary on the system PATH. Not a Python package — must be installed at the OS/container level. Already likely present since `ultralytics` uses FFmpeg internally. |
| Error parsing | Must parse stderr text for error classification. Error messages are human-readable but not structured — requires regex matching on strings like `401 Unauthorized`, `Connection refused`, `Server returned 404`. Fragile across FFmpeg versions. |
| Performance | Subprocess spawn adds ~50–100 ms overhead. Total probe: 2–5 s. Two subprocess calls needed if Stage 3 is separate. |
| Timeout control | `-timeout` flag (in microseconds) controls RTSP negotiation timeout. `-analyzeduration` and `-probesize` control how long `ffprobe` waits for stream data. |
| Security | Credentials visible in process list (`ps aux`). Must take care with `/proc` exposure in containers. Can mitigate by passing URL via stdin pipe or environment variable, but this is non-trivial with `ffprobe`. |

**Verdict**: Powerful but requires external binary, stderr parsing is fragile, credentials leak in process args, and separating Stage 1 from Stage 2 requires a manual TCP step anyway.

---

### Option C: `av` (PyAV — FFmpeg Python Bindings)

**How it works**: PyAV (`av` package) provides Pythonic bindings to FFmpeg's libavformat/libavcodec. Open an RTSP URL as a container, inspect streams, and optionally decode frames — all in-process.

```python
import av

container = av.open(
    "rtsp://user:pass@192.168.1.10:554/stream",
    options={"rtsp_transport": "tcp", "stimeout": "5000000"},
    timeout=10.0,
)
# Stage 2: inspect streams
video_streams = [s for s in container.streams if s.type == "video"]
# Stage 3: decode one frame
for frame in container.decode(video=0):
    break  # got one frame
container.close()
```

| Dimension | Assessment |
|-----------|-----------|
| Stage 1 (TCP) | **Implicit.** `av.open()` performs TCP connect internally. Connection errors raise `av.error.ConnectionRefusedError` or `av.error.TimeoutError`. But no separate "TCP unreachable" vs "RTSP error" distinction from `av.open()` alone — would still want a manual TCP check for Stage 1. |
| Stage 2 (DESCRIBE) | **Excellent.** After `av.open()`, `container.streams` contains the parsed SDP metadata (codec, resolution, framerate, etc.). Auth failures raise `av.error.HTTPUnauthorizedError` (maps to FFmpeg's 401 handling). |
| Stage 3 (Frame) | **Excellent.** `container.decode(video=0)` decodes the first video frame. Returns a `VideoFrame` object with pixel data. This is the definitive proof the stream works. |
| `rtsps://` support | **Full support.** Uses FFmpeg's TLS implementation under the hood. Same `rtsps://` handling as CLI FFmpeg. |
| Auth support | **Full support.** Inherits FFmpeg's auth handling (Basic, Digest) from URL credentials. |
| Dependencies | `av` package (PyAV). ~15 MB wheel that bundles FFmpeg libraries. **Not currently in requirements.txt.** Would add a new dependency. However, `ultralytics` already bundles/uses OpenCV which also links to FFmpeg — no actual system FFmpeg conflict. |
| Error handling | Raises Python exceptions that map to FFmpeg error codes. Much better than parsing stderr strings. `av.error.FileNotFoundError` (404), `av.error.ConnectionRefusedError`, `av.error.InvalidDataError`, etc. |
| Performance | In-process FFmpeg — no subprocess overhead. `av.open()` → stream inspection → one frame decode: **1–3 seconds** total on LAN. |
| Timeout control | `timeout` parameter on `av.open()` for the overall open operation. `options={"stimeout": "5000000"}` (microseconds) for RTSP socket-level timeout. |
| Resource cleanup | Must explicitly call `container.close()` or use context manager. If forgotten, holds the RTSP connection open. Use `try/finally`. |
| Thread safety | FFmpeg operations are **not thread-safe** per container. Each probe must use its own container instance. Fine for our use case (one probe per connect request). |

**Verdict**: Best programmatic access to FFmpeg's full RTSP stack. Clean exception hierarchy. But adds a new dependency. Stage 1 still benefits from a separate TCP check.

---

### Option D: `opencv-python` (`cv2.VideoCapture`) — **RECOMMENDED**

**How it works**: OpenCV's `VideoCapture` can open RTSP URLs using its FFmpeg backend. The `read()` method decodes frames.

```python
import cv2

cap = cv2.VideoCapture("rtsp://user:pass@192.168.1.10:554/stream", cv2.CAP_FFMPEG)
cap.set(cv2.CAP_PROP_OPEN_TIMEOUT_MSEC, 5000)
cap.set(cv2.CAP_PROP_READ_TIMEOUT_MSEC, 5000)

if not cap.isOpened():
    raise ConnectionError("Failed to open RTSP stream")

# Stream metadata
width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
fps = cap.get(cv2.CAP_PROP_FPS)

# Stage 3: read one frame
ret, frame = cap.read()
cap.release()

if not ret or frame is None:
    raise ConnectionError("No video data received")
```

| Dimension | Assessment |
|-----------|-----------|
| Stage 1 (TCP) | **Implicit.** `VideoCapture` performs TCP connect internally. A dedicated TCP check is still preferable for Stage 1 to give a distinct "network unreachable" error before attempting the heavier RTSP open. |
| Stage 2 (DESCRIBE) | **Implicit.** `cap.isOpened()` returns `True` only if RTSP DESCRIBE + SETUP succeeded. Stream metadata available via `cap.get()` properties (width, height, fps, codec fourcc). |
| Stage 3 (Frame) | **Excellent.** `cap.read()` decodes and returns the first frame as a numpy array. This is identical to what the detection pipeline uses, so if the probe succeeds, the pipeline will too. |
| `rtsps://` support | **Full support.** OpenCV's FFmpeg backend handles `rtsps://` natively. Requires OpenCV built with FFmpeg + TLS support (the default `opencv-python` pip package includes this). |
| Auth support | **Full support.** Inherits FFmpeg's auth handling from URL credentials (Basic, Digest). |
| Dependencies | **Already installed.** `ultralytics==8.3.61` depends on `opencv-python`. Zero new dependencies. |
| Error handling | **Limited.** `cap.isOpened()` returns a boolean — no exception, no error code, no distinction between "auth failed" and "host unreachable." `cap.read()` returns `(False, None)` on failure — again no reason. This is the main weakness. **Mitigation**: Use the separate TCP check (Stage 1) for network errors, and `cv2.setLogLevel()` + log capture for more detail. |
| Performance | Same FFmpeg backend as PyAV. `VideoCapture` open + one `read()`: **1–3 seconds** on LAN. |
| Timeout control | `CAP_PROP_OPEN_TIMEOUT_MSEC` (available in OpenCV ≥4.6) controls how long `VideoCapture()` waits during open. `CAP_PROP_READ_TIMEOUT_MSEC` controls `read()` timeout. **Important**: These properties may not work with all backends. Always wrap with an outer `threading.Timer` or `signal.alarm` as a safety net. |
| Resource cleanup | `cap.release()` frees the connection. Must be in a `finally` block. |
| Thread safety | `VideoCapture` is thread-safe per instance. Each probe gets its own instance. |

**Verdict**: Zero new dependencies (already bundled with ultralytics). Matches the exact code path the detection pipeline uses. Weak error granularity mitigated by the dedicated TCP Stage 1 and careful timeout handling.

---

## 2. Recommendation

### Use: **Raw Socket (Stage 1) + `cv2.VideoCapture` (Stages 2–3)**

**Rationale**:

1. **Zero new dependencies.** `opencv-python` is already installed via `ultralytics`. No new package to add, audit, version-pin, or build in CI.

2. **Same code path as production.** The detection pipeline will use `cv2.VideoCapture` to read frames from RTSP. If the probe confirms `VideoCapture.read()` works, the pipeline will work. Testing with a different library (e.g., PyAV) could give false positives/negatives.

3. **Stage 1 separation gives clean error classification.** The raw TCP socket check is 10 lines of stdlib code and definitively answers "is the host:port reachable?" before we attempt the heavier OpenCV open. This gives us the distinct "Network unreachable" error that `cv2.VideoCapture` alone cannot provide.

4. **Sufficient for our needs.** The main weakness of OpenCV (poor error detail for Stage 2 failures) is acceptable because:
   - Stage 1 already filters out network-level failures.
   - If Stage 1 passes but `cap.isOpened()` fails, the cause is almost certainly authentication or stream-path issues — a narrow enough failure set to give a useful error message.
   - For `rtsps://`, TLS handshake failures also manifest as `isOpened() == False` after TCP succeeds — the error message can note TLS as a possible cause.

5. **PyAV would be the upgrade path** if we later need:
   - Parsing the exact RTSP error code (401 vs 404 vs 453).
   - Extracting SDP details (audio/video track listing, codec parameters).
   - More granular timeout control.
   - Adding PyAV later is additive (doesn't break the OpenCV approach).

### Why Not PyAV?

PyAV is technically superior for error granularity. However:

- Adds ~15 MB dependency with its own bundled FFmpeg (potential version conflicts with OpenCV's bundled FFmpeg in edge cases).
- The probe is a one-shot validation step — the error granularity difference rarely matters in practice (users care about "does it work?" not "was the 401 Digest-MD5 or Digest-SHA-256?").
- If we later need PyAV for other reasons (e.g., server-side recording, transcoding), it can be added then and the probe upgraded.

### Why Not `ffprobe` Subprocess?

- Credentials visible in process list — security concern (spec FR-015).
- Error parsing via stderr regex is fragile across FFmpeg versions.
- Subprocess overhead (fork + exec) is unnecessary when we have in-process FFmpeg via OpenCV.
- Two separate subprocess calls needed (ffprobe for metadata, ffmpeg for frame decode).

---

## 3. Error Classification

### Mapping probe failures to user-facing error categories:

| Probe Stage | Failure Condition | Error Category | User-Facing Message | Technical Detail (logs only) |
|-------------|-------------------|----------------|---------------------|------------------------------|
| **Stage 1** | `socket.timeout` | `NETWORK_TIMEOUT` | "Camera Unreachable — Connection timed out. Please verify the IP address and that the camera is powered on and connected to the network." | `TCP connect to {host}:{port} timed out after {timeout}s` |
| **Stage 1** | `ConnectionRefusedError` | `NETWORK_REFUSED` | "Camera Unreachable — Connection was actively refused. Please verify the port number and that the RTSP service is running on the camera." | `TCP connect to {host}:{port} refused (RST)` |
| **Stage 1** | `socket.gaierror` | `NETWORK_DNS` | "Camera Unreachable — Could not resolve hostname. Please verify the camera address." | `DNS resolution failed for {host}` |
| **Stage 1** | `OSError` (general) | `NETWORK_ERROR` | "Camera Unreachable — A network error occurred. Please check your network connection and the camera address." | `TCP connect error: {errno} {strerror}` |
| **Stage 2** | `cap.isOpened() == False` after Stage 1 passed | `RTSP_AUTH_OR_PATH` | "RTSP Connection Failed — The camera responded to the network connection but rejected the RTSP request. Please verify the username, password, and stream path in the URL." | `cv2.VideoCapture failed to open {masked_url}` |
| **Stage 2** | Timeout during `VideoCapture()` constructor | `RTSP_TIMEOUT` | "RTSP Connection Timed Out — The camera accepted the network connection but did not complete the RTSP handshake in time." | `VideoCapture open timed out after {timeout}s for {masked_url}` |
| **Stage 3** | `cap.read()` returns `(False, None)` | `NO_VIDEO_DATA` | "No Video Data — The camera connected successfully but is not producing video. The camera may be initializing, misconfigured, or the stream path may be incorrect." | `cap.read() failed for {masked_url}` |
| **Stage 3** | Timeout during `cap.read()` | `VIDEO_TIMEOUT` | "No Video Data — The camera connected but no video frame was received within the timeout period. The camera may be overloaded or the stream may be corrupted." | `Frame decode timed out after {timeout}s for {masked_url}` |
| **Any** | Total probe timeout exceeded | `PROBE_TIMEOUT` | "Connection Validation Timed Out — The camera probe could not complete within the allowed time. Please try again or check camera responsiveness." | `Total probe timeout ({timeout}s) exceeded at stage {stage}` |

### Exception Hierarchy

```python
class CameraConnectionError(Exception):
    """Base exception for all camera connection failures."""
    def __init__(self, category: str, message: str, stage: int, detail: str = ""):
        self.category = category      # e.g., "NETWORK_TIMEOUT"
        self.message = message         # User-facing message
        self.stage = stage             # 1, 2, or 3
        self.detail = detail           # Technical detail for logs
        super().__init__(message)

class NetworkError(CameraConnectionError):
    """Stage 1 failures — TCP-level."""
    pass

class RtspError(CameraConnectionError):
    """Stage 2 failures — RTSP protocol level."""
    pass

class VideoError(CameraConnectionError):
    """Stage 3 failures — frame decode level."""
    pass
```

---

## 4. RTSPS (RTSP-over-TLS) Support

### How Each Library Handles `rtsps://`

| Library | `rtsps://` Support | Mechanism | Certificate Verification | Notes |
|---------|-------------------|-----------|--------------------------|-------|
| **Raw socket** | Manual | Wrap with `ssl.SSLContext.wrap_socket()` | Configurable (`check_hostname`, `verify_mode`) | Must detect `rtsps://` scheme and enable TLS before RTSP handshake |
| **ffprobe/ffmpeg** | Native | FFmpeg's internal TLS (OpenSSL/GnuTLS depending on build) | Off by default, enable with `-tls_verify 1` | Just use `rtsps://` URL directly |
| **PyAV** | Native | Same as FFmpeg (shares TLS backend) | Off by default, enable via `options={"tls_verify": "1"}` | Transparent — `av.open("rtsps://...")` works |
| **OpenCV (`cv2`)** | Native | FFmpeg backend handles TLS | Off by default. Most IP cameras use self-signed certs, so verification is typically disabled. | `cv2.VideoCapture("rtsps://...")` works directly. No special API needed. |

### Stage 1 TCP Check with `rtsps://`

For the dedicated TCP socket check in Stage 1, the scheme determines whether to wrap with TLS:

```python
import socket
import ssl
from urllib.parse import urlparse

def tcp_check(url: str, timeout: float = 3.0) -> None:
    parsed = urlparse(url)
    host = parsed.hostname
    port = parsed.port or (322 if parsed.scheme == "rtsps" else 554)
    
    sock = socket.create_connection((host, port), timeout=timeout)
    
    if parsed.scheme == "rtsps":
        ctx = ssl.create_default_context()
        # Most IP cameras use self-signed certs
        ctx.check_hostname = False
        ctx.verify_mode = ssl.CERT_NONE
        sock = ctx.wrap_socket(sock, server_hostname=host)
    
    sock.close()
```

**Note on default ports**: RTSP uses port 554 by default. RTSPS has no official default port; common conventions are 322 or 554. The spec should require explicit ports in URLs to avoid ambiguity.

### Recommendation

- **For the TCP check (Stage 1)**: Detect scheme and conditionally wrap with TLS. Use `CERT_NONE` for self-signed cert compatibility (IP cameras almost universally use self-signed certs).
- **For Stages 2–3 (OpenCV)**: No special handling needed — `cv2.VideoCapture("rtsps://...")` works transparently.

---

## 5. Security: Handling RTSP URLs with Embedded Credentials

### Threat Model

RTSP URLs often contain credentials: `rtsp://admin:P@ssw0rd@192.168.1.10/stream1`. These must be protected:

| Threat | Mitigation |
|--------|-----------|
| Credentials in logs | **Mask before logging.** Replace `user:pass@` with `***:***@` in all log messages. |
| Credentials in error messages | **Same masking.** User-facing error messages must never contain credentials. |
| Credentials in database | **Encrypt at rest.** Store encrypted in DB, decrypt only at point of use (per spec FR-015). |
| Credentials in memory | **Minimize residence time.** Decrypt → use → clear. Don't cache decrypted URLs in instance variables. |
| Credentials in process list | **Not applicable** for OpenCV (in-process). Would be a concern for `ffprobe` subprocess. |
| Credentials in exception tracebacks | **Sanitize before raising.** Exception messages must use masked URLs. |

### URL Masking Utility

```python
import re
from urllib.parse import urlparse, urlunparse

def mask_rtsp_url(url: str) -> str:
    """Mask credentials in an RTSP URL for safe display/logging.
    
    rtsp://admin:pass@192.168.1.10/stream → rtsp://***:***@192.168.1.10/stream
    rtsp://192.168.1.10/stream             → rtsp://192.168.1.10/stream (no change)
    """
    parsed = urlparse(url)
    if parsed.username or parsed.password:
        # Rebuild with masked credentials
        masked_netloc = f"***:***@{parsed.hostname}"
        if parsed.port:
            masked_netloc += f":{parsed.port}"
        return urlunparse(parsed._replace(netloc=masked_netloc))
    return url
```

### Credential Extraction for Probe

The probe needs the raw credentials to authenticate with the camera. The flow:

1. **Decrypt** the stored RTSP URL (from encrypted DB field).
2. **Pass** the decrypted URL directly to `cv2.VideoCapture()` — OpenCV/FFmpeg extracts credentials from the URL internally.
3. **Mask** the URL before any logging or error message construction.
4. **Release** the decrypted URL from the local variable scope as soon as the probe completes.

```python
def probe_rtsp(encrypted_url: str) -> ProbeResult:
    decrypted_url = decrypt(encrypted_url)  # Point of use
    masked = mask_rtsp_url(decrypted_url)
    try:
        _run_probe_stages(decrypted_url, masked)
    finally:
        del decrypted_url  # Minimize residence time
```

---

## 6. Performance Expectations

### Per-Stage Timing (LAN, Camera on Same Network Segment)

| Stage | Happy Path | Failure (Timeout) | Failure (Immediate) |
|-------|-----------|-------------------|---------------------|
| **Stage 1 — TCP Connect** | 5–50 ms | 3 s (configurable timeout) | <5 ms (connection refused) |
| **Stage 2 — RTSP Open** | 200–800 ms | 5 s (configurable timeout) | 100–500 ms (auth rejected) |
| **Stage 3 — Frame Decode** | 300–2000 ms | 3 s (configurable timeout) | <100 ms (immediate EOF) |
| **Total (happy path)** | **0.5–3 s** | — | — |
| **Total (failure at Stage 1)** | — | **3 s** | **<5 ms** |
| **Total (failure at Stage 2)** | — | **8 s** | **0.1–0.5 s** |
| **Total (failure at Stage 3)** | — | **6–8 s** | **0.5–1 s** |

### Stage 3 First-Frame Latency

The first-frame decode time depends on the codec:

| Codec | First Frame Latency | Reason |
|-------|---------------------|--------|
| MJPEG | 50–200 ms | Each frame is independent (intra-only). Decode starts immediately. |
| H.264 | 300–2000 ms | Must wait for a keyframe (I-frame). GOP interval is typically 1–2 seconds. |
| H.265/HEVC | 500–3000 ms | Similar to H.264 but with potentially longer GOP intervals. |

**Implication**: The Stage 3 timeout should be at least 3 seconds to accommodate H.265 streams with long GOP intervals.

### Budget Allocation

Given the 10-second total timeout (spec requirement), recommended allocation:

| Stage | Timeout | Rationale |
|-------|---------|-----------|
| Stage 1 | 3 s | Generous for WAN; fast failure for unreachable hosts |
| Stage 2 | 4 s | RTSP handshake + potential auth challenge-response |
| Stage 3 | Remaining (10 - elapsed) | At least 3 s for H.265 keyframe wait |

Use a single monotonic clock for the total timeout, with each stage receiving the minimum of its individual allocation and the remaining total budget.

---

## 7. Recommended Implementation Pattern

### Architecture

```
RtspProber (public API)
├── _check_tcp()          → Stage 1: raw socket connect
├── _check_rtsp_open()    → Stage 2: cv2.VideoCapture open + metadata
└── _check_frame_decode() → Stage 3: cv2.VideoCapture.read()
```

### Full Implementation Pattern

```python
"""
RTSP URL Validation Probe — Three-Stage Implementation.

Uses raw TCP socket (Stage 1) + cv2.VideoCapture (Stages 2–3).
Designed for Django/Celery context (sync execution).
"""

import socket
import ssl
import time
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional
from urllib.parse import urlparse

import cv2


# ──────────────────────────────────────────────
# Error Classification
# ──────────────────────────────────────────────

class ProbeStage(int, Enum):
    TCP_CONNECT = 1
    RTSP_DESCRIBE = 2
    FRAME_DECODE = 3


class ProbeErrorCategory(str, Enum):
    NETWORK_TIMEOUT = "network_timeout"
    NETWORK_REFUSED = "network_refused"
    NETWORK_DNS = "network_dns"
    NETWORK_ERROR = "network_error"
    RTSP_AUTH_OR_PATH = "rtsp_auth_or_path"
    RTSP_TIMEOUT = "rtsp_timeout"
    NO_VIDEO_DATA = "no_video_data"
    VIDEO_TIMEOUT = "video_timeout"
    PROBE_TIMEOUT = "probe_timeout"


class CameraConnectionError(Exception):
    """Base exception for camera connection probe failures."""

    def __init__(
        self,
        category: ProbeErrorCategory,
        message: str,
        stage: ProbeStage,
        detail: str = "",
    ):
        self.category = category
        self.message = message
        self.stage = stage
        self.detail = detail
        super().__init__(message)


class NetworkError(CameraConnectionError):
    """Stage 1 — TCP-level failure."""
    pass


class RtspError(CameraConnectionError):
    """Stage 2 — RTSP protocol-level failure."""
    pass


class VideoError(CameraConnectionError):
    """Stage 3 — Video data failure."""
    pass


# ──────────────────────────────────────────────
# Probe Result
# ──────────────────────────────────────────────

@dataclass
class ProbeResult:
    """Result of a successful RTSP probe."""
    success: bool
    width: Optional[int] = None
    height: Optional[int] = None
    fps: Optional[float] = None
    codec: Optional[str] = None
    elapsed_ms: float = 0.0
    stages_completed: list[ProbeStage] = field(default_factory=list)


# ──────────────────────────────────────────────
# URL Utilities
# ──────────────────────────────────────────────

def mask_rtsp_url(url: str) -> str:
    """Replace embedded credentials with '***:***' for safe logging/display."""
    parsed = urlparse(url)
    if not parsed.username and not parsed.password:
        return url
    masked_netloc = f"***:***@{parsed.hostname}"
    if parsed.port:
        masked_netloc += f":{parsed.port}"
    return parsed._replace(netloc=masked_netloc).geturl()


def parse_rtsp_host_port(url: str) -> tuple[str, int]:
    """Extract host and port from an RTSP URL."""
    parsed = urlparse(url)
    host = parsed.hostname
    if not host:
        raise ValueError(f"No hostname in URL: {mask_rtsp_url(url)}")
    default_port = 322 if parsed.scheme == "rtsps" else 554
    port = parsed.port or default_port
    return host, port


# ──────────────────────────────────────────────
# Prober
# ──────────────────────────────────────────────

class RtspProber:
    """
    Three-stage RTSP URL validation probe.

    Stage 1 — TCP Connect: Verify host:port is reachable.
    Stage 2 — RTSP Open: Verify cv2.VideoCapture can open the stream.
    Stage 3 — Frame Decode: Verify at least one video frame is readable.
    """

    DEFAULT_TOTAL_TIMEOUT: float = 10.0
    DEFAULT_TCP_TIMEOUT: float = 3.0

    def __init__(self, total_timeout: float = DEFAULT_TOTAL_TIMEOUT):
        self.total_timeout = total_timeout

    def probe(self, rtsp_url: str) -> ProbeResult:
        """
        Run the full three-stage probe.

        Args:
            rtsp_url: Decrypted RTSP URL (with credentials if needed).

        Returns:
            ProbeResult on success.

        Raises:
            NetworkError: Stage 1 failure.
            RtspError: Stage 2 failure.
            VideoError: Stage 3 failure.
        """
        masked = mask_rtsp_url(rtsp_url)
        start = time.monotonic()
        stages_completed: list[ProbeStage] = []

        def remaining() -> float:
            elapsed = time.monotonic() - start
            left = self.total_timeout - elapsed
            if left <= 0:
                raise CameraConnectionError(
                    category=ProbeErrorCategory.PROBE_TIMEOUT,
                    message=(
                        "Connection Validation Timed Out — The camera probe "
                        "could not complete within the allowed time. "
                        "Please try again or check camera responsiveness."
                    ),
                    stage=ProbeStage(len(stages_completed) + 1),
                    detail=f"Total probe timeout ({self.total_timeout}s) "
                           f"exceeded after stages {stages_completed}",
                )
            return left

        # ── Stage 1: TCP Connect ──
        self._check_tcp(rtsp_url, masked, timeout=min(self.DEFAULT_TCP_TIMEOUT, remaining()))
        stages_completed.append(ProbeStage.TCP_CONNECT)

        # ── Stages 2 & 3: RTSP Open + Frame Decode ──
        result = self._check_video(rtsp_url, masked, timeout=remaining())
        stages_completed.extend([ProbeStage.RTSP_DESCRIBE, ProbeStage.FRAME_DECODE])

        elapsed_ms = (time.monotonic() - start) * 1000
        result.stages_completed = stages_completed
        result.elapsed_ms = elapsed_ms
        return result

    # ── Stage 1 ──

    def _check_tcp(self, url: str, masked: str, timeout: float) -> None:
        """Verify TCP connectivity to the RTSP host:port."""
        host, port = parse_rtsp_host_port(url)
        parsed = urlparse(url)

        try:
            sock = socket.create_connection((host, port), timeout=timeout)
        except socket.timeout:
            raise NetworkError(
                category=ProbeErrorCategory.NETWORK_TIMEOUT,
                message=(
                    "Camera Unreachable — Connection timed out. "
                    "Please verify the IP address and that the camera "
                    "is powered on and connected to the network."
                ),
                stage=ProbeStage.TCP_CONNECT,
                detail=f"TCP connect to {host}:{port} timed out after {timeout:.1f}s",
            )
        except socket.gaierror as exc:
            raise NetworkError(
                category=ProbeErrorCategory.NETWORK_DNS,
                message=(
                    "Camera Unreachable — Could not resolve hostname. "
                    "Please verify the camera address."
                ),
                stage=ProbeStage.TCP_CONNECT,
                detail=f"DNS resolution failed for {host}: {exc}",
            )
        except ConnectionRefusedError:
            raise NetworkError(
                category=ProbeErrorCategory.NETWORK_REFUSED,
                message=(
                    "Camera Unreachable — Connection was actively refused. "
                    "Please verify the port number and that the RTSP service "
                    "is running on the camera."
                ),
                stage=ProbeStage.TCP_CONNECT,
                detail=f"TCP connect to {host}:{port} refused",
            )
        except OSError as exc:
            raise NetworkError(
                category=ProbeErrorCategory.NETWORK_ERROR,
                message=(
                    "Camera Unreachable — A network error occurred. "
                    "Please check your network connection and the camera address."
                ),
                stage=ProbeStage.TCP_CONNECT,
                detail=f"TCP connect error to {host}:{port}: {exc}",
            )

        try:
            # Wrap with TLS for rtsps://
            if parsed.scheme == "rtsps":
                ctx = ssl.create_default_context()
                ctx.check_hostname = False
                ctx.verify_mode = ssl.CERT_NONE  # IP cameras use self-signed certs
                sock = ctx.wrap_socket(sock, server_hostname=host)
        except ssl.SSLError as exc:
            raise NetworkError(
                category=ProbeErrorCategory.NETWORK_ERROR,
                message=(
                    "Camera Unreachable — TLS handshake failed. "
                    "The camera may not support encrypted connections."
                ),
                stage=ProbeStage.TCP_CONNECT,
                detail=f"TLS handshake failed for {host}:{port}: {exc}",
            )
        finally:
            sock.close()

    # ── Stages 2 & 3 ──

    def _check_video(self, url: str, masked: str, timeout: float) -> ProbeResult:
        """
        Open the RTSP stream and decode one frame.

        Combines stages 2 (RTSP open) and 3 (frame decode) because
        cv2.VideoCapture handles both via a single connection.
        """
        timeout_ms = int(timeout * 1000)

        cap = cv2.VideoCapture(url, cv2.CAP_FFMPEG)
        try:
            # Set timeouts (OpenCV ≥4.6)
            cap.set(cv2.CAP_PROP_OPEN_TIMEOUT_MSEC, timeout_ms)
            cap.set(cv2.CAP_PROP_READ_TIMEOUT_MSEC, timeout_ms)

            # Stage 2: Verify RTSP open (DESCRIBE + SETUP)
            if not cap.isOpened():
                raise RtspError(
                    category=ProbeErrorCategory.RTSP_AUTH_OR_PATH,
                    message=(
                        "RTSP Connection Failed — The camera responded "
                        "to the network connection but rejected the RTSP "
                        "request. Please verify the username, password, "
                        "and stream path in the URL."
                    ),
                    stage=ProbeStage.RTSP_DESCRIBE,
                    detail=f"cv2.VideoCapture failed to open {masked}",
                )

            # Extract metadata
            width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
            height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
            fps = cap.get(cv2.CAP_PROP_FPS)
            fourcc_int = int(cap.get(cv2.CAP_PROP_FOURCC))
            codec = (
                "".join(chr((fourcc_int >> 8 * i) & 0xFF) for i in range(4))
                if fourcc_int
                else None
            )

            # Stage 3: Decode one frame
            ret, frame = cap.read()
            if not ret or frame is None:
                raise VideoError(
                    category=ProbeErrorCategory.NO_VIDEO_DATA,
                    message=(
                        "No Video Data — The camera connected successfully "
                        "but is not producing video. The camera may be "
                        "initializing, misconfigured, or the stream path "
                        "may be incorrect."
                    ),
                    stage=ProbeStage.FRAME_DECODE,
                    detail=f"cap.read() returned ({ret}, {type(frame)}) for {masked}",
                )

            return ProbeResult(
                success=True,
                width=width if width > 0 else None,
                height=height if height > 0 else None,
                fps=fps if fps > 0 else None,
                codec=codec,
            )
        finally:
            cap.release()
```

### Usage in Django Service

```python
# apps/cameras/services.py

from apps.cameras.exceptions import CameraConnectionError
from apps.cameras.probe import RtspProber, mask_rtsp_url

import logging

logger = logging.getLogger(__name__)


class CameraService:
    @staticmethod
    def validate_stream(decrypted_url: str, timeout: float = 10.0) -> ProbeResult:
        """Validate an RTSP URL via the three-stage probe."""
        masked = mask_rtsp_url(decrypted_url)
        logger.info("Starting RTSP probe for %s", masked)

        prober = RtspProber(total_timeout=timeout)
        try:
            result = prober.probe(decrypted_url)
            logger.info(
                "RTSP probe succeeded for %s — %dx%d @ %.1f fps (%.0f ms)",
                masked,
                result.width or 0,
                result.height or 0,
                result.fps or 0,
                result.elapsed_ms,
            )
            return result
        except CameraConnectionError as exc:
            logger.warning(
                "RTSP probe failed for %s at stage %d [%s]: %s",
                masked,
                exc.stage,
                exc.category,
                exc.detail,
            )
            raise
```

### Thread-Safety for Celery Workers

The probe is safe to run in Celery workers because:

- Each `RtspProber.probe()` call creates its own `socket` and `cv2.VideoCapture` instances — no shared state.
- `cv2.VideoCapture` is thread-safe per instance.
- The probe is entirely synchronous — no async/await needed.
- Each Celery worker process has its own GIL — probes in different workers are fully parallel.

For the connect endpoint (synchronous DRF view), the probe blocks the request thread for up to 10 seconds. This is acceptable because:

- Camera connect is a low-frequency operation (not called in a hot loop).
- Django's thread-based request handling means other requests are not blocked.
- The 10-second timeout is bounded and configurable.

If needed, the probe can be offloaded to a Celery task to avoid blocking the request thread entirely:

```python
# apps/cameras/tasks.py

from celery import shared_task

@shared_task(bind=True, max_retries=0, time_limit=15, soft_time_limit=12)
def probe_rtsp_stream(self, encrypted_url: str, camera_id: str) -> dict:
    """Celery task wrapper for RTSP probe."""
    decrypted = decrypt(encrypted_url)
    prober = RtspProber(total_timeout=10.0)
    try:
        result = prober.probe(decrypted)
        return {"success": True, "width": result.width, "height": result.height, "fps": result.fps}
    except CameraConnectionError as exc:
        return {"success": False, "category": exc.category.value, "message": exc.message, "stage": exc.stage.value}
    finally:
        del decrypted
```

---

## 8. Edge Cases & Gotchas

| Issue | Mitigation |
|-------|-----------|
| **OpenCV `CAP_PROP_OPEN_TIMEOUT_MSEC` not respected** | Some OpenCV builds ignore this property. Always enforce an outer timeout using `threading.Timer` that calls `cap.release()` if the operation hangs. Or use Python's `signal.alarm` (Unix-only, not suitable for Windows dev). |
| **`cv2.VideoCapture` constructor blocks indefinitely** | Known issue with some RTSP servers. The constructor does the full RTSP handshake. Mitigation: run `VideoCapture()` in a daemon thread with a join timeout. If it doesn't return, the daemon thread is abandoned (its `VideoCapture` is garbage-collected). |
| **Self-signed TLS certificates** | `rtsps://` with self-signed certs works by default because OpenCV/FFmpeg does not verify certificates by default. No action needed. |
| **RTSP over UDP vs TCP** | Some cameras default to UDP transport. `cv2.VideoCapture` uses TCP by default with FFmpeg backend, which is more reliable through firewalls/NATs. Can force TCP with: `cap.set(cv2.CAP_PROP_FFMPEG_CAPTURE_OPTIONS, "rtsp_transport;tcp")`. However, this property is not available in all OpenCV versions. Alternative: set `OPENCV_FFMPEG_CAPTURE_OPTIONS=rtsp_transport;tcp` environment variable. |
| **Camera returns video but wrong codec** | The probe confirms a frame decodes — if OpenCV can decode it, the detection pipeline can too. No extra checking needed. |
| **Multicast RTSP** | Rare for IP cameras in exam monitoring. `cv2.VideoCapture` handles multicast addresses. No special code needed. |
| **URL with special characters in password** | `urlparse` handles URL-encoded characters. Passwords with `@`, `:`, `/` should be URL-encoded in the input: `rtsp://admin:P%40ssw0rd@host/stream`. |
| **IPv6 RTSP addresses** | `urlparse` handles `rtsp://[::1]:554/stream` correctly. `socket.create_connection` handles IPv6. No special code needed. |

---

## 9. Testing Strategy

### Unit Tests (mock socket & cv2)

```python
# Recommended test cases:

# Stage 1 tests:
# - test_tcp_connect_success
# - test_tcp_connect_timeout → NetworkError(NETWORK_TIMEOUT)
# - test_tcp_connect_refused → NetworkError(NETWORK_REFUSED)
# - test_tcp_dns_failure → NetworkError(NETWORK_DNS)
# - test_tcp_connect_rtsps_tls_handshake

# Stage 2 tests:
# - test_rtsp_open_success (mock cap.isOpened() → True)
# - test_rtsp_open_failure (mock cap.isOpened() → False) → RtspError
# - test_rtsp_open_timeout → RtspError(RTSP_TIMEOUT)

# Stage 3 tests:
# - test_frame_decode_success (mock cap.read() → (True, ndarray))
# - test_frame_decode_failure (mock cap.read() → (False, None)) → VideoError
# - test_frame_returns_metadata (width, height, fps, codec)

# Cross-cutting tests:
# - test_total_timeout_exceeded → CameraConnectionError(PROBE_TIMEOUT)
# - test_mask_rtsp_url_with_credentials
# - test_mask_rtsp_url_without_credentials
# - test_parse_host_port_rtsp
# - test_parse_host_port_rtsps
# - test_probe_cleans_up_resources (cap.release called even on error)
```

### Integration Tests (real camera or RTSP simulator)

For CI, use an RTSP test server (e.g., `rtsp-simple-server` / `mediamtx` in a Docker sidecar) that serves a test pattern:

```yaml
# docker-compose.test.yml
services:
  rtsp-simulator:
    image: bluenviron/mediamtx:latest
    ports:
      - "8554:8554"
    environment:
      MTX_PATHS_TEST_SOURCE: "publisher"
  
  # Feed a test pattern into the simulator
  ffmpeg-publisher:
    image: linuxserver/ffmpeg:latest
    command: >
      -re -f lavfi -i testsrc=size=640x480:rate=15
      -c:v libx264 -f rtsp rtsp://rtsp-simulator:8554/test
    depends_on:
      - rtsp-simulator
```

---

## 10. Summary & Decision Record

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | **Stage 1: Raw TCP socket** | Stdlib, zero dependencies, gives distinct network-level error classification |
| 2 | **Stages 2–3: `cv2.VideoCapture`** | Already installed (via ultralytics), same code path as detection pipeline, proven RTSP support |
| 3 | **Not using PyAV** | Adds dependency, potential FFmpeg version conflicts, marginal benefit for error granularity |
| 4 | **Not using subprocess ffprobe** | Credential leakage in process list, fragile stderr parsing, unnecessary subprocess overhead |
| 5 | **Outer timeout via monotonic clock** | Ensures hard 10s cap regardless of per-stage timeouts |
| 6 | **Exception hierarchy with category enum** | Maps cleanly to user-facing error messages and frontend error types |
| 7 | **URL masking utility** | Single function for all credential sanitization in logs/errors/UI |
| 8 | **RTSPS via conditional TLS wrap (Stage 1) + transparent OpenCV (Stages 2–3)** | Supports both schemes with minimal code |
| 9 | **CERT_NONE for TLS** | IP cameras universally use self-signed certificates |
| 10 | **Sync execution (not async)** | Probe is a one-shot operation in a request/task context; async adds complexity without benefit |
