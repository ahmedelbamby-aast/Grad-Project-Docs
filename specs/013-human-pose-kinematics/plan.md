# Implementation Plan: Human Pose Kinematics Layer

**Branch**: `013-human-pose-kinematics` | **Date**: 2026-06-05 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `specs/013-human-pose-kinematics/spec.md`

## Summary

Add a deterministic Human Pose Kinematics Layer after the existing RTMPose
keypoint runtime and before higher-level fusion. The layer converts raw
per-track keypoints into compact, fusion-ready pose mechanics evidence:
quality, normalized geometry, anchors, joint angles, limb vectors, head/torso
orientation, posture, hand-raise evidence, physical validity, bounded
correction/smoothing, contradiction monitoring, and governed override events.

The implementation must not introduce a new pose model or a second pose
authority. It consumes the current RTMPose output from
`backend/apps/pipeline/services/pose_runtime.py` and the existing pose artifact
surface in `backend/apps/video_analysis/tasks.py`. It persists compact
relational evidence in PostgreSQL and writes full keypoint arrays to
versioned artifacts/exports. All override thresholds are configuration-driven
through `.env`/Django settings.

## Technical Context

**Language/Version**: Python 3.12 backend; Django/Celery runtime; TypeScript/React frontend only for display/export surfacing if tasks require it.  
**Primary Dependencies**: Existing Django ORM, Celery, PostgreSQL, Redis, NumPy/OpenCV pose preprocessing, RTMPose through Triton, telemetry app. No new inference model dependency is planned.  
**Storage**: PostgreSQL is authoritative for compact summaries and override events. Full raw/normalized/corrected/smoothed keypoint arrays are stored in versioned artifacts/exports under the existing job artifact model. SQLite is forbidden.  
**Testing**: `pytest` against PostgreSQL semantics; unit tests for geometry/quality/correction/override gates; integration tests for offline replay artifacts and API/export contract; production benchmark validation on the Linux RTX 5090 workflow before any maturity claim.  
**Target Platform**: Native Linux production server with RTX 5090 for production validation; Windows/local validation is contract-only and cannot close production gates.  
**Project Type**: Django web-service plus Celery video inference pipeline and artifact/export surface.  
**Performance Goals**: Kinematics computation must be cheaper than the measured RTMPose tail and must be reported separately as `pose_kinematics_ms`, `pose_kinematics_ms_per_subject`, and per-phase latency. Offline acceptance requires a production Linux RTX 5090 baseline-disabled versus candidate-enabled matrix on `combined.mp4`, with no material regression in Step 2 through-pose wall, DB FPS, model agreement, or completion status. Live acceptance requires a real-media live-profile run or governed live-capture manifest through the active production inference path.
**Constraints**: Stream-safe bounded state only; per-track history is capped at `POSE_KINEMATICS_HISTORY_SECONDS` with default `5`; no whole-video dependency; no backward seek; no unbounded frame buffering; no retroactive live mutation; failures produce `unavailable`/`degraded` evidence instead of blocking the camera or fabricating pose mechanics.  
**Scale/Scope**: One pose mechanics record per pose-eligible `(job/session, camera, frame, tracking_id)` pair. Full keypoint arrays remain artifact-backed to avoid large DB JSON write amplification.  
**Runtime Scenarios**: Offline file replay and live RTSP/RTSPS/WHEP/WebRTC bridge scenarios both apply. Offline may replay deterministically but still uses at most the configured bounded history. Live uses current frame plus bounded same-track history and must expose drop/gap/degraded states.  
**Inference/Tracking Reference**: RTMPose remains the keypoint source via `PoseRuntimeResult` (`keypoints_xy`, `keypoint_scores`, `tracking_id`, `bbox_xyxy`). Existing `StudentTrack`, `PoseStreamRecord`, and pose artifact paths are integration references.  
**Runtime Authority**: Production validation uses Triton-only GPU inference with a single active endpoint profile. The kinematics layer is post-inference deterministic logic and must record active profile, model route, config digest, and artifact lineage in evidence.  
**Temporal/Identity Authority**: Each output carries frame index, timestamp/source time where available, session/camera/job scope, tracking identity, history window seconds, history sample count, continuity/gap state, and whether the identity is canonical, local, unresolved, or degraded.  
**Evidence/Schema Authority**: Schema version `pose_kinematics.v1` governs relational summaries, artifacts, telemetry rows, and export/API fields. Contract changes require migration/export/API compatibility tests.  
**Deployment Topology**: No Docker or sudo assumption for production. Feature flags are read from `backend/.env` through `backend/config/settings/base.py`; production launch remains governed by existing Triton/Celery scripts.  
**Runtime Reconciliation**: Job metadata, DB summaries, artifacts, telemetry, and frontend/export payloads must converge. Invalid or missing pose generates an explicit state and reason. Override events preserve original model prediction, corrected pose basis, confidence values, bounded-history evidence, spike-vs-contradiction classification, and root-cause fields.  
**Lineage/Fingerprints**: Evidence records include feature version, code SHA, `.env`/config digest, RTMPose model identity, source media digest or stream identity, job/replay key, artifact digest, and telemetry artifact path.  
**Budgets/SLOs**: Defaults are `POSE_KINEMATICS_ENABLED=1` by the 2026-06-05 initial production enablement exception, `POSE_KINEMATICS_HISTORY_SECONDS=5`, `POSE_KINEMATICS_OVERRIDE_MARGIN=0.15`, and `POSE_KINEMATICS_OVERRIDE_MIN_FRAMES=3`. Reviewer-label agreement and live validation remain open acceptance gates before final Cycle 013 acceptance.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design.*

