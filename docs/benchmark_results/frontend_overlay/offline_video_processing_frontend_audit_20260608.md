# Offline Video Processing Frontend Audit

**Last updated:** 2026-06-08
**Status:** frontend implementation and probe evidence; no inference benchmark decision.

This note records the current frontend work for the offline Video Processing
page and the Metrics & Analysis tab. It does not accept, reject, or replace an
inference optimization cycle.

## Source References

| Kind | Reference |
|---|---|
| File | `frontend/src/pages/VideoAnalysisPage.tsx` |
| File | `frontend/src/pages/VideoAnalysisPage.module.css` |
| File | `frontend/src/hooks/useVideoAnalysisLive.ts` |
| File | `frontend/src/components/telemetry/JobMetricsPanel.tsx` |
| File | `frontend/src/components/telemetry/WebGLTimeseriesPlot.tsx` |
| File | `frontend/src/components/telemetry/WebGLMatrixHeatmap.tsx` |
| File | `frontend/src/components/VideoPlayer/VideoPlayer.tsx` |
| File | `frontend/src/api/client.ts` |
| File | `frontend/src/stores/authStore.ts` |
| File | `tools/prod/prod_configure_ngrok_frontend.sh` |
| File | `tools/benchmarks/run_line_plot_renderer_benchmark.mjs` |
| Doc | `docs/benchmark_results/frontend_overlay/line_plot_renderer_latest.md` |
| Doc | `docs/production_inference_benchmark.md` |
| Job | `f6238d53-e86f-4bdf-b7c9-c5545f711911` |

## Requirement Audit

| Requirement | Current evidence | Status |
|---|---|---|
| Offline Video Processing page responds better to user clicks | `VideoAnalysisPage.module.css` reduces left-rail width pressure and uses auto-fit metadata; retry selection clears frame/playback state immediately in `VideoAnalysisPage.tsx`. | Implemented locally |
| Metrics & Analysis tab renders figures | `JobMetricsPanel.tsx` renders stage/span charts, distance/vector matrices, model output timelines, and pose timelines with guarded chart boundaries. | Implemented locally |
| Real-time WebGL plotting | `WebGLTimeseriesPlot.tsx` keeps one WebGL context, clips to the current time window, uses typed arrays, supports follow-live, pan, zoom, and per-student/per-signal toggles. | Implemented locally |
| Dense line plots stay responsive | `line_plot_renderer_latest.md` records a 192000-point synthetic probe: LOD path uploads 46080 vertices and reduces median total render time from 5.200 ms to 4.400 ms. | Probe evidence |
| Frames appear during preprocessing/inference | `useVideoAnalysisLive.ts` accepts `preview_frame` events with `frame_url` or `image_url`, updates progress without an image URL, and polls saved preview frames while processing. | Implemented locally |
| Original/annotated video toggle does not collapse | `VideoPlayer.tsx` uses a stable `videoShell` with a fixed fallback aspect ratio, syncs source playback state, and surfaces media errors. | Implemented locally |
| Matrix columns do not escape the overlay | `WebGLMatrixHeatmap.tsx` sizes the grid from the actual row/column count; `WebGLMatrixHeatmap.module.css` makes headers sticky inside a bounded scroll region. | Implemented locally |
| Expired ngrok session does not leave upload broken | `client.ts`, `authStore.ts`, and `LoginPage.tsx` convert protected auth `401/403` responses into a session-expired state; `prod_configure_ngrok_frontend.sh` now probes authenticated `/auth/me/`. | Implemented locally |

## Validation

| Check | Result |
|---|---|
| Focused Vitest suite | `50` tests passed across auth client/store/login, live preview hook, telemetry figures, and video player. |
| TypeScript app check | `npm run type-check` passed. |
| Vite production bundle | `npx vite build` passed. |
| ngrok helper syntax | `bash -n tools/prod/prod_configure_ngrok_frontend.sh` passed. |
| Playwright smoke | Mocked `/video-analysis` login/session flow rendered `2` time-series plots, `16` matrix cells, and `0` horizontal overflow at `1440x900`. |

`npm run build` still fails before Vite because of existing unrelated test
TypeScript issues in scene-map tests, scene renderer/type tests, GPU canvas
test mocks, recording canvas mocks, and `RuntimePage.kpiTruth.test.tsx`.

## Latest Production Bottleneck Evidence

Latest production evidence is job `f6238d53-e86f-4bdf-b7c9-c5545f711911` in
`docs/production_inference_benchmark.md` section 53.

| Metric | Last accepted baseline | No-sudo instrumented run | Delta |
|---|---:|---:|---:|
| DB-completed FPS | `5.680314` | `5.255214` | `-7.48 %` |
| Step 2 frame wall | `462.188348 s` | `462.996777 s` | `+0.17 %` |
| Step 2 through pose wall | `633.939294 s` | `705.429605 s` | `+11.28 %` |
| Step 2 pose tail only | `unavailable` | `242.432828 s` | new measurement |
| Behavior RTT mean | `84.171 ms` | `84.273 ms` | `+0.12 %` |
| GPU average utilization | `12.168 %` | `11.276 %` | `-7.33 %` |
| GPU peak utilization | `51.000 %` | `53.000 %` | `+3.92 %` |

The current bottleneck evidence points away from GPU compute saturation:
behavior RTT is flat, average GPU utilization remains low, and the larger
regression is in the through-pose wall. The last run is therefore documented as
an observability/reproducibility result, not a new accepted baseline.

## Remaining Production Step

Run `tools/prod/prod_configure_ngrok_frontend.sh` on production with the current
ngrok URL and admin credentials in environment variables. The helper now fails
if login succeeds but authenticated `/api/v1/auth/me/` still returns `403`.
