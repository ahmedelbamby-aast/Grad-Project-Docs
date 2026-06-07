# Research: YOLOE Scene Segmentation And SRVL

**Date**: 2026-06-07  
**Feature**: `014-yoloe-scene-srvl`

## Decision 1: V1 Runtime Scope

**Decision**: Build V1 for offline video processing only. Live stream enablement
is disabled and must return `disabled_for_live` or `unavailable` without
blocking the camera flow.

**Rationale**: The feature introduces full-frame scene segmentation, mask
artifact writing, relation matrices, and frontend rendering. Those operations
need measured latency, bounded queues, and no-seek stream proof before entering
the live profile.

**Alternatives considered**:

- Enable for live and offline together: rejected because live has stricter
per-frame latency, reconnect, and bounded queue invariants.
- Implement live first: rejected because the canonical acceptance video and
existing benchmark plan are offline.

## Decision 2: YOLOE Prompt-Locked Export

**Decision**: Use `yoloe-26s-seg.pt` as the default checkpoint and require the
complete prompt profile before any ONNX or TensorRT export. The export helper
must load `YOLOE`, resolve `.env` prompts, call `model.set_classes(...)`, then
export and write a prompt digest manifest.

**Rationale**: YOLOE exports are prompt-bound. A TensorRT or ONNX artifact built
before prompts are applied cannot be treated as the deployed classroom profile.
The default benchmark/test prompt profile is `classroom_roi_guard_v1`, while
users/developers can change the ordered class list from `.env` if they re-export
and revalidate.

**Alternatives considered**:

- Export base checkpoint first and apply prompts later: rejected because it
breaks prompt artifact lineage.
- Hardcode prompts in source only: rejected because the user requires a `.env`
override path for controlled experiments.

## Decision 3: YOLOE Instance Segmentation Output

**Decision**: Treat YOLOE-Seg detection and segmentation as one unified model
output. V1 normalization consumes and preserves every output emitted by the
runtime route: boxes, class IDs/names, confidence scores, masks, image/output
shape metadata, timing metadata, and raw-output artifact references when the
backend exposes them. Masks are expected through `results[0].masks` before
conversion into repository artifacts.

**Rationale**: YOLOE integrates instance segmentation by extending the detection
head with a mask prediction branch, similar to the YOLO segmentation family, but
for prompted classes. This avoids adding a separate segmentation model for
pixel-precise boundaries. Confidence scores are also decision evidence: non-ROI
contradictions, false-alarm handling, missing-person eligibility, and
visualization truth states need the original score plus the configured threshold
that interpreted it.

**Alternatives considered**:

- Run a separate segmentation model after detection: rejected for latency,
artifact duplication, and weaker prompt lineage.
- Store only boxes: rejected because non-ROI guards, missing-person localization,
and classroom maps require masks.
- Drop confidence scores after filtering: rejected because downstream decisions
must record the score, threshold, and reason instead of relying on a hidden
filter result.

## Decision 4: Artifact Format

**Decision**: Store compact relational summaries in PostgreSQL and store large
payloads as job-scoped artifacts. V1 masks use RLE-Zstd. Dense SRVL matrices use
compressed NPZ. Every artifact is referenced by a JSON manifest with SHA-256
digest, schema version, source frame, and size.

**Rationale**: Masks and `[N,N]` matrices can be too large for relational JSON
and too sensitive for public links. Sidecar artifacts preserve recoverability,
digest lineage, and access control while keeping database rows indexable.

**Alternatives considered**:

- Raw JSON masks/matrices in PostgreSQL: rejected for size and query overhead.
- Public static artifact URLs: rejected because artifacts contain classroom
imagery and must be job-scoped and authenticated.

## Decision 5: People Mismatch Recovery

**Decision**: Compare YOLOE person count against current student-plus-teacher
count. On mismatch, query Redis embedding evidence first, use deterministic
one-to-one assignment, classify each YOLOE person as matched existing,
false alarm/duplicate, missing candidate, or unresolved, and keep re-pass
disabled by default.

**Rationale**: Redis already stores fast per-job track embedding evidence. The
identity constitution forbids raw local-ID equality as proof, so recovery needs
appearance/geometry scoring and unresolved states. Candidate-only default avoids
retroactive mutation and timeline risk until a separate benchmark proves re-pass
correctness and latency.

**Alternatives considered**:

- Force-merge closest track labels: rejected by identity governance.
- Re-pass every unmatched YOLOE person immediately: rejected for V1 because it
changes downstream timeline behavior without measured proof.

## Decision 6: Non-ROI Guard Behavior

