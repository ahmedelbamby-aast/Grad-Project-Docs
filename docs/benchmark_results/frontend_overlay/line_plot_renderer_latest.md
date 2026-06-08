# Line Plot Renderer Benchmark

Measured: 2026-06-08T13:22:15.749Z

This is a frontend renderer probe only. It does not create an inference benchmark decision.

| Profile | Vertices | Build ms median | Draw ms median | Total ms median |
|---|---:|---:|---:|---:|
| webgl_full_buffer_visible_clamped | 192000 | 5.000 | 0.300 | 5.200 |
| webgl_lod_minmax_visible_window | 46080 | 4.400 | 0.100 | 4.400 |

| Delta | Value |
|---|---:|
| LOD vertices / full vertices | 0.2400 |
| LOD total / full total | 0.8462 |
| Total median improvement | 15.38% |
| WebGPU available in benchmark browser | no |