| Gate | Status | Plan response |
| --- | --- | --- |
| Production runtime authority | PASS | No alternate pose inference authority is introduced. Production validation remains native Linux RTX 5090 plus Triton-only RTMPose. |
| Heterogeneous runtime maturity | PASS | Local tests validate contracts only; throughput, latency, and acceptance require a real production benchmark. |
| Temporal and identity truth | PASS | Outputs are scoped by job/session/camera/frame/tracking identity and record bounded history, gaps, and continuity state. |
| Pose and behavior semantics | PASS | Raw, normalized, corrected, smoothed, and display keypoint views are separate. Pose supports or overrides fusion only through explicit evidence. |
| Queue and failure | PASS | No worker, thread, pool, or queue topology change is planned. Live failures are fail-stop/degraded at the evidence boundary, not silent success. |
| Concurrency scaling authority | PASS | N/A. The feature does not increase Celery workers, threads, in-process batch concurrency, or GPU concurrency caps. |
| Contract and storage | PASS | `pose_kinematics.v1` contracts govern compact DB summaries, artifacts, overrides, telemetry, and exports. |
| Observability and scientific evidence | PASS | Telemetry must report per-phase and per-record cost, unavailable/degraded counts, contradiction/override counters, and artifact paths. |
| Production benchmark authority | PASS | No optimization/quality decision can be accepted from probes alone. The benchmark table must include baseline/candidate replay/job, FPS, Step 2 through-pose wall, RTT/model metrics, GPU, memory, DB parity, model agreement, and evidence paths. |
| Live/offline validation | PASS | Both scenarios are first-class; live compatibility is `stream-safe-with-config` using bounded per-track history. |
| Anti-regression runtime truth | PASS | Validation must reconcile process/profile, DB rows, telemetry, artifacts, and frontend/export states. |
| Evidence integrity and lineage | PASS | Artifacts must be digest-addressed and immutable after publication; placeholder evidence cannot close the feature. |
| Runtime job lifecycle/vector integrity | PASS | No embedding/vector persistence is changed. New durable writes need idempotency keys and stage outcome accounting. |
| Documentation anti-hallucination | PASS | Plan artifacts avoid unsourced production claims and reference real code paths only where inspected. |

**Post-design re-check**: PASS. Phase 1 artifacts keep the layer
deterministic, bounded, config-driven, stream-compatible, and off by default.
No unresolved clarification markers remain.

## Project Structure