**Decision**: Segment non-ROI classroom regions and record append-only
contradiction events when downstream detections or recovery candidates strongly
overlap those regions. Do not delete, suppress, or rewrite the original
detection rows in V1.

**Rationale**: The guard is valuable as hallucination evidence, but mutating
existing detections would violate append-only evidence and rollback safety.

**Alternatives considered**:

- Suppress downstream boxes automatically: rejected for V1 because correctness
needs reviewer-label or production model-agreement evidence.
- Ignore non-ROI masks after rendering: rejected because the user specifically
needs hallucination checks.

## Decision 7: SRVL Compute Backend

**Decision**: Implement SRVL pairwise relations with vectorized GPU tensor
operations, starting with PyTorch CUDA. Full dense matrices are allowed up to
128 objects/frame; above that, the module must switch to top-k, thresholded, or
heatmap-only modes.

**Rationale**: Pairwise relations are `O(N^2)`. GPU broadcasting computes
`dx`, `dy`, distance, `atan2`, and vector stacking without Python pair loops.
The dense limit keeps memory and transfer costs bounded.

**Alternatives considered**:

- Python nested loops: rejected for latency and throughput.
- Custom Triton/CUDA kernel first: deferred until PyTorch CUDA benchmarks prove
the simpler path is insufficient.
- CPU NumPy production path: allowed only as fallback/test unless production
benchmarks accept it.

## Decision 8: Visualization Renderer

**Decision**: Benchmark PixiJS WebGL first for production frontend rendering.
Consider deck.gl or Three.js only if benchmark evidence rejects the simpler
PixiJS/WebGL texture path. Matplotlib is limited to offline debug/report output
with a non-interactive backend.

**Rationale**: The frontend is React/Vite and needs smooth 2D masks, heatmaps,
and overlays. PixiJS is a focused 2D GPU renderer and is less complex than
deck.gl or custom Three.js shaders for this V1 surface.

**Alternatives considered**:

- Backend-render every heatmap image: rejected for latency and transfer cost.
- Matplotlib production rendering: rejected as CPU-bound for real-time display.
- Three.js first: deferred until a simpler renderer fails with evidence.

## Decision 9: Benchmark And Metrics

**Decision**: Acceptance requires real native Linux RTX 5090 benchmarks on
`combined.mp4`, generated plots from the same raw artifacts as the decision
table, and explicit unavailable-metric reasons. Helper scripts may install or
use user-space metric tools, but not Docker or `sudo`.

**Rationale**: The repository constitution makes production benchmarks and
figure evidence binding for optimization decisions. The feature touches GPU
inference, Redis, PostgreSQL, frontend rendering, and artifact I/O, so component
probes cannot close acceptance.

**Alternatives considered**:

- Use local Windows or mock media as acceptance: rejected by governance.
- Hide unavailable GPU/power/frontend metrics: rejected; unavailable values must
be reported with reasons and not plotted as zero.

## Decision 10: Contract And Compatibility

**Decision**: Add versioned optional API, WebSocket, artifact, frontend, and
helper-command contracts. Existing clients/jobs must remain valid when scene
fields are absent.

**Rationale**: Scene fields are optional, feature-flagged, and artifact-backed.
Compatibility is safer than expanding existing serializers with unversioned
payloads or large embedded matrices.

**Alternatives considered**:

- Return full masks and matrices from job status by default: rejected for
payload size, privacy, and frontend responsiveness.
- Unversioned ad hoc JSON fields: rejected by contract/storage governance.

## Decision 11: Configuration Authority

**Decision**: Every operational YOLOE/SRVL/recovery/render/artifact value must
be declared through `.env` and parsed through the governed settings layer.
Feature code may define env key names and typed settings schemas, but it must
not hardcode prompt classes, confidence thresholds, overlap thresholds, score
weights, cadences, dense limits, top-k values, matrix modes, artifact codecs,
renderer choices, benchmark toggles, or recovery gates.

**Rationale**: The feature's decisions depend on configurable thresholds and
prompt choices. If those values are hardcoded in implementation code, benchmark
lineage, rollback, production reproducibility, and contradiction/recovery
evidence become untrustworthy. `.env` values and their fingerprint must be part
of every export, runtime, benchmark, and rollback evidence package.

**Alternatives considered**:

- Hardcode benchmark defaults in source: rejected because the user requires
all parameters to be `.env` declared and production evidence must name the
active values.
- Allow helper scripts to carry independent defaults: rejected because scripts
would drift from backend/frontend settings.
- Leave low-risk values such as dense limits or render modes as constants:
rejected because those values alter latency, artifact size, and decisions.
