# Spec Addendum: Design Decisions & Gap Resolution

**Feature**: 001-exam-monitor-dashboard | **Date**: 2026-02-28
**Purpose**: Resolve all remaining checklist gaps from comprehensive.md, frontend.md, setup.md, and ux.md with concrete, implementable decisions.

---

## 1. Visual Design System

### 1.1 Design Tokens (CSS Custom Properties)

All three themes use the following token schema. Values for Black+Purple are canonical; Full Black and White themes override only color tokens.

**Color Tokens**:
| Token | Black+Purple | Full Black | White |
|-------|-------------|------------|-------|
| `--color-bg-primary` | `#1A1A2E` | `#000000` | `#FFFFFF` |
| `--color-bg-secondary` | `#16213E` | `#0A0A0A` | `#F8F8F8` |
| `--color-bg-tertiary` | `#0F3460` | `#141414` | `#F0F0F0` |
| `--color-bg-card` | `#1E2A4A` | `#111111` | `#FFFFFF` |
| `--color-bg-input` | `#16213E` | `#0A0A0A` | `#F5F5F5` |
| `--color-text-primary` | `#E0E0E0` | `#E0E0E0` | `#1A1A1A` |
| `--color-text-secondary` | `#A0A0B0` | `#888888` | `#666666` |
| `--color-text-muted` | `#6B6B80` | `#555555` | `#999999` |
| `--color-accent` | `#7C3AED` | `#7C3AED` | `#7C3AED` |
| `--color-accent-hover` | `#6D28D9` | `#6D28D9` | `#6D28D9` |
| `--color-accent-muted` | `#7C3AED33` | `#7C3AED33` | `#7C3AED1A` |
| `--color-border` | `#2A2A4A` | `#222222` | `#E0E0E0` |
| `--color-border-focus` | `#7C3AED` | `#7C3AED` | `#7C3AED` |
| `--color-success` | `#22C55E` | `#22C55E` | `#16A34A` |
| `--color-warning` | `#EAB308` | `#EAB308` | `#CA8A04` |
| `--color-error` | `#EF4444` | `#EF4444` | `#DC2626` |
| `--color-info` | `#3B82F6` | `#3B82F6` | `#2563EB` |

**Severity Color Tokens** (consistent across all contexts):
| Token | Value | Usage |
|-------|-------|-------|
| `--color-severity-high` | `#EF4444` | High severity anomaly borders, badges |
| `--color-severity-medium` | `#EAB308` | Medium severity |
| `--color-severity-low` | `#3B82F6` | Low severity (blue — not distracting) |

**Detection Class Color Tokens** (resolved conflict: R-009 takes precedence over narrative US-2):
| Token | Value | Class |
|-------|-------|-------|
| `--color-class-student` | `#22D3EE` | Student (cyan) |
| `--color-class-teacher` | `#A78BFA` | Teacher (purple) |
| `--color-class-standing` | `#F97316` | Standing (orange) |
| `--color-class-sitting` | `#22C55E` | Sitting (green) |
| `--color-class-looking-left` | `#FB923C` | Looking Left (amber) |
| `--color-class-looking-right` | `#FACC15` | Looking Right (yellow) |
| `--color-class-looking-forward` | `#34D399` | Looking Forward (emerald) |
| `--color-class-looking-down` | `#A3E635` | Looking Down (lime) |

**Spacing Tokens**:
| Token | Value |
|-------|-------|
| `--spacing-xs` | `4px` |
| `--spacing-sm` | `8px` |
| `--spacing-md` | `16px` |
| `--spacing-lg` | `24px` |
| `--spacing-xl` | `32px` |
| `--spacing-2xl` | `48px` |

**Typography Tokens**:
| Token | Value |
|-------|-------|
| `--font-family` | `'Inter', system-ui, -apple-system, sans-serif` |
| `--font-mono` | `'JetBrains Mono', 'Fira Code', monospace` |
| `--font-size-xs` | `0.75rem` (12px) |
| `--font-size-sm` | `0.875rem` (14px) |
| `--font-size-base` | `1rem` (16px) |
| `--font-size-lg` | `1.125rem` (18px) |
| `--font-size-xl` | `1.25rem` (20px) |
| `--font-size-2xl` | `1.5rem` (24px) |
| `--font-size-3xl` | `1.875rem` (30px) |
| `--font-weight-normal` | `400` |
| `--font-weight-medium` | `500` |
| `--font-weight-semibold` | `600` |
| `--font-weight-bold` | `700` |
| `--line-height-tight` | `1.25` |
| `--line-height-normal` | `1.5` |
| `--line-height-relaxed` | `1.75` |

**Shape Tokens**:
| Token | Value |
|-------|-------|
| `--radius-sm` | `4px` |
| `--radius-md` | `8px` |
| `--radius-lg` | `12px` |
| `--radius-xl` | `16px` |
| `--radius-full` | `9999px` |
| `--shadow-sm` | `0 1px 2px rgba(0,0,0,0.3)` |
| `--shadow-md` | `0 4px 6px rgba(0,0,0,0.3)` |
| `--shadow-lg` | `0 10px 15px rgba(0,0,0,0.3)` |
| `--shadow-xl` | `0 20px 25px rgba(0,0,0,0.4)` |

**Transition Tokens**:
| Token | Value |
|-------|-------|
| `--transition-fast` | `150ms ease-out` |
| `--transition-normal` | `200ms ease-out` |
| `--transition-slow` | `300ms ease-in-out` |
| `--transition-theme` | `300ms ease-in-out` |

### 1.2 Theme Switch Animation

- **Easing**: `ease-in-out` (CSS `transition-timing-function`)
- **Duration**: 300ms (FR-022)
- **Properties animated**: `background-color`, `color`, `border-color`, `box-shadow`, `fill`, `stroke`
- **Stagger order**: None — all properties transition simultaneously via CSS `transition: all var(--transition-theme)`
- **Measurement**: Time from `data-theme` attribute change on `<html>` to `getComputedStyle()` returning new values. Measured via Playwright `page.evaluate()` + `performance.now()`.
- **FOUC prevention**: Inline `<script>` in `index.html` reads `localStorage.getItem('theme')` and sets `data-theme` before first paint. If script fails or JS disabled, default theme (Black+Purple) applies via CSS `:root` defaults.

### 1.3 Iconography

- **Library**: Lucide React (`lucide-react`) — MIT license, tree-shakeable, 1000+ icons
- **Icon-to-Meaning Mapping**:
  | Status | Icon | Color Token |
  |--------|------|-------------|
  | Connected | `CheckCircle` | `--color-success` |
  | Disconnected | `XCircle` | `--color-error` |
  | Reconnecting | `RefreshCw` (spinning) | `--color-warning` |
  | Recording | `Circle` (filled, pulsing) | `--color-error` |
  | Error | `AlertTriangle` | `--color-error` |
  | Warning | `AlertCircle` | `--color-warning` |
  | Info | `Info` | `--color-info` |
  | New anomaly | `Bell` | `--color-severity-high` |
  | Acknowledged | `CheckCheck` | `--color-accent` |
  | Dismissed | `X` | `--color-text-muted` |

