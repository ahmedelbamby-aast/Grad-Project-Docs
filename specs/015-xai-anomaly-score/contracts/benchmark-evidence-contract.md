# Contract: XAI Cycle Benchmark And Figure Evidence

**Contract ID**: `xai.benchmark.evidence.v1`

## Decision Authority

Only a completed native Linux RTX 5090 end-to-end production benchmark using
frame stride `1` may accept, reject, close, or replace a production baseline.
Local tests, component probes, direct model calls, reduced clips, and sampled
runs are probe-only.

## Throughput Target Authority

The current required target is `>=15 FPS` DB-completed end-to-end throughput on
canonical `combined.mp4` production runs at frame stride `1`. The practical
batch envelope is `32` frames completing their full authoritative cycle in
`<=2` seconds (>=16 FPS effective). The target is satisfied only after
inference, pose/behavior postprocess, PostgreSQL frame/detection/bounding-box
writes, embeddings and derived records, telemetry/artifacts, reconciliation,
and terminal lifecycle state are complete.

Every Cycle 015 decision record must include the observed DB-completed FPS and
the 32-frame full-cycle envelope status (`met`, `not_met`, or `unavailable`
with reason). Inference-only, frame-loop-only, progress-percent, direct Triton,
or GPU-only FPS cannot satisfy this target.

## Required Decision Record

| Field | Requirement |
|---|---|
| Cycle ID/state | Atomic cycle and decision |
| Baseline/candidate/best | Replay keys, job IDs, SHAs, exact media |
| Runtime truth | Environment fingerprint, route snapshot, profile, models |
| Config delta | One causal candidate variable |
| Performance | Live/cumulative/DB FPS, latency percentiles, throughput, phase wall |
| Throughput target | `>=15 FPS` DB-completed status, 32-frame full-cycle `<=2s` status, and unavailable reason if missing |
| Model/runtime | Per-model RTT/call rate/status/input shape, GPU/VRAM, CPU/RSS |
| Storage/queues | PostgreSQL/Redis latency/volume, queue/backpressure/retries |
| Correctness | Model output parity or task metrics, identity metrics |
| XAI quality | Calibration, fidelity, stability, sanity, explanation completeness |
| Anomaly pattern quality | Reconstruction, deterministic replay, controlled fixtures, metamorphic/invariant/sensitivity, cold-start/contamination/drift/quarantine, coverage/withholding metrics |
| Frontend | WebGL backend, frame/update latency, contexts, memory proxy |
| Lifecycle | Deadlines, reconciler, idempotency, failure accounting |
| Missing metrics | Explicit unavailable reason |
| Causal interpretation | What changed and what remains bottleneck |
| Rollback | Command, execution, and reconciliation proof |
| Evidence | Raw paths, manifests, digests, docs, ledger row |
| Knowledge limits | Proof of no anomaly training/fine-tuning path, no behavioral-ground-truth claim, and required observed-pattern vocabulary |

## Figure Roles

Before any decision:

- exactly one **Figure Planner** defines figures, source artifacts,
  unavailable-metric policy, and Markdown targets;
- exactly one **Figure Implementer** owns generator code/tests, produced
  artifacts, manifest, and digests;
- if one agent performs both, the results document explains why and preserves
  separate plan/implementation evidence.

## Minimum Figure Bundle

Every cycle includes:

- baseline/candidate/best metric delta;
- latency/throughput/phase wall;
- model RTT/call rate;
- GPU/VRAM and CPU/RSS;
- PostgreSQL/Redis/queue impact;
- correctness/identity or task-quality delta;
- cycle-specific XAI/anomaly/calibration/renderer quality;
- controlled pattern-fixture and invariant outcomes where anomaly behavior is
  in scope;
- unavailable-metric summary;
- evidence completeness;
- rollback verification.

Figures consume the same raw artifacts as the decision table. Missing values
are not plotted as zero.

## No-Ground-Truth Decision Boundary

Under this plan, no decision record may claim anomaly/cheating/normality
accuracy, precision, recall, F1, AUROC, AUPRC, false-positive rate, or
false-negative rate. Reviewer assessments, heuristic outputs, source-model
agreement, BSIL output, and assumed-normal history are not ground truth.
Controlled fixtures prove algorithm semantics only. Any unavailable
behavioral-validity metric is recorded as unavailable with
`no_accepted_behavioral_ground_truth`.

## Manifest

```json
{
  "schema_version": "xai.figure_manifest.v1",
  "cycle_id": "015.3",
  "decision": "NEEDS_FURTHER_ITERATION",
  "figure_planner": "named-owner",
  "figure_implementer": "named-owner",
  "inputs": [
    {"path": "raw.json", "sha256": "..."}
  ],
  "figures": [
    {"path": "adapter_latency.png", "sha256": "...", "inputs": ["raw.json"]}
  ],
  "unavailable_metrics": [
    {"metric": "example", "reason": "explicit reason"}
  ],
  "markdown_targets": ["docs/xai_anomaly/cycle_015_3_results.md"]
}
```
