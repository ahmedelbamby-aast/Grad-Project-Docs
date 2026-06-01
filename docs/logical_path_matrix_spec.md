# Logical Path Matrix (LPM) — System Specification

**Status:** SPECIFICATION ONLY. No part of this is accepted, deployed, or
flagged successful until a production benchmark on the Linux RTX 5090 server
shows the documented metrics. See §10 acceptance gates.
**Date:** 2026-06-01
**Author placement:** new `apps/pipeline/services/logical_path_matrix.py` module
(Python-side first); BLS server-side migration tracked as a follow-up.
**Cycle slot:** Cycle 10 of the optimization plan (replaces the pose-only Cycle 10
in `docs/cycles_9_to_12_implementation_playbook.md`; the pose-parallelization
work moves to Cycle 10b once LPM is measured).

---

## 1. Problem Statement

The three gaze classifiers (`gaze_horizontal_model`, `gaze_vertical_model`,
`gaze_depth_model`) emit independent softmax decisions per detected person
per frame. Independence at inference time means the three decisions can:

1. **Contradict each other within a single axis** — both `LEFT` and `RIGHT`
   above threshold for the same person on the same frame.
2. **Oscillate frame-to-frame** — same person flipping `UP`/`DOWN` every few
   frames without a coherent physical motion.
3. **Conflict with the head pose** — e.g. the head-yaw from RTMPose says the
   person is turned strongly left, but `gaze_horizontal_model` returns `RIGHT`.

The Logical Path Matrix is a deterministic mathematical layer applied
**after** these three models emit their probabilities and **before** the
result is persisted, designed to:

- eliminate the impossible states above,
- normalize each person into one coherent `(H, V, D)` decision per frame,
- reduce downstream DB noise, embedding inconsistency, and tracking ambiguity.

---

## 2. HARD CONSTRAINTS (NON-NEGOTIABLE)

### 2.1 Scope — exactly three models

LPM applies **only** to:

| Model | Output axis | Classes |
|---|---|---|
| `gaze_horizontal_model` | H | `LEFT`, `RIGHT`, `UNKNOWN` |
| `gaze_vertical_model` | V | `UP`, `DOWN`, `UNKNOWN` |
| `gaze_depth_model` | D | `FORWARD`, `BACKWARD`, `UNKNOWN` |

### 2.2 Explicit EXCLUSIONS (do not modify)

LPM **must not** alter, gate, or override the outputs of:

| Model | Why excluded |
|---|---|
| `rtmpose_model` | Ground-truth physical pose. Used only as an *input signal* into the C4 constraint (head-yaw informs whether a horizontal-gaze transition is plausible). LPM never modifies pose keypoints, never substitutes pose outputs. |
| `person_detector` | Ground-truth detection. Bounding boxes / classes must pass through LPM unchanged. |
| `posture_model` (sitting_standing) | Ground-truth physical state. Not a directional behavior; cannot be normalized by directional constraints. |

If any code path in the LPM module reads or writes any of these three excluded
models' outputs, the regression unit tests in
`backend/tests/unit/pipeline/test_logical_path_matrix.py` MUST fail loudly.

---

## 3. Formal Mathematical Model

### 3.1 Per-person state definition

For each `(person p, frame f)`:

```
State(p, f) = ( H ∈ {LEFT, RIGHT, UNKNOWN},
                V ∈ {UP,   DOWN,  UNKNOWN},
                D ∈ {FORWARD, BACKWARD, UNKNOWN} )
```

### 3.2 Model probability outputs

Each of the three models emits a softmax-normalized 3-class vector
(`[positive_class, negative_class, unknown_class]`). Today the prod path
collapses these to argmax inside `_decode_yolo_output0`; LPM consumes the
**pre-argmax probabilities** so it can reason over confidences.

```
P_H(p, f) = [ p_left, p_right, p_unknown_h ]   with Σ = 1
P_V(p, f) = [ p_up,   p_down,  p_unknown_v ]   with Σ = 1
P_D(p, f) = [ p_fwd,  p_back,  p_unknown_d ]   with Σ = 1
```

### 3.3 Constraints

**C1 — Mutual exclusivity within each axis**

```
∀ axis a ∈ {H, V, D}:
    p_positive_a + p_negative_a ≤ 1
```

Violated when both classes are above the same threshold τ_axis. Enforced by
re-normalizing the axis vector so the dominant class wins and the loser is
demoted to `UNKNOWN` if the margin is below τ_margin (configurable, default
0.15).