### Documentation (this feature)

```text
specs/013-human-pose-kinematics/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── pose-kinematics-contract.md
└── checklists/
    └── requirements.md
```

### Source Code (repository root)

```text
backend/
├── apps/
│   ├── pipeline/
│   │   └── services/
│   │       ├── pose_runtime.py
│   │       ├── rtmpose_pipeline.py
│   │       └── logical_path_matrix.py
│   ├── video_analysis/
│   │   ├── models.py
│   │   ├── tasks.py
│   │   ├── views.py
│   │   └── services/
│   │       └── pose_quality.py
│   └── telemetry/
│       ├── models.py
│       ├── session.py
│       └── writer.py
├── config/
│   └── settings/base.py
└── tests/
    ├── unit/
    ├── integration/
    └── contract/

tools/
└── prod/
    ├── prod_collect_benchmark_metrics.py
    ├── prod_watch_benchmark_metrics.sh
    └── prod_enable_parallel_flow.sh
```

**Structure Decision**: Use the existing Django backend and pose runtime
boundaries. The first implementation should add deterministic kinematics
services under `backend/apps/pipeline/services/` or
`backend/apps/video_analysis/services/`, wire compact persistence through
`backend/apps/video_analysis/models.py`, expose/export via existing
`video_analysis` views/artifact metadata, and extend telemetry/benchmark
scripts only for measured evidence.

## Phase 0: Research Output

Research decisions are recorded in [research.md](research.md). They resolve:

- layer placement and authority;
- bounded temporal history;
- persistence split between compact DB summary and artifact-backed arrays;
- configurable override gate defaults;
- live-stream compatibility;
- benchmark and observability requirements.

## Phase 1: Design Output

Design artifacts:

- [data-model.md](data-model.md)
- [contracts/pose-kinematics-contract.md](contracts/pose-kinematics-contract.md)
- [quickstart.md](quickstart.md)

## Implementation Strategy

1. Add configuration keys in `backend/config/settings/base.py` with safe bounds:
   `POSE_KINEMATICS_ENABLED`, `POSE_KINEMATICS_HISTORY_SECONDS`,
   `POSE_KINEMATICS_OVERRIDE_MARGIN`,
   `POSE_KINEMATICS_OVERRIDE_MIN_FRAMES`,
   `POSE_KINEMATICS_MIN_KEYPOINT_CONFIDENCE`,
   `POSE_KINEMATICS_ARTIFACTS_ENABLED`,
   `POSE_KINEMATICS_TELEMETRY_ENABLED`.
2. Add pure deterministic kinematics computation with no Django/Triton import
   dependency at module import time.
3. Add bounded same-track temporal state with eviction by both time and sample
   count. State must be scoped by session/camera/job plus tracking identity.
4. Persist compact summaries and override events idempotently; keep full arrays
   artifact-backed and digest-addressed.
5. Wire offline pose artifact enrichment after RTMPose output creation and
   before fusion/export.
6. Wire live pose payload enrichment without blocking the live frame path and
   without retroactive mutation.
7. Extend telemetry and benchmark collection with kinematics latency,
   counts, states, contradiction categories, override counts, history sizes,
   and unavailable/degraded reasons.
8. Add reviewer-labeled validation manifest support for at least 500
   person-frame samples and enforce posture, hand-raise, torso/head
   orientation, and attention-support agreement thresholds in a durable report.
9. Add production validation wrappers for both the offline baseline/candidate
   matrix and a real-media live-profile run, each recording replay/session IDs,
   source media or stream manifest, env/config delta, deployed SHA, telemetry,
   artifact paths, and terminal status.
10. Add runtime reconciliation evidence that joins task state, queue state,
    PostgreSQL rows, artifact manifests, telemetry counters, and API/export
    payloads before the feature can be accepted.
11. Add rollback and disabled-feature verification proving existing pose and
    behavior flows continue with explicit `unavailable` kinematics evidence
    rather than false success.

## Validation Plan