---

## 2. Component Library Specifications

### 2.1 UI Component Props & States

**Button**:
- Variants: `primary` (accent bg), `secondary` (border only), `ghost` (no border), `danger` (error bg)
- Sizes: `sm` (28px height), `md` (36px height), `lg` (44px height)
- States: default, hover (`--color-accent-hover`), active (scale 0.98), focus (2px accent ring), disabled (opacity 0.5, `cursor: not-allowed`), loading (spinner replaces text)

**Card**:
- Variants: `default` (bg-card, border), `elevated` (shadow-md), `interactive` (hover: shadow-lg, slight lift)
- Used by: `StudentCard`, `AnomalyAlert`, recording list items (all extend base Card)

**Modal**:
- Focus trap: Yes — focus trapped inside modal, returned to trigger on close
- Backdrop: `rgba(0,0,0,0.6)`, click-to-dismiss (unless `preventClose`)
- Animation: fade-in backdrop (200ms), slide-up content (200ms)
- ARIA: `role="dialog"`, `aria-modal="true"`, `aria-labelledby` pointing to title

**StatusBadge**:
- Variants: `success`, `warning`, `error`, `info`, `neutral`
- Dot indicator + label text
- Size: always `sm` (12px dot + 12px font)

**FormInput**:
- States: default, focus (accent ring), error (error border + error text below), disabled (muted bg)
- Validation: inline error text appears below input, `aria-describedby` for error message
- Character counter: shown when `maxLength` prop provided

**Toggle**:
- Track: 40px × 22px, rounded-full
- Thumb: 18px circle, accent color when on, muted when off
- Animation: 150ms ease-out translate

**LoadingSpinner**:
- Sizes: `sm` (16px), `md` (24px), `lg` (40px)
- Animation: `spin 0.75s linear infinite`
- Color: inherits `currentColor`

### 2.2 Layout Composition