**C2 — Temporal smoothness for the same person across frames**

```
∀ axis a, person p, frame f:
    || encode(S_a(p, f)) − encode(S_a(p, f-1)) || ≤ τ_temporal
```

where `encode(LEFT) = -1`, `encode(UNKNOWN) = 0`, `encode(RIGHT) = +1`
(and analogous for V and D), and τ_temporal is the per-axis switching budget
(default 1.0 — at most one categorical step per consecutive frame). A
violation triggers a **hysteresis check**: a flip from `LEFT` at frame f-1
to `RIGHT` at frame f is accepted only if `p_right` at frame f is at least
τ_temporal_strict above `p_left`. Otherwise the LPM keeps the prior state.

**C3 — Cross-class impossibility within an axis (strict)**

```
P(H = LEFT  ∧ H = RIGHT)    = 0
P(V = UP    ∧ V = DOWN)     = 0
P(D = FORWARD ∧ D = BACKWARD) = 0
```

C3 is a sanity check that follows from C1 after re-normalization. Surfaced
explicitly so the debug log can flag any axis where both probabilities exceed
0.5 at intake — that should be impossible if the upstream model is calibrated
and is treated as an alert, not as silent correction.

**C4 — Physical coupling using RTMPose head-yaw only**

```
let θ_yaw = head_yaw_from_rtmpose(p, f)
if |θ_yaw| > θ_pose_threshold:
    suppress improbable horizontal-gaze transitions
    i.e. transitions to the side opposite to head_yaw direction
        require a stricter p_threshold (τ_pose_override)
```

C4 reads RTMPose keypoints (nose / shoulders / ears) to estimate head-yaw in
degrees. `θ_pose_threshold` default 35°. C4 NEVER overwrites a pose
classification or detection — it only raises the bar for a directional gaze
transition to be accepted when the head is strongly turned. The pose itself
remains untouched.

### 3.4 Optimization Objective

For each `(p, f)` LPM chooses:

```
S*(p, f) = argmax_S [ Σ_a log P_a(S_a)    (model evidence)
                     − λ1 · Φ_C1(S)        (within-axis exclusivity penalty)
                     − λ2 · Φ_C2(S, S(p, f-1))  (temporal stability penalty)
                     − λ3 · Φ_C3(S)        (impossible-state penalty)
                     − λ4 · Φ_C4(S, θ_yaw) (pose-coupling penalty)
                   ]
```

The `Φ_*` functions are convex piecewise-linear penalties (margin-based, not
hard barriers, so LPM degrades gracefully on noisy input). All four `λ_i`
weights are read from settings (defaults in §5).

---

## 4. Architecture Placement

### 4.1 Phase 1 (this cycle) — Python-side post-processor

LPM is implemented as a Python module that runs **after** the four
behavior/gaze gRPC calls return and **before** detections are appended to
`fd.boxes`. Concretely, the hook lives in
`_run_crop_behaviour_for_items()` immediately after each crop's
`_decode_yolo_output0` call.

```
Frame
 → person_detector
 → crop generator
 → behavior_ensemble (4 sub-models; gives 4 dense outputs)
 → _decode_yolo_output0   (per crop, per model, gives [class_id, score, xyxy] list)
 → LPM ←─ rtmpose head_yaw (read-only)
 → _map_back → fd.boxes
 → tracking + persistence
```

Phase 1 is **flag-gated** by `LPM_ENABLED` (default `0` / off). When off, LPM
imports as a no-op so the existing path is bit-identical to today's behavior.

### 4.2 Phase 2 (future cycle) — BLS server-side

After Phase 1 has measured production impact, LPM moves into a Triton Python
backend (`triton_repository_cuda12/logical_path_matrix/`) that runs as the
final ensemble step. The same constraint math runs server-side so the dense
behavior outputs never have to be returned to Python at all. This dovetails
with **Option 1** of the Cycle 9b plan.

The Phase 1 code is structured so the constraint solver is a pure function
(`apply(per_person_probs, prior_state, pose_yaw, config) → decision`) with
no Django / Triton dependencies — that same function is what the Python
backend invokes.

---

## 5. Configurable Thresholds

