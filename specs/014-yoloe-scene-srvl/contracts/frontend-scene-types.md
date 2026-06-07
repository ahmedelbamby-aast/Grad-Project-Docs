# Contract: Frontend Scene Types

**Version**: `scene-frontend.v1`  
**Feature**: `014-yoloe-scene-srvl`

## TypeScript Interfaces

```typescript
export type SceneStatus =
  | "unknown"
  | "unavailable"
  | "degraded"
  | "valid"
  | "disabled_for_live";

export type SceneTruthState =
  | "valid"
  | "contradiction"
  | "recovery_candidate"
  | "unresolved"
  | "unavailable";

export interface ScenePromptProfile {
  name: string;
  classesSha256: string;
}

export interface SceneObjectObservation {
  frameLocalObjectId: number;
  classId: number;
  className: string;
  confidence: number;
  confidenceSource: "yoloe_score" | "calibrated_score" | "unavailable";
  confidenceThreshold?: {
    envKey: string;
    value: number;
  };
  bbox: [number, number, number, number];
  center: [number, number];
  anchor: [number, number];
  maskRef?: string;
  rawYoloeOutputRef?: string;
  unavailableOutputReasons?: Record<string, string>;
  isPerson: boolean;
  isNonRoi: boolean;
  truthState?: SceneTruthState;
}

export interface SceneRelationRefs {
  matrixMode: "full" | "thresholded" | "top_k" | "heatmap_only";
  objectCount: number;
  denseLimit: number;
  distanceMatrixRef?: string;
  angleMatrixRef?: string;
  vectorMatrixRef?: string;
  cartesianVectorRef?: string;
  distanceHeatmapRef?: string;
  angleMapRef?: string;
  correlationMapRef?: string;
}

export interface SceneFramePayload {
  schemaVersion: "scene-api.v1";
  jobId: string;
  frameNumber: number;
  timestampMs: number;
  sceneStatus: SceneStatus;
  promptProfile?: ScenePromptProfile;
  objects: SceneObjectObservation[];
  relations?: SceneRelationRefs;
  traceRef?: string;
}
```

## Rendering Rules

- Existing playback must work when every scene field is absent.
- Scene overlays must be independently toggleable from existing bounding boxes.
- PixiJS WebGL is the first production renderer candidate.
- Visualization input should use compact summaries, binary buffers, sparse
  top-k relation lists, downsampled heatmaps, or RGBA texture buffers.
- Do not stream huge `[N,N]` JSON matrices to the UI every frame.
- If the renderer falls behind, display the latest valid scene state and count
  dropped visualization frames.
- Full mask artifacts load only on explicit inspection, not by default playback.
- Confidence scores and threshold snapshots must be visible to inspection views
  and available to contradiction/recovery evidence panels.
- Frontend behavior reads renderer and visualization settings from API/settings
  values derived from `.env`; component code must not hardcode operational
  thresholds, class lists, matrix modes, renderer choices, or codecs.

## Required UI States

- `unknown`: old job or scene fields absent.
- `unavailable`: scene lane failed or did not run; show reason if available.
- `degraded`: partial scene data exists.
- `valid`: scene data and artifacts are ready.
- `contradiction`: downstream prediction overlaps non-ROI evidence.
- `recovery_candidate`: unmatched YOLOE person candidate exists.
- `unresolved`: mismatch was ambiguous and not force-merged.

## Frontend Metrics Contract

Renderer benchmark evidence must report:

- FPS p50/p95/p99 or unavailable reason.
- Frame time p50/p95/p99.
- Main-thread long task count.
- WebGL context loss count.
- Payload decode time.
- Dropped visualization frames.
- Memory growth.
- Screenshot evidence for desktop and constrained viewport.
- Active scene `.env` fingerprint and renderer env keys used for the run.