- **AppLayout** wraps all authenticated pages: Header (fixed top, 56px) + Sidebar (left, 240px collapsed to 56px icons-only) + Main content area
- **Sidebar**: Always rendered for authenticated users. Collapsible via hamburger icon in Header. Collapsed state persisted to localStorage. On mobile (<1024px): overlay drawer
- **Header**: Logo left, breadcrumb center, ThemeToggle + UserMenu right. Theme toggle is in the Header (canonical location per plan's `ThemeToggle.tsx` in layout). Settings page also has theme selection for discoverability

### 2.3 Focus Indicators

Across all three themes:
- **Default focus ring**: `2px solid var(--color-border-focus)` with `2px offset`
- **High contrast mode** (via `@media (forced-colors: active)`): browser default focus indicators
- **Keyboard-only**: Focus ring shown only on `:focus-visible` (not on mouse click)

---

## 3. Page Layout Specifications

### 3.1 Login Page

- **Layout**: Full viewport, centered card (max-width 400px)
- **Visual treatment**: Dark gradient background (`--color-bg-primary` → `--color-bg-secondary`), subtle animated gradient shimmer on card border using `--color-accent`
- **Elements**: Logo/brand mark, "Exam Monitor" title (font-size-3xl), identifier input, password input, "Sign In" primary button, error alert below form
- **"Visually striking"**: Gradient accent border animation (3s loop, accent → accent-hover → accent), card with elevated shadow

### 3.2 Dashboard Page

- **Layout**: 2-column grid on ≥1440px, single column on smaller
- **Content** (FR-046):
  - Active Sessions card: count badge, list of session cards with exam name, camera count, anomaly count
  - Camera Status card: connected/disconnected/total with StatusBadges
  - Recent Anomalies card: last 5 anomaly alerts with severity badges (links to full alert history)
  - System Health card: StatusBadge for each subsystem (API, WS, Detection, Storage)
  - Quick Actions: "Start Session" and "Connect Camera" buttons
- **Admin dashboard** extends with: instructor overview, storage stats, activity feed

### 3.3 Camera Feed Page

- **Layout**: Grid area (FR-007 breakpoints) + optional Predictions Panel sidebar (right, 320px)
- **Each camera tile**: Video element + Canvas overlay + status bar (bottom: camera name, status badge, recording indicator)
- **Fullscreen**: Double-click camera tile → fullscreen (CSS `position: fixed` with z-index). Small expand icon in top-right corner as visual affordance for discoverability (CHK064)

### 3.4 Predictions Page

- **Layout**: Left panel = Camera feed (single camera selector), Right panel = StudentCard grid
- **StudentCard arrangement**: CSS Grid, 2 columns on ≥1200px, 1 column on smaller. Sorted by tracking_id ascending. Scrollable container with virtual scroll for 20+ students
- **StudentCard contents (compact)**: Tracking ID, primary posture badge, highest-confidence gaze direction, mini confidence bar
- **StudentCard contents (expanded)**: All pyramid fields with confidence bars (0-100% progress bar), constraint violation indicator, timestamp of last update

### 3.5 Recordings Page

- **Layout**: Table list with columns: Camera Name, Session, Duration, Anomaly Count, Date, Actions (Play, Export)
- **Sorting**: by date (default desc), duration, anomaly count
- **Filtering**: by session, camera, date range

### 3.6 Recording Playback Page

- **Layout**: Video player (center, max-width) + Canvas overlay + anomaly timeline scrubber (bottom)
- **Controls**: Play/Pause (Space), Seek bar, Speed selector (0.5x, 1x, 1.5x, 2x), Volume, Fullscreen
- **Timeline markers**: Small circles (8px diameter) colored by severity (high=red, medium=yellow, low=blue). Hover shows tooltip with description. Click snaps playback to marker timestamp. Not draggable
- **Detection overlay data strategy**: Fetch detection frames in windowed chunks of ±30 seconds around current playback position. Prefetch next 30s chunk during playback. Cache up to 5 chunks in memory (FIFO eviction)

### 3.7 Health Dashboard Page

- **Layout**: Grid of metric cards (2×3 on ≥1440px, 2×2 on smaller, 1 column on mobile)
- **Metric visualizations**:
  - API Status: StatusBadge (up/down) + response time number
  - WebSocket Health: StatusBadge + active connections count + message rate gauge
  - Detection FPS: Per-camera horizontal bar chart (green ≥15fps, yellow 10-14, red <10)
  - Storage: Progress bar with percentage, colored by threshold (green <80%, yellow 80-95%, red >95%)
  - Active Sessions: Number badge with list of session summaries
- **Auto-refresh**: Fixed 10-second polling interval (FR-040). Not configurable in UI — admin can override via environment variable `HEALTH_REFRESH_INTERVAL_MS`

### 3.8 Settings Page

- **Layout**: Single column, max-width 640px, centered
- **Sections**:
  - Theme Selection: 3 theme preview cards, current theme highlighted
  - Profile: First name, last name, email (read-only), username (read-only)
  - Change Password: Current password, new password, confirm new password
  - About: App version, support info

---

## 4. State Designs

### 4.1 Empty States

All empty states use the same pattern: centered icon (48px, muted color) + title (font-size-lg) + description (font-size-sm, text-muted) + optional action button.

| Page/Component | Icon | Title | Description | Action |
|---------------|------|-------|-------------|--------|
| Camera Feed (no cameras) | `Camera` | No cameras connected | Connect a camera to start monitoring | "Connect Camera" button |
| Predictions Panel (no detections) | `Search` | Waiting for detections... | Detection results will appear here once the pipeline starts | — |
| Anomaly List (no anomalies) | `ShieldCheck` | No anomalies detected | All students appear to be behaving normally | — |
| Recordings (no recordings) | `Video` | No recordings yet | Recordings are created when you start a monitoring session | "Start Session" button |
| Session List (no sessions) | `Clock` | No sessions found | Start a new monitoring session from the Camera Feed | "Go to Camera Feed" button |
| Alert History (empty) | `Bell` | No alert history | Anomaly alerts will appear here as they are detected | — |

### 4.2 Loading States

- **Page transitions**: Route-level `<Suspense>` with centered `LoadingSpinner` (lg size)
- **Data fetching**: Skeleton screens for list/card content. Skeleton uses `--color-bg-tertiary` with shimmer animation (1.5s, left-to-right)
- **Camera connecting**: Camera tile shows `LoadingSpinner` + "Connecting..." text centered on dark bg
- **Export**: Progress bar (determinate) with percentage from `progress_percent` field
- **Async operations** (triage, save): Button enters `loading` state (spinner replaces text, disabled)

### 4.3 Error States

All error displays follow the pattern: icon (AlertTriangle or XCircle) + bold title + description + suggested action.

| Error Scenario | UI Treatment | Template |
|---------------|--------------|----------|
| Camera connection failed | Red banner on camera tile | "Connection Failed — Check that the camera is online and the RTSP URL is correct. [Retry]" |
| WebSocket disconnected | Top-of-page persistent banner | "Connection Lost — Reconnecting (attempt {n}/{max})..." then "Connection Lost — Please refresh the page. [Refresh]" |
| Detection service unavailable | Yellow banner below camera grid | "Detection Service Unavailable — Raw Feed Only. Detection pipeline is offline. Bounding boxes and predictions are temporarily unavailable." |
| Storage warning 80% | Persistent yellow banner (top of page, dismissible) | "Storage at {n}% — Consider exporting and deleting old recordings." |
| Storage warning 95% | Blocking modal (non-dismissible by instructors) | "Storage Critical ({n}%) — Recording will stop automatically at 100%. Contact administrator." |
| API request failed | Toast notification (bottom-right, 5s auto-dismiss) | "[Action] failed — [Reason]. [Retry]" |
| WHEP/WebRTC failed | Overlay on camera tile | "Video Stream Error — Unable to establish WebRTC connection. [Retry] / Check go2rtc status." |
| SDP negotiation failed | Same as WHEP failed | "Stream Negotiation Failed — go2rtc may be unreachable. [Retry]" |
| MediaStream track ended | Camera tile shows still frame + overlay | "Stream Ended Unexpectedly — [Reconnect]" |

### 4.4 Toast/Notification System

- **Position**: Bottom-right, stacked vertically (max 3 visible)
- **Auto-dismiss**: 5 seconds (errors: 8 seconds)
- **Variants**: `success` (green), `error` (red), `warning` (yellow), `info` (blue)
- **Actions triggering toasts**:
  - Camera connected/disconnected: success/info
  - Anomaly acknowledged/dismissed: success
  - Recording started/stopped: info
  - Export completed: success with download link
  - Triage conflict (409): warning — "Already triaged by [Name]. Refreshing..."
  - Network error: error
- **ARIA**: Each toast uses `role="status"` and `aria-live="polite"` (errors use `aria-live="assertive"`)

---

## 5. Anomaly Management

### 5.1 Alert History Layout (FR-045)

- **Layout**: Table view (not timeline) with columns: Severity, Student ID, Description, Status, Triaged By, Time
- **Pagination**: 20 items per page, standard prev/next pagination
- **Sorting**: Default by timestamp descending. Sortable columns: severity, status, timestamp
- **Filtering**: By severity (multi-select), status (multi-select), camera, date range
- **Search**: Text search on description and tracking_id

### 5.2 Anomaly Status Visual Treatment

| Status | Badge Color | Opacity | Visual Treatment |
|--------|-------------|---------|------------------|
| `new` | `--color-severity-{level}` | 1.0 | Full color, slight pulse animation on border (2s ease-in-out) |
| `acknowledged` | `--color-accent` | 0.7 | Muted — reduced opacity, accent badge replacing severity color |
| `dismissed` | `--color-text-muted` | 0.5 | Strongly muted — grayed out, strikethrough on description |

### 5.3 Batch Triage

- **Not supported in v1**. Individual triage only. Batch triage deferred to v2 backlog.
- Rationale: First-write-wins conflict resolution is complex to batch. Individual triage ensures proper audit trail per action.

### 5.4 Triage Rejection (First-Write-Wins 409)

When a triage action returns HTTP 409:
1. Show warning toast: "Already triaged by [Instructor Name]. Refreshing status..."
2. Auto-refresh the anomaly status from server (GET anomaly detail)
3. Update the AnomalyAlert card to reflect current status
4. TriageActions component re-renders with updated status — if `acknowledged`, only Dismiss is available; if `dismissed`, both buttons disabled

### 5.5 Dismiss Reason Input (FR-042)

- **UI Type**: Inline expanding text area below the Dismiss button (not modal)
- **Character counter**: Shows "{current}/{min 5}" below textarea, red when <5
- **Validation**: Submit button disabled until ≥5 characters. Error text: "Reason must be at least 5 characters"
- **Size**: 3 rows, max 500 characters

### 5.6 Anomaly Annotation (FR-043)

- **Location**: Expandable "Notes" section below each AnomalyAlert card
- **Input**: Single-line text input + "Add Note" button
- **Note list**: Chronological list below input — each note shows: user name, timestamp, content
- **ARIA**: Notes section uses `aria-label="Anomaly notes"`

### 5.7 Click-to-Highlight Bounding Box (FR-018)

- **Visual treatment**: 3px stroke width (normal: 2px), pulsing glow animation (`box-shadow: 0 0 8px var(--color-class-{type})`, 1s ease-in-out infinite)
- **Auto-dismiss**: 5 seconds, then reverts to normal stroke
- **Trigger**: Click on anomaly alert card → highlights corresponding student bbox on camera canvas

### 5.8 Rapid Anomaly Debouncing

- **Backend-side**: Anomaly events for the same student with the same rule violation are debounced with a 30-second cooldown. A new AnomalyEvent is created only if the behavior persists beyond the previous event's `behavior_ended_at` + 30s.
- **Frontend-side**: No additional debouncing needed — display all received events as-is.

---

## 6. Camera & Canvas

### 6.1 Connection Lost Overlay (FR-010)

- **Background**: `rgba(0,0,0,0.75)`
- **Icon**: `RefreshCw` (spinning, 48px, white)
- **Text**: "Connection Lost — Reconnecting..." (font-size-lg, white)
- **Retry counter**: "Attempt {n} of 5" (font-size-sm, text-muted)
- **After max retries**: Icon changes to `XCircle`, text changes to "Connection Failed", shows "Retry" button

### 6.2 Canvas Coordinate System

- **Source**: Bounding box `[x, y, w, h]` values are in source video resolution pixels (e.g., 1920×1080)
- **Scaling**: Canvas scales bbox coordinates proportionally to the displayed video element dimensions. Scaling factor: `displayWidth / sourceWidth` and `displayHeight / sourceHeight`
- **On resize**: `ResizeObserver` on the video container re-calculates canvas dimensions and re-renders current frame's bounding boxes immediately. No flicker tolerance needed — Canvas re-draw is synchronous within the same animation frame
- **CSS variables for colors**: Canvas `strokeStyle` reads color values via `getComputedStyle(document.documentElement).getPropertyValue('--color-class-student')` at theme change. Values cached in a JS Map and invalidated on `data-theme` attribute change via `MutationObserver`

### 6.3 Maximum Bounding Boxes

- **Cap**: 200 simultaneous bounding boxes across all visible camera tiles (8 cameras × 25 students = 200 practical max)
- **Degradation strategy**: If >200, skip rendering labels (text) and only draw rectangles. If >400 (highly unlikely), skip alternate frames
- **Frame buffer**: Frontend detection store retains the latest 1 frame per camera (newest wins). No time-window buffer. Total memory: ~8 × 200 detections × ~200 bytes = ~320KB — negligible

### 6.4 Preview Mode (FR-006)

Before session starts (camera connected but no active session):
- **Visible**: Live video feed, camera name, connection status badge
- **Hidden**: Bounding boxes, predictions panel, anomaly panel, recording indicator
- **Status text**: "Preview Mode — Start a session to enable detection" (yellow text, font-size-sm)

### 6.5 Filter Panel (FR-013)

- **Type**: Inline horizontal toggle bar above the camera grid (not sidebar or popover)
- **Position**: Below the page header, above the camera grid
- **Controls**: Toggle pills for each detection class (Student, Teacher, Standing, Sitting, etc.). Active pills are accent-colored; inactive are muted border
- **Behavior**: Toggling a class immediately shows/hides those bounding boxes on the canvas (<200ms per SC-003)

### 6.6 Fullscreen Affordance

- **Visual affordance**: Small expand icon (`Maximize2`, 20px, semi-transparent white) in the top-right corner of each camera tile, visible on hover
- **Trigger**: Double-click on camera tile OR click expand icon
- **Exit**: Press Escape or click collapse icon

---

## 7. WebSocket & Real-time Behavior

### 7.1 Dashboard Refresh

- **Mechanism**: WebSocket push via `dashboard.update` message (not polling). Backend sends dashboard data via the existing WebSocket connection when any subscribed metric changes
- **Fallback**: If WebSocket is disconnected, frontend falls back to polling every 10 seconds until WS reconnects

### 7.2 Reconnection UI

- **During reconnection attempts**: Top-of-page persistent yellow banner: "Reconnecting... (attempt {n}/10)"
- **Per-attempt update**: Yes, the banner updates the attempt number in real-time
- **After all retries exhausted**: Banner turns red: "Connection Lost — Please refresh the page. [Refresh]"
- **On successful reconnection**: Banner auto-dismisses with brief green flash, auto-resubscribes to all previous channels

### 7.3 Message Ordering & Deduplication

- **Ordering**: Display as-received. No reordering by timestamp. Rationale: at ~10ms between frames, visual ordering artifacts are imperceptible
- **Deduplication**: Frontend stores latest `frame_number` per camera. If a received frame's `frame_number` ≤ stored value, silently discard
- **Corrupted/malformed messages**: `try/catch` around JSON.parse. If parsing fails or required fields missing, log to `console.error` with message preview (first 200 chars), silently skip. No user-facing warning for individual bad messages; if >10 consecutive failures, show degraded status banner

### 7.4 Partial Subscription Failure

- After WebSocket reconnect, if some camera subscriptions fail:
- Show camera-specific error overlay ("Subscription Failed — [Retry]") on the affected camera tiles
- Other camera tiles continue normally
- Do not block the entire grid for partial failures

### 7.5 WebSocket Throughput Strategy

- **Processing**: Batch incoming `detection.frame` messages into `requestAnimationFrame` cycles. Accumulate messages in a per-camera buffer, then process the latest message per camera per rAF cycle (dropping intermediate frames)
- **Result**: At 60fps rAF, each camera processes at most 60 frames/sec — well within budget

### 7.6 Backpressure / Flow Control

- **Client-side**: If the message queue exceeds 100 unprocessed messages, log a warning and drop the oldest 50. This should never happen with the rAF batching strategy
- **Server-side**: Django Channels' Redis channel layer has built-in capacity limits per channel (default 100). If a consumer falls behind, oldest messages are dropped automatically

### 7.7 Retry Asymmetry Rationale (FR-010 vs FR-019)

- **Camera (5 retries)**: RTSP connections are infrastructure-level. If a camera doesn't respond after 5 attempts (with exponential backoff), it's likely a hardware/network issue requiring manual intervention
- **WebSocket (10 retries)**: Browser WebSocket reconnects are typically transient (brief network hiccups, server restart). More retries with exponential backoff (1s→16s) cover typical server rolling restart durations (~30-60s)

---

## 8. Session & Authentication

### 8.1 Session Invalidation on Password Change

- When a user changes their password (PUT `/auth/me/password/`), all other active sessions for that user are invalidated immediately
- Implementation: Django's `update_session_auth_hash()` invalidates other sessions automatically
- When an admin deactivates a user (DELETE `/admin/users/{id}/`): all active sessions for that user are flushed from the session store. If the user has an active monitoring session, it is ended gracefully first

### 8.2 Frontend Session Timeout Detection

- **Mechanism**: On any API 401 response, the axios interceptor redirects to `/login`
- **Proactive detection**: Frontend maintains a `lastActivity` timestamp (updated on mouse/keyboard events). If `Date.now() - lastActivity > 29 minutes`, show a warning modal: "Your session will expire in 1 minute due to inactivity. [Stay Logged In] [Log Out]". "Stay Logged In" calls `GET /auth/me/` to refresh the session
- **WebSocket keepalive**: Ping/pong every 30 seconds counts as activity on the server side

### 8.3 Deep Link Behavior

- **Unauthenticated access to protected route**: `RouteGuard` stores target path in `sessionStorage`, redirects to `/login`. After successful login, redirects back to stored path then clears it
- **Invalid routes**: Show `NotFoundPage` (404)

### 8.4 First-Write-Wins Message Format

Both acknowledge (FR-041) and dismiss (FR-042) rejection messages include the acting instructor's identity:
- Response 409 body: `{ "error": "Already triaged by [Name]", "code": "ALREADY_TRIAGED" }`
- Frontend displays: toast with "Already triaged by [Name]" + auto-refreshes anomaly status

### 8.5 Multi-Tab Behavior

- Each tab operates independently with its own WebSocket connection and camera subscriptions (per edge case §multi-tab)
- Triage actions sync naturally: when Tab A triages, the backend broadcasts the status change via WebSocket. Tab B receives the update and re-renders the anomaly status
- Theme changes sync across tabs via `window.addEventListener('storage', ...)` listening for localStorage changes

---

## 9. Recording & Export

### 9.1 Recording Player Controls

Full control set:
- **Play/Pause**: Click or Space key
- **Seek**: Click on progress bar or arrow keys (Left/Right: ±5s, Shift+Left/Right: ±30s)
- **Speed**: Dropdown selector — 0.5×, 1× (default), 1.5×, 2×
- **Volume**: Slider + mute toggle (M key) — note: most surveillance recordings have no audio
- **Fullscreen**: Button or F key

### 9.2 Seek Performance (SC-008)

- "<500ms recording seek" includes: video element seek + fetching the nearest detection chunk + canvas overlay re-render
- If the target position's chunk is already cached, seek should be <200ms
- If chunk needs fetching, a 300ms loading skeleton shows on the canvas overlay

### 9.3 Export Flow

- **Polling interval**: 2 seconds
- **Maximum poll duration**: 10 minutes (after which show: "Export is taking longer than expected. You can close this page and return later — the export will continue in the background.")
- **Timeout**: None — polling continues until status is `completed` or `failed`
- **Error states**:
  - POST initiate 500: Toast error — "Export failed to start. Please try again. [Retry]"
  - Polling detects `failed`: Toast error — "Export failed: {error message}. [Retry]" (retry re-triggers POST)
  - Download link expired: Toast warning — "Download link expired. Re-exporting..." (auto-retrigger)

### 9.4 Export Archive Structure (FR-048)

Per rest-api.md contract — already defined:
```
session_{id}_export/
├── recordings/          # Raw video MP4 files (one per camera)
│   ├── camera_{uuid}.mp4
├── detections.json      # All DetectionFrame + Detection + PyramidPrediction data
├── anomalies.json       # All AnomalyEvent records with triage status + notes
├── session_metadata.json  # MonitoringSession info, camera list, instructor
└── tracking_ids.json    # Student tracking ID mapping
```

### 9.5 Export Progress Computation

- **Method**: Composite of phases:
  - 0-10%: Querying data from database
  - 10-60%: Encoding video files (per-camera progress averaged)
  - 60-90%: Serializing JSON data files
  - 90-100%: Creating ZIP archive

---

## 10. Configuration & Limits

### 10.1 Inactivity Timeout

- **Default**: 30 minutes (FR-004)
- **Bounds**: Min 5 minutes, max 480 minutes (8 hours)
- **Configuration mechanism**: Environment variable `SESSION_TIMEOUT_MINUTES=30`. Also settable via Django admin panel (SystemConfig model, if implemented)

### 10.2 Anomaly Rule Engine Configuration (FR-017)

- **Format**: JSON configuration stored in the database (SystemConfig model)
- **Default rules** (hardcoded as initial migration data):
  ```json
  {
    "rules": [
      {
        "id": "rule-gaze-left-30s",
        "description": "Student looking left for >30 seconds",
        "condition": { "horizontal_gaze": "left", "duration_seconds": 30 },
        "severity": "high"
      },
      {
        "id": "rule-posture-turned-30s",
        "description": "Student turned posture for >30 seconds",
        "condition": { "posture": "turned", "duration_seconds": 30 },
        "severity": "high"
      },
      {
        "id": "rule-gaze-down-60s",
        "description": "Student looking down for >60 seconds",
        "condition": { "vertical_gaze": "down", "duration_seconds": 60 },
        "severity": "medium"
      }
    ]
  }
  ```
- **Admin UI**: Deferred to v2 — rules edited via Django admin in v1

### 10.3 Retention Configuration

- **Mechanism**: Environment variables with fallback to defaults from data-model.md:
  - `RETENTION_DETECTION_FRAMES_DAYS=90`
  - `RETENTION_ANOMALY_EVENTS_DAYS=365`
  - `RETENTION_RECORDINGS_DAYS=30`
  - `RETENTION_AUDIT_LOG_DAYS=365`
  - `RETENTION_EXPORTS_DAYS=7`
- **Enforcement**: Celery periodic task (`cleanup_expired_data`) runs daily at 02:00

### 10.4 Storage Capacity Scope

- **Scope**: Mount point of the `MEDIA_ROOT` directory (the storage volume where recordings are saved)
- **Measurement**: Python `shutil.disk_usage(settings.MEDIA_ROOT)`
- **80% threshold**: Persistent warning banner (dismissible, but returns if still over threshold on next check). Check interval: every 5 minutes
- **95% threshold**: Blocking modal requiring admin acknowledgment

### 10.5 Auto-Restart (FR-049)

- **30 seconds**: Aspirational target, not a hard SLA. Implemented via Docker's `restart: unless-stopped` policy
- **Monitoring**: Health endpoint (`GET /api/v1/health/`) returns 503 when services are degraded. External monitoring (e.g., uptime checker) should poll this endpoint
- **Data integrity**: PostgreSQL WAL ensures transaction durability. Active recordings use atomic file writes (write to temp file, rename on completion). Celery tasks use `acks_late=True` for at-least-once delivery

### 10.6 Database Query Performance

- **Target**: All list endpoints respond within 200ms at p95 under normal load (≤20 concurrent users)
- **Playback endpoint** (`GET /detections/frames/`): Target 100ms at p95 for 100-frame pages (covered by PostgreSQL time-range partitioning on DetectionFrame + composite index)
- **No hard SLA** — monitored via health dashboard response times

### 10.7 Resource Limits

Defined in Docker Compose (recommended, not enforced in native installs):
| Service | CPU Limit | Memory Limit |
|---------|-----------|--------------|
| Django (ASGI) | 2 cores | 1GB |
| Celery worker | 2 cores | 2GB |
| go2rtc | 1 core | 512MB |
| PostgreSQL 16 | 2 cores | 1GB |
| Redis 7 | 0.5 core | 256MB |

### 10.8 API Rate Limiting

- **Login endpoint**: 5 attempts per minute per IP (FR-005 brute force protection)
- **All other endpoints**: No rate limiting in v1 — internal university network only (per Assumptions)
- **WebSocket**: No explicit rate limiting. Channel layer capacity limit (100 messages per channel) provides implicit backpressure

---

## 11. Accessibility

### 11.1 WCAG 2.1 AA Contrast Ratios

Per WCAG 2.1 AA, verified per theme:
| Element Type | Required Ratio | Verification |
|-------------|----------------|--------------|
| Body text (≤18px) | ≥ 4.5:1 | axe-core automated scan |
| Large text (>18px or >14px bold) | ≥ 3:1 | axe-core automated scan |
| UI components, icons | ≥ 3:1 | axe-core automated scan |

**Per-theme verification**: Each theme is tested independently via axe-core Playwright integration. CI pipeline runs axe scans on Login, Dashboard, Camera Feed, and Settings pages for all 3 themes (12 total scans).

### 11.2 ARIA Live Regions

| Content | ARIA Attribute | Rationale |
|---------|---------------|-----------|
| Anomaly alerts (new) | `aria-live="assertive"` | Critical — instructor must know immediately |
| Detection count updates | `aria-live="polite"` | Informational — doesn't interrupt current task |
| Camera status changes | `role="status"` + `aria-live="polite"` | Status change, not urgent |
| Toast notifications | `role="status"` + `aria-live="polite"` (errors: `assertive`) | Transient feedback |
| Connection banner | `role="alert"` | System-wide critical status |

### 11.3 Keyboard Shortcuts

| Action | Shortcut | Page |
|--------|----------|------|
| Focus camera 1-8 | `Alt+1` through `Alt+8` | Camera Feed |
| Toggle filter panel | `Alt+F` | Camera Feed |
| Acknowledge selected anomaly | `Alt+A` | Camera Feed, Alert History |
| Play/Pause recording | `Space` | Recording Playback |
| Seek backward 5s | `Left Arrow` | Recording Playback |
| Seek forward 5s | `Right Arrow` | Recording Playback |
| Seek backward 30s | `Shift+Left` | Recording Playback |
| Seek forward 30s | `Shift+Right` | Recording Playback |
| Fullscreen | `F` | Camera Feed, Recording Playback |
| Toggle sidebar | `Alt+S` | All authenticated pages |

### 11.4 Skip Navigation

- "Skip to main content" link at the very top of every page (`position: absolute`, visible only on focus)
- Targets `<main id="main-content">` element, bypassing Header and Sidebar

### 11.5 Reduced Motion

When `@media (prefers-reduced-motion: reduce)` is active:
- Theme transition duration: 0ms (instant switch)
- Bounding box pulse animation: disabled (static highlight)
- Loading spinner: still icon instead of rotating
- Skeleton shimmer: disabled (static background)
- Toast slide-in: instant appear/disappear

### 11.6 Modal Focus Management

- On open: Focus moves to first focusable element inside modal (or close button)
- Tab cycle: Focus is trapped inside modal (Tab wraps from last to first, Shift+Tab wraps from first to last)
- On close: Focus returns to the element that triggered the modal
- Escape key: Closes the modal (unless `preventClose` is set)
- Screen reader announcement: `aria-modal="true"`, `aria-labelledby` pointing to modal title

---

## 12. Edge Cases

### 12.1 Shared Camera Disconnect

When a camera is disconnected by one instructor:
- All other instructors viewing that camera see the "Connection Lost — Reconnecting..." overlay
- After 5 failed reconnection attempts, overlay changes to "Camera Disconnected by [Instructor Name]" with a "Reconnect" button
- Anomaly alerts for that camera remain visible (historical, not cleared)

### 12.2 Rule Engine Reconfiguration Mid-Session

- Deferred to v2 — in v1, rule engine changes require a service restart
- If implemented later: predictions panel would show an info banner "Detection rules updated — new anomaly criteria now active"

### 12.3 Export Failure Recovery

- **Partial export cleanup**: Failed exports are cleaned up by a Celery periodic task (every hour). Partial ZIP files are deleted from MEDIA_ROOT
- **Disk space recovery**: Automatic — partial file deletion reclaims space
- **Retry**: User can click "Retry" which re-triggers POST initiate (old failed export is garbage collected)

### 12.4 Backend Rolling Restart (FR-049)

- **WebSocket**: Connections drop during restart. Frontend reconnects automatically (10 retries with exponential backoff, covering ~60s restart window)
- **Active sessions**: Session state is persisted in PostgreSQL. No data loss. Frontend re-associates with the session after reconnection
- **Recordings**: Active recordings may have a brief gap (~30s). go2rtc buffers RTSP frames during restart; Django reconnects to the buffer on restart

### 12.5 go2rtc Unreachable

- Camera tiles show: "Video Stream Unavailable — go2rtc service is offline. Detection continues via direct RTSP." (if backend can still process RTSP directly)
- Or: "Video & Detection Unavailable — go2rtc service is offline. Contact administrator."
- Health dashboard shows `detection_pipeline: down`

### 12.6 Long Sessions (>8 hours)

- **File size**: Recordings are split into 1-hour segments automatically (Celery task creates new Recording object every hour, ends previous one)
- **Memory**: Frontend detection store holds only latest frame per camera — no memory growth
- **DB partitions**: DetectionFrame partitioned by date (one partition per day). No single-partition bloat for long sessions

### 12.7 Variable Frame Rates

- **<1 FPS**: Frontend shows last known frame with "Low Frame Rate" warning badge on camera tile
- **>30 FPS**: Frontend's rAF batching naturally down-samples to display refresh rate. No action needed
- **Backend**: Detection pipeline processes at configurable rate (default: every 3rd frame for >15fps sources)

### 12.8 PostgreSQL Partition Auto-Creation

- **Strategy**: Partitions are pre-created by a Celery periodic task (`ensure_partitions`) that runs daily and creates partitions for the next 7 days
- **Fallback**: If a partition doesn't exist when data arrives, PostgreSQL's default partition (if configured) catches it, or the insert fails with a clear error logged. The next `ensure_partitions` run creates the missing partition

### 12.9 Student Re-Identification (Frontend)

- When backend detects a tracking ID change (student re-identified):
  - WebSocket sends `detection.tracking_update` message: `{ "old_id": "S-0012", "new_id": "S-0013", "camera_id": "uuid" }`
  - Predictions Panel: Old StudentCard fades out (300ms), new card appears with a "Re-identified" badge (auto-dismisses after 10s)
  - Bounding Box: Color briefly flashes to indicate ID change (200ms white flash)
  - Toast: "Student ID changed: S-0012 → S-0013 (possible re-identification)"

### 12.10 Bandwidth Degradation

- **Detection**: Frontend measures incoming frame rate per camera. If <5 FPS for >10 seconds, show warning
- **UI**: Yellow warning badge on affected camera tile: "Low Bandwidth — Reduced quality"
- **Action**: Info tooltip on badge: "Frame rate has dropped. Video quality may be reduced. Check network connection."
- **Resolution reduction**: Entirely backend-controlled (go2rtc transcoding). Frontend just displays what it receives

### 12.11 Mutual Exclusion Violation (FR-020)

- **Backend enforces**: Pyramid prediction constraints. If violation detected, the prediction `constraint_violation` field is `true`
- **Frontend display**: StudentCard shows an orange warning badge: "⚠ Conflicting Prediction" with tooltip explaining which fields conflict
- **No auto-correction**: Display the as-received prediction with the warning indicator

### 12.12 Browser Crash Recovery

- On page reload after crash: Frontend checks `GET /auth/me/` — if session still valid, auto-redirects to last known page (stored in `sessionStorage` if available, otherwise Dashboard)
- Active monitoring sessions persist server-side — instructor can navigate back to Camera Feed and see their session still running
- WebSocket reconnects automatically. Any anomaly triage done by other instructors during the crash is visible immediately

---

## 13. Performance & Measurement

### 13.1 15 FPS Canvas Rendering (SC-004)

- **Measurement**: Custom `PerformanceObserver` tracking `requestAnimationFrame` callback intervals. Metric: rolling average of last 60 frames. Reported to health store
- **Dashboard indicator**: Per-camera FPS shown in Health Dashboard
- **Target**: ≥15 FPS per camera canvas. If dropping below 10 FPS, reduce rendering quality (skip labels, reduce stroke width)

### 13.2 200ms UI Interactions (SC-003)

- **Scope**: Filter toggle, theme switch acknowledgment, triage button response, anomaly card expansion. Measured from `mousedown`/`keydown` event to next `requestAnimationFrame` callback showing updated UI
- **Not in scope**: API-dependent actions (network latency excluded). Those show loading states immediately

### 13.3 Page Load Budget

- **Total**: <3 seconds time-to-interactive on 4G connection
- **JS bundle cap**: 500KB gzipped (initial bundle). Code-split per route
- **CSS**: <50KB (tokens + component styles)
- **Fonts**: Inter (subset: latin, latin-ext) — ~30KB woff2
- **Initial API calls**: max 2 parallel requests (auth/me + page-specific data)

### 13.4 80% Coverage Gate

- **Measurement**: Single aggregate `vitest --coverage` metric across all frontend code
- **Not per-category** — a single 80% threshold simplifies CI/CD. Stores and hooks naturally get higher coverage; page components may be slightly lower
- **Backend**: Same approach — single `pytest --cov` aggregate

---

## 14. Setup & Infrastructure

### 14.1 Development-Ready Conditions

A machine is "development-ready" when all of the following succeed:
1. `docker compose ps` shows all services as "running" (or equivalently: PostgreSQL accepts connections on 5432, Redis PING returns PONG on 6379, go2rtc serves on 1984)
2. `python manage.py migrate --check` exits with code 0 (all migrations applied)
3. `python manage.py runserver 0.0.0.0:8000` starts without errors
4. `pytest --co -q` discovers tests without import errors

### 14.2 Setup Failure Criteria

A setup is "blocked" when any of:
- PostgreSQL/Redis connection refused after service start → error message with port and troubleshooting steps
- `pip install -r requirements.txt` fails → suggest checking Python version, virtualenv
- `python manage.py migrate` fails → suggest checking DB connection, running `docker compose logs db`

### 14.3 Docker vs Native Equivalence

Both paths produce equivalent results:
- Same ports: PostgreSQL 5432, Redis 6379, go2rtc 1984/8555
- Same data persistence: PostgreSQL data in Docker volume or system data directory
- Docker path is recommended (fewer manual steps). Native path documented for developers preferring system packages
- Tests pass identically on both paths

### 14.4 Port Collision Handling

- **Not auto-handled in v1**. If ports 5432/6379/1984/8555 are in use, Docker Compose will fail with a clear error
- **Remediation**: `.env` allows overriding: `POSTGRES_PORT=5433`, `REDIS_PORT=6380`, etc. Documented in quickstart troubleshooting section
- **Django dev server**: Default 8000, overridable via `PORT=8001`

### 14.5 Invalid Environment Values

- **Empty SECRET_KEY**: Django raises `ImproperlyConfigured` at startup with clear error message
- **Malformed URLs**: Database URL parsing fails with `dj-database-url` error. Redis URL parsing fails with `redis-py` error
- **Validation**: Settings module includes explicit checks for required environment variables with descriptive `ImproperlyConfigured` messages

### 14.6 Secret Handling

- `.env.example` contains placeholder values (e.g., `SECRET_KEY=change-me-in-production`)
- `.env` is in `.gitignore` — never committed
- Development defaults are safe for local use only (DEBUG=True, CORS_ALLOW_ALL=True)
- Production deployment requires real secrets via environment variables (not files)

### 14.7 External Dependency Assumptions

| Dependency | Assumption | Documented In |
|-----------|-----------|---------------|
| go2rtc | WHEP endpoint at `/api/ws?src={stream_name}`, REST API at `/api/streams` | research.md §R-011 |
| Redis | Single-node, localhost. No Sentinel/Cluster needed for <20 concurrent users | plan.md §Technical Context |
| PostgreSQL | Local auth (`trust` or `md5`), UTF-8 encoding, `uuid-ossp` extension available | quickstart.md §Prerequisites |

### 14.8 Observability Bootstrap

- **Phase 1 output**: Health endpoint skeleton (`GET /api/v1/health/` returns `{"status": "healthy"}`)
- **Logging**: Structured JSON logging via Python `logging` module with `pythonjsonlogger` formatter. Log format: `{"timestamp": "ISO8601", "level": "INFO", "logger": "app.module", "message": "...", "correlation_id": "uuid"}`
- **Correlation IDs**: Middleware generates a UUID per request, passed to log records and Celery tasks via `celery.utils.log`

---

## 15. Testing Strategy

### 15.1 Mock Strategy

| Layer | Mock Approach |
|-------|--------------|
| WebSocket hooks | `vitest` with `vi.fn()` mocking `WebSocket` constructor. Manual `onmessage` trigger |
| Zustand stores | Direct `useStore.setState()` in tests — no mocks needed |
| API services | `msw` (Mock Service Worker) intercepting fetch/axios requests |
| Canvas rendering | `vitest` with `jest-canvas-mock` for `getContext('2d')` |
| Router | `MemoryRouter` from `react-router-dom` |

### 15.2 E2E Traceability

| Playwright Test | User Story Scenarios Covered |
|----------------|------------------------------|
| T114: Login + Dashboard | US-1 scenarios 1-5 (login, error, redirect, theme persistence, dashboard) |
| T115: Camera Feed + Session | US-2 scenarios 1-4 (camera connect, grid layout, session start, detection display) |
| T116: Anomaly Triage | US-4 scenarios 1-3 (alert display, acknowledge, dismiss) |
| T117: Recording Playback | US-6 scenarios 1-3 (recording list, playback with overlays, timeline markers) |

Additional manual tests for US-5 (theme switching) and US-7 (camera management) per QA checklist.

### 15.3 TDD Enforceability

- **Process guideline only** — no CI enforcement of test-first ordering
- **Convention**: Each task in tasks.md lists both test and implementation sub-tasks. Developers write tests first per convention, verified in code review
- **CI gate**: `vitest --coverage` must pass with ≥80% on every PR

### 15.4 Backend API Mocks for Frontend Development

- **MSW (Mock Service Worker)**: `frontend/src/mocks/` directory with handlers for all REST API endpoints
- **Toggle**: `VITE_ENABLE_MOCKS=true` enables MSW in development
- **Contract alignment**: Mock response shapes match `rest-api.md` contract exactly

---

## 16. Miscellaneous

### 16.1 NFR Identification Scheme

Non-functional requirements are identified as:
- Performance: `PERF-{NNN}` (PERF-001: <3s page load, PERF-002: ≥15 FPS, PERF-003: <200ms interactions, PERF-004: <500ms seek, PERF-005: <2s anomaly alert)
- Security: `SEC-{NNN}` (SEC-001: session-based auth, SEC-002: RBAC, SEC-003: CSRF/CORS, SEC-004: brute-force protection, SEC-005: audit logging)
- Availability: `AVAIL-{NNN}` (AVAIL-001: 30s restart, AVAIL-002: data integrity on crash)
- Accessibility: `A11Y-{NNN}` (A11Y-001: WCAG 2.1 AA, A11Y-002: keyboard navigation, A11Y-003: screen reader)

### 16.2 Edge Case Traceability

| Edge Case | Related FRs | Test Coverage |
|-----------|-------------|---------------|
| Concurrent sessions | FR-004, FR-006 | Unit test: session creation with existing active session |
| Camera deduplication | FR-009, FR-026 | Unit test: recording reuse for shared cameras |
| First-write-wins triage | FR-041, FR-042 | Unit test: concurrent triage returns 409 |
| Detection service unavailable | FR-036 | E2E test: status banner with mocked WS |
| Browser crash recovery | FR-004, FR-019 | Manual test: session persistence after page refresh |
| Storage capacity warnings | FR-031 | Unit test: threshold calculation + banner display |
| Re-identification | FR-011 | Unit test: tracking ID update in detection store |
| Bandwidth degradation | FR-007 | Unit test: FPS monitoring + warning badge |
| Multi-tab support | FR-019 | Manual test: independent camera configs confirmed |

### 16.3 YOLO Model Compatibility

- **Model**: Custom YOLOv8 weights trained on exam monitoring data
- **Input resolution**: 640×640 (standard YOLO input, auto-resized by Ultralytics)
- **Confidence threshold**: 0.5 (configurable via `DETECTION_CONFIDENCE_THRESHOLD` env var)
- **Classes**: Defined by the trained model — the detection pipeline reads class names from the model metadata. Frontend displays whatever classes the model outputs
- **Fallback**: If model file not found at startup, detection pipeline starts in degraded mode (no inference, raw feed only)

### 16.4 Camera Dialog Validation

- **RTSP URL**: Regex validation pattern: `^rtsp://[^\s]+$`. Error: "Please enter a valid RTSP URL (e.g., rtsp://192.168.1.100:554/stream)"
- **Camera name**: Required, 1-100 characters. Error: "Camera name is required" / "Camera name must be under 100 characters"
- **Loading state during connection**: Button shows spinner + "Connecting..." text. Disable form inputs during connection attempt
- **Permission denied (local device)**: Show error: "Camera access denied. Please allow camera permissions in your browser settings." Provide link to browser permission instructions

### 16.5 Predictions Panel Sidebar

- **Trigger**: Toggle button (chevron icon) on the right edge of the camera feed area
- **Width**: 320px expanded, 0px collapsed (smooth slide animation, 200ms)
- **Persistence**: Expanded/collapsed state persisted to localStorage
- **On small screens** (<1200px): Full-width overlay from right, with backdrop

### 16.6 Minimum Viewport (FR-037)

- **Below 1920×1080**: Content scrolls horizontally and vertically. No scaling, no warning
- **Below 1024px**: Sidebar collapses to overlay drawer. Camera grid uses 1-column layout
- **Below 768px**: Single-column layout for all pages. Not a primary support target — informational warning on login: "For the best experience, use a screen resolution of at least 1920×1080"

### 16.7 Performance Under Load

- **Baseline (1 instructor × 1 camera)**: All metrics well under targets. <100ms API responses, <5 FPS detection (single camera)
- **Peak (20 instructors × 8 cameras)**: Per spec assumptions. Backend micro-batching keeps WS at ~8-10 msg/sec per connection. PostgreSQL handles ~2,400 detection inserts/sec via bulk_create
- **No intermediate load profiles defined** — peak targets are the design targets

---

## Appendix: Items Explicitly Deferred to v2

| Item | Reason |
|------|--------|
| Batch anomaly triage | Complexity of first-write-wins in batch mode |
| Admin rule engine UI | Rules editable via Django admin in v1 |
| Notification preferences | Out of scope — all instructors get all alerts |
| Configurable health refresh | Fixed 10s in v1, sufficient for monitoring use case |
| Per-category test coverage | Single aggregate threshold simpler to enforce |
| Stale data indicators on tab refocus | Zustand stores are updated in real-time via WebSocket — staleness unlikely |
| .env segmentation by profile | Single `.env.example` sufficient for grad project scope |