```python
# backend/config/settings/base.py (Cycle 10)
LPM_ENABLED                  = bool(env "LPM_ENABLED",                 default 0)
LPM_LAMBDA_C1                = float(env "LPM_LAMBDA_C1",              default 0.6)
LPM_LAMBDA_C2                = float(env "LPM_LAMBDA_C2",              default 0.3)
LPM_LAMBDA_C3                = float(env "LPM_LAMBDA_C3",              default 0.8)
LPM_LAMBDA_C4                = float(env "LPM_LAMBDA_C4",              default 0.5)
LPM_THETA_POSE_THRESHOLD_DEG = float(env "LPM_THETA_POSE_THRESHOLD_DEG", default 35.0)
LPM_TAU_MARGIN               = float(env "LPM_TAU_MARGIN",             default 0.15)
LPM_TAU_TEMPORAL_STRICT      = float(env "LPM_TAU_TEMPORAL_STRICT",    default 0.25)
LPM_DEBUG_LOG                = bool(env "LPM_DEBUG_LOG",               default 0)
```

`LPM_DEBUG_LOG=1` emits one log line per LPM call summarizing pre-state,
post-state, eliminated contradictions, and constraint violations. Default OFF
in production; enabled for the parity benchmark only.

---

## 6. Inputs and Outputs

### 6.1 Inputs

```python
@dataclass
class LpmFrameInput:
    person_id: int                  # tracking_id; 0 if untracked
    h_probs: tuple[float, float, float]  # (left, right, unknown_h)
    v_probs: tuple[float, float, float]
    d_probs: tuple[float, float, float]
    rtmpose_head_yaw_deg: float | None    # signed yaw, None if pose unavailable
    prior_state: LpmState | None          # state at frame f-1 (per-track cache)
```

### 6.2 Output

```python
@dataclass
class LpmState:
    h: str  # one of "LEFT" | "RIGHT" | "UNKNOWN"
    v: str  # "UP" | "DOWN" | "UNKNOWN"
    d: str  # "FORWARD" | "BACKWARD" | "UNKNOWN"
    h_confidence: float                  # the winning class probability
    v_confidence: float
    d_confidence: float
    violations: list[str]                # constraint IDs that fired this frame
    eliminated_contradictions: int       # count of pre-LPM contradictions removed
```

The downstream `_map_back → fd.boxes` consumer reads `(h, v, d)` only; the
extra fields are for observability and telemetry.

---

## 7. Debug Mode Contract

When `LPM_DEBUG_LOG=1`:

```
[LPM] frame=137 person=42  yaw=+47.3°
      pre  : H=(left=.81 right=.79 unk=.00)  V=(up=.62 down=.04 unk=.34)  D=(fwd=.71 back=.65 unk=.00)
      post : H=LEFT(.81)  V=UP(.62)  D=FORWARD(.71)
      violations=[C1_H,C3_D]  eliminated_contradictions=2
```

The `violations` list must include the constraint identifier (`C1_H`,
`C1_V`, …, `C4_pose_override`) so the bench harness can count violations
per cycle and the LPM acceptance gate (§10.4) can verify the contradiction
count actually goes down.

---

## 8. Telemetry Contract

Add one row per frame-with-LPM-active to a new table
`telemetry_lpm_events` with columns:

| Column | Type | Note |
|---|---|---|
| session_id (FK telemetry_sessions) | UUID | |
| frame_index | int | |
| person_count | int | how many persons LPM saw this frame |
| eliminated_contradictions | int | sum across persons |
| violations_c1 | int | count of C1 fires this frame |
| violations_c2 | int | count of C2 fires |
| violations_c3 | int | count of C3 fires (alerts) |
| violations_c4 | int | count of C4 fires |
| latency_ms | float | LPM total time this frame |

This lets the acceptance gate compute the **contradiction reduction rate** and
the **LPM latency overhead** without log scraping.

---

## 9. Risk and Rollback

**Risk: low-to-medium.**

- LPM never touches detection / pose / tracker outputs; the worst case is that
  it changes which behavior box LANDS in `fd.boxes`, never which person /
  pose / xyxy exists.
- The constraint math is pure-Python and deterministic — same input always
  produces the same output.
- The hook into `_run_crop_behaviour_for_items` is flag-gated by
  `LPM_ENABLED`. When off, the existing path is byte-identical (verified by
  unit test `test_lpm_disabled_is_identity`).
- Phase 1 stays in Python; no Triton repository changes.

**Rollback:**

```bash
LPM_ENABLED=0
```

Single env flag. Restart the offline worker. No code revert needed.

---

## 10. Acceptance Gates (production-evidence only)

Cycle 10 (LPM) is **only** accepted when **every** gate below passes on a fresh
Linux RTX 5090 production benchmark run on `combined.mp4`:

### 10.1 Correctness (zero regression)

