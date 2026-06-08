# Contract: Shared WebGL2 XAI Workbench

**Contract ID**: `xai.frontend.webgl.v1`
**Owner**: `frontend/src/services/webgl`

## Accepted Rendering Rule

Every analytical figure, plot, time series, matrix, heatmap, contribution
chart, attribution overlay, scene map, telemetry lane, and explanation graph
must report `backend=webgl2` for accepted production rendering.

When WebGL2 is unavailable, the UI retains accessible numeric/table evidence
and reports `webgl_unavailable`. Canvas2D is not presented as an accepted
analytical figure substitute.

## Required Behavioral Semantics

- Primary pattern states are `within_observed_pattern`, `pattern_deviation`,
  `insufficient_context`, and `withheld`.
- Every score/pattern view exposes the no-ground-truth knowledge limit.
- The workbench never presents pattern conformity as proof of normal,
  non-cheating, or innocent behavior.
- The workbench never presents pattern deviation or a high score as proof of
  abnormal intent, misconduct, or cheating.
- Reviewer feedback controls are described as operational assessments, never
  truth labels or anomaly-model training input.
- Per-student pattern-envelope, current-window, deviation-contribution,
  cold-start, contamination, drift, and quarantine views use the shared WebGL2
  renderer.

## Renderer Host Contract

```ts
interface RendererHost {
  readonly backend: 'webgl2' | 'unavailable'
  readonly contextId: string | null
  mount(target: HTMLCanvasElement): void
  render(scene: RenderScene, viewport: Viewport): RenderStats
  resize(widthCssPx: number, heightCssPx: number, dpr: number): void
  handleContextLoss(): void
  recoverContext(): Promise<boolean>
  dispose(): void
}
```

## Data Contract

```ts
interface BinaryArrayDescriptor {
  schemaVersion: 'xai.binary.array.v1'
  dtype: 'float32' | 'uint32' | 'uint16' | 'uint8'
  shape: readonly number[]
  byteOrder: 'little'
  digest: string
  sequenceStart: number
  sequenceEnd: number
}
```

Required stores:

- bounded typed-series ring store;
- tiled matrix store with LOD;
- graph node/edge store;
- attribution texture store;
- stable color registry keyed by canonical/source identity and behavior key.
- observed-pattern envelope/window/deviation stores keyed by compatible
  profile and feature-schema versions.

## Interaction Contract

- Pan and zoom do not rebuild all source data.
- Live follow tracks the latest source time until the user pans away.
- Visibility toggles update dirty layers only.
- Resize is stable and does not create a new context.
- Hover/pick is bounded and disabled/degraded under overload.
- Download exports the current figure/view plus lineage metadata.
- Student identity colors remain stable across related plots.
- Behavior colors are distinct from student identity colors.

## Performance Telemetry

Every view records:

- render backend and renderer version;
- frame-time p50/p95/p99;
- update-latency p50/p95/p99;
- context count and context losses/recoveries;
- vertices/cells/edges/textures rendered;
- bytes uploaded and buffer reuse;
- dropped/coalesced updates;
- visible range and source sample count;
- downsample/tile/LOD method;
- unavailable metrics and reasons.

## Required Budgets

Numerical budgets are set and accepted in Cycle 015.10 after representative
baseline measurement. The implementation must define gates for:

- context count;
- p95/p99 frame time;
- p95 update latency;
- main-thread long tasks;
- typed-array/buffer memory;
- upload bytes per frame;
- context-loss recovery time;
- maximum series samples and matrix cells retained;
- dropped update ratio under stress.

## Failure And Recovery

- `webglcontextlost` pauses rendering and preserves data stores.
- Recovery recreates programs/buffers from retained typed stores.
- Repeated loss beyond the configured limit marks rendering unavailable.
- A section-level renderer failure cannot crash the whole page.
- Stale rendering work may be dropped; authoritative data remains available.