| Layer | Required validation |
| --- | --- |
| Unit | Keypoint quality labels, anchors, geometry, joint angles, orientation, posture, hand raising, missing-joint estimation, physical-validity rules, smoothing, spike detection, override gate config. |
| Contract | `pose_kinematics.v1` payload examples validate required states and no hardcoded thresholds. |
| Integration | Offline replay produces one state per pose-eligible track/frame and preserves raw/corrected/smoothed separation in artifact/export evidence. |
| Live compatibility | Simulated RTSP/live payload uses current frame plus bounded history only; overflow/gap/degraded states are emitted. Acceptance also requires a production live-profile real-media run or governed live-capture manifest that records session identity, latency budget compliance, frame/drop/gap counts, unavailable/degraded reasons, and artifact paths. |
| Reviewer labels | A manifest-backed validation report covers at least 500 person-frame samples and records posture, hand-raise, torso/head orientation, and attention-support agreement against the configured thresholds. |
| Offline benchmark | Production Linux RTX 5090 baseline-disabled and candidate-enabled runs on `Raw Data/Diverse Classroom Enviroments/combined.mp4`; compare replay/job IDs, FPS, Step 2 through-pose wall, kinematics latency, GPU, memory, DB parity, model agreement, and artifact completeness. |
| Runtime reconciliation | Production evidence joins task state, queue state, PostgreSQL rows, artifact manifests, telemetry counters, and API/export payloads; acceptance is blocked if any state is unreconciled. |
| Rollback/disabled behavior | Production or production-equivalent evidence proves disabling the layer preserves existing pose/behavior flows and emits explicit `unavailable` kinematics state. |

## Benchmark Decision Table Template

No decision may be made until this table is filled by a completed production
benchmark.

| Metric | Baseline | Candidate | Delta | Decision reason |
| --- | --- | --- | --- | --- |
| Replay key / job id | pending benchmark | pending benchmark | N/A | Must be real production Linux run. |
| Deployed SHA | pending benchmark | pending benchmark | N/A | Candidate SHA must match deployed code. |
| Video | `combined.mp4` | `combined.mp4` | N/A | Same video unless explicitly impossible. |
| Status / frames | pending benchmark | pending benchmark | N/A | Must complete or explain governed partial state. |
| DB completed FPS | pending benchmark | pending benchmark | pending benchmark | Throughput gate. |
| Step 2 through-pose wall | pending benchmark | pending benchmark | pending benchmark | Critical-path gate. |
| Pose kinematics ms/frame | N/A | pending benchmark | pending benchmark | New feature budget. |
| Pose kinematics ms/subject | N/A | pending benchmark | pending benchmark | Per-subject cost. |
| RTMPose RTT / p95 | pending benchmark | pending benchmark | pending benchmark | Must not hide pose runtime regressions. |
| GPU avg / peak | pending benchmark | pending benchmark | pending benchmark | Resource gate. |
| RSS / VRAM peak | pending benchmark | pending benchmark | pending benchmark | Memory gate. |
| DB row parity | pending benchmark | pending benchmark | pending benchmark | Correctness gate. |
| Model agreement | pending benchmark | pending benchmark | pending benchmark | Correctness gate. |
| Pose labeled accuracy | pending benchmark | pending benchmark | pending benchmark | Reviewer-label gate for posture, hand raising, torso/head orientation, and attention support. |
| Override count / categories | N/A | pending benchmark | pending benchmark | Monitoring gate. |
| Artifact completeness | pending benchmark | pending benchmark | pending benchmark | Evidence gate. |
| Runtime reconciliation | pending benchmark | pending benchmark | pending benchmark | Task, queue, DB, telemetry, artifact, and API/export convergence gate. |
| Rollback / disabled behavior | pending benchmark | pending benchmark | pending benchmark | Existing flows must continue with explicit unavailable state. |
| Live production validation | pending live run | pending live run | pending live run | Real-media live-profile or governed live-capture-manifest gate. |

## Complexity Tracking

No constitutional violation is currently justified. The plan intentionally
avoids new inference stages, new worker topology, offline-only buffering, and
hardcoded decision thresholds.