- Frame rows count unchanged vs. Cycle 8 baseline (`4 541`).
- Detection rows unchanged within ±0.05 % (BoT-SORT non-determinism only).
- Per-class bbox counts for `attention_tracking / hand_raising /
  person_detection / sitting_standing` unchanged within ±0.05 %. (LPM does
  not touch these classes; numbers must be effectively constant.)
- RTMPose `pose_record_count` unchanged within ±0.1 %.
- StudentTrack count unchanged.
- FrameEmbedding count unchanged within ±0.1 %.

### 10.2 LPM math correctness

- New `telemetry_lpm_events` rows show **C3 violation count = 0 after LPM**
  (because LPM enforces it as a hard constraint).
- C1 violation count is non-zero (it was firing pre-LPM) AND
  `eliminated_contradictions > 0` per affected frame.
- Pre-LPM vs post-LPM probability vectors logged in debug mode: every axis
  has a single class above 0.5 post-LPM.

### 10.3 Performance (must not regress)

- DB-completed FPS not lower than Cycle 9 (`4.09`).
- Step 2 wall not higher than Cycle 9 by more than 1 % (LPM is post-decode
  Python; cost target is < 5 ms / frame).
- GPU utilization unchanged (LPM is CPU-only).

### 10.4 Improvement (must show measurable behavioral signal)

- Per-track gaze-flip rate (count of frame-to-frame `H` changes per track,
  normalized) drops by at least 25 % vs Cycle 9.
- DB row count of impossible states (e.g. simultaneous `attention_left` and
  `attention_right` bboxes on the same Detection) drops to 0 in the post-LPM
  run. Pre-LPM, this count is the baseline measurement.

### 10.5 Operational

- `tools/prod/prod_parallel_flow_probe.sh` runs cleanly.
- `tools/prod/prod_verify_per_frame_signals.sh` runs cleanly.
- LPM-disabled rerun on the same SHA produces the same numbers as Cycle 9
  (the LPM-off identity property is preserved across deployments).

If **any** of §§10.1–10.5 fails, LPM is **REJECTED** or **NEEDS-MORE-DATA**,
and the env stays `LPM_ENABLED=0` in production. See the constitution
note at the bottom of `AGENTS.md`.

---

## 11. Implementation Plan (one focused change)

1. Add `backend/apps/pipeline/services/logical_path_matrix.py` with:
   - `LpmConfig` dataclass (reads settings).
   - `LpmState`, `LpmFrameInput` dataclasses.
   - `apply(inputs: list[LpmFrameInput], cfg: LpmConfig) → list[LpmState]`
     pure function implementing §3.4.
   - `estimate_head_yaw_deg(pose_keypoints) → float | None` helper using
     nose / left-ear / right-ear positions.
   - Module-level metrics: per-call latency and violation counts (exposed
     via prometheus client if available, otherwise debug log).
2. Add `LPM_*` settings to `backend/config/settings/base.py`.
3. Wire the hook into `apps/video_analysis/tasks.py:_run_crop_behaviour_for_items`
   immediately after the 4-model dispatch, behind `if settings.LPM_ENABLED:`.
4. Add `backend/tests/unit/pipeline/test_logical_path_matrix.py` with:
   - Disabled-flag identity test.
   - C1 mutual-exclusivity correctness test.
   - C2 temporal-smoothness test (per-person hysteresis).
   - C3 impossible-state detection test.
   - C4 pose-coupling test using a fake yaw value.
   - Optimization objective monotonicity test.
   - Debug-log structure test.
5. Add `telemetry_lpm_events` Django model + migration and wire
   `TelemetrySession.record_lpm_event(...)` through the JSON + PostgreSQL
   writer.
6. Update `.github/workflows/inference-parallelization.yml` to gate the
   new unit and integration tests.
7. Update `tools/prod/prod_enable_parallel_flow.sh` to set `LPM_ENABLED=0`
   explicitly (so the production env has the knob registered even before
   LPM is turned on).

No Triton config changes, no engine rebuilds, no Celery routing changes.
The change set is the LPM module, telemetry event storage, tests, one settings
block, and one hook in the existing dispatch function.

---

## 12. Out of Scope (will be a separate cycle)

- BLS Python backend implementation (Phase 2; depends on Phase 1
  measurement results).
- Cross-camera identity merging using LPM consistency (depends on the
  identity pipeline in §4 of the constitution).
- Tracker integration of LPM state into the cost matrix (BoT-SORT cost
  blending).
