# OpenVINO v2 GPU Throughput Sweep Results

**Last updated:** 2026-05-22

- Device: `GPU`
- Hint: `THROUGHPUT`
- Warmup: `2.0s`, Run: `8.0s` per case
- Generated at: `2026-05-22 20:20:59`

## Best Config Per Model

| Model | Best nstreams | Best nireq | FPS | Avg Latency (ms) |
|---|---:|---:|---:|---:|
| person_detector | 1 | 1 | 19.98 | 49.44 |
| posture_model | 2 | 8 | 13.06 | 592.25 |
| gaze_horizontal_model | 2 | 2 | 13.07 | 151.36 |
| gaze_depth_model | 2 | 2 | 13.15 | 150.16 |
| gaze_vertical_model | 4 | 2 | 13.01 | 151.44 |
| rtmpose_onnx | 4 | 16 | 13.53 | 1062.04 |
| rtmpose_model | 2 | 16 | 13.56 | 1059.62 |

## Full Sweep

| Model | nstreams | nireq | FPS | Avg Latency (ms) | Status | Note |
|---|---:|---:|---:|---:|---|---|
| person_detector | 1 | 1 | 19.98 | 49.44 | ok |  |
| person_detector | 1 | 2 | 19.93 | 99.35 | ok |  |
| person_detector | 1 | 4 | 19.64 | 201.04 | ok |  |
| person_detector | 1 | 8 | 19.58 | 399.14 | ok |  |
| person_detector | 1 | 16 | 19.43 | 786.70 | ok |  |
| person_detector | 2 | 1 | 19.00 | 51.98 | ok |  |
| person_detector | 2 | 2 | 19.82 | 99.72 | ok |  |
| person_detector | 2 | 4 | 19.81 | 198.90 | ok |  |
| person_detector | 2 | 8 | 19.88 | 392.95 | ok |  |
| person_detector | 2 | 16 | 19.85 | 770.25 | ok |  |
| person_detector | 4 | 1 | 18.87 | 52.25 | ok |  |
| person_detector | 4 | 2 | 19.81 | 99.74 | ok |  |
| person_detector | 4 | 4 | 19.41 | 203.19 | ok |  |
| person_detector | 4 | 8 | 19.36 | 403.20 | ok |  |
| person_detector | 4 | 16 | 19.39 | 788.23 | ok |  |
| person_detector | 8 | 1 | 18.98 | 51.99 | ok |  |
| person_detector | 8 | 2 | 19.67 | 100.42 | ok |  |
| person_detector | 8 | 4 | 19.43 | 202.80 | ok |  |
| person_detector | 8 | 8 | 18.22 | 428.27 | ok |  |
| person_detector | 8 | 16 | 18.48 | 826.02 | ok |  |
| person_detector | AUTO | 1 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| person_detector | AUTO | 2 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| person_detector | AUTO | 4 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| person_detector | AUTO | 8 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| person_detector | AUTO | 16 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| posture_model | 1 | 1 | 12.75 | 77.82 | ok |  |
| posture_model | 1 | 2 | 12.58 | 157.38 | ok |  |
| posture_model | 1 | 4 | 12.68 | 310.24 | ok |  |
| posture_model | 1 | 8 | 12.64 | 612.03 | ok |  |
| posture_model | 1 | 16 | 12.55 | 1191.83 | ok |  |
| posture_model | 2 | 1 | 12.34 | 80.23 | ok |  |
| posture_model | 2 | 2 | 12.87 | 153.78 | ok |  |
| posture_model | 2 | 4 | 13.03 | 301.77 | ok |  |
| posture_model | 2 | 8 | 13.06 | 592.25 | ok |  |
| posture_model | 2 | 16 | 13.04 | 1149.54 | ok |  |
| posture_model | 4 | 1 | 12.52 | 79.11 | ok |  |
| posture_model | 4 | 2 | 13.03 | 151.88 | ok |  |
| posture_model | 4 | 4 | 12.73 | 308.77 | ok |  |
| posture_model | 4 | 8 | 12.76 | 606.09 | ok |  |
| posture_model | 4 | 16 | 12.87 | 1162.40 | ok |  |
| posture_model | 8 | 1 | 12.53 | 79.13 | ok |  |
| posture_model | 8 | 2 | 12.86 | 153.84 | ok |  |
| posture_model | 8 | 4 | 12.69 | 309.85 | ok |  |
| posture_model | 8 | 8 | 12.02 | 642.72 | ok |  |
| posture_model | 8 | 16 | 12.03 | 1241.43 | ok |  |
| posture_model | AUTO | 1 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| posture_model | AUTO | 2 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| posture_model | AUTO | 4 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| posture_model | AUTO | 8 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| posture_model | AUTO | 16 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| gaze_horizontal_model | 1 | 1 | 12.64 | 78.48 | ok |  |
| gaze_horizontal_model | 1 | 2 | 12.63 | 156.89 | ok |  |
| gaze_horizontal_model | 1 | 4 | 12.58 | 312.72 | ok |  |
| gaze_horizontal_model | 1 | 8 | 12.57 | 615.24 | ok |  |
| gaze_horizontal_model | 1 | 16 | 12.67 | 1181.04 | ok |  |
| gaze_horizontal_model | 2 | 1 | 12.53 | 79.11 | ok |  |
| gaze_horizontal_model | 2 | 2 | 13.07 | 151.36 | ok |  |
| gaze_horizontal_model | 2 | 4 | 12.96 | 303.38 | ok |  |
| gaze_horizontal_model | 2 | 8 | 12.97 | 596.39 | ok |  |
| gaze_horizontal_model | 2 | 16 | 12.99 | 1153.71 | ok |  |
| gaze_horizontal_model | 4 | 1 | 12.45 | 79.51 | ok |  |
| gaze_horizontal_model | 4 | 2 | 13.07 | 151.44 | ok |  |
| gaze_horizontal_model | 4 | 4 | 12.69 | 309.47 | ok |  |
| gaze_horizontal_model | 4 | 8 | 12.75 | 606.20 | ok |  |
| gaze_horizontal_model | 4 | 16 | 12.68 | 1179.50 | ok |  |
| gaze_horizontal_model | 8 | 1 | 10.36 | 94.96 | ok |  |
| gaze_horizontal_model | 8 | 2 | 12.23 | 160.54 | ok |  |
| gaze_horizontal_model | 8 | 4 | 11.71 | 333.89 | ok |  |
| gaze_horizontal_model | 8 | 8 | 12.58 | 612.78 | ok |  |
| gaze_horizontal_model | 8 | 16 | 12.11 | 1227.46 | ok |  |
| gaze_horizontal_model | AUTO | 1 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| gaze_horizontal_model | AUTO | 2 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| gaze_horizontal_model | AUTO | 4 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| gaze_horizontal_model | AUTO | 8 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| gaze_horizontal_model | AUTO | 16 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| gaze_depth_model | 1 | 1 | 11.80 | 83.56 | ok |  |
| gaze_depth_model | 1 | 2 | 11.19 | 176.14 | ok |  |
| gaze_depth_model | 1 | 4 | 12.25 | 320.56 | ok |  |
| gaze_depth_model | 1 | 8 | 12.22 | 631.65 | ok |  |
| gaze_depth_model | 1 | 16 | 12.09 | 1234.82 | ok |  |
| gaze_depth_model | 2 | 1 | 12.31 | 80.22 | ok |  |
| gaze_depth_model | 2 | 2 | 13.15 | 150.16 | ok |  |
| gaze_depth_model | 2 | 4 | 13.03 | 301.18 | ok |  |
| gaze_depth_model | 2 | 8 | 13.10 | 590.32 | ok |  |
| gaze_depth_model | 2 | 16 | 13.06 | 1145.90 | ok |  |
| gaze_depth_model | 4 | 1 | 12.20 | 80.89 | ok |  |
| gaze_depth_model | 4 | 2 | 13.10 | 150.72 | ok |  |
| gaze_depth_model | 4 | 4 | 12.88 | 304.16 | ok |  |
| gaze_depth_model | 4 | 8 | 13.02 | 593.91 | ok |  |
| gaze_depth_model | 4 | 16 | 13.03 | 1150.10 | ok |  |
| gaze_depth_model | 8 | 1 | 12.13 | 81.38 | ok |  |
| gaze_depth_model | 8 | 2 | 12.83 | 153.79 | ok |  |
| gaze_depth_model | 8 | 4 | 12.93 | 303.40 | ok |  |
| gaze_depth_model | 8 | 8 | 12.90 | 598.56 | ok |  |
| gaze_depth_model | 8 | 16 | 12.70 | 1176.64 | ok |  |
| gaze_depth_model | AUTO | 1 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| gaze_depth_model | AUTO | 2 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| gaze_depth_model | AUTO | 4 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| gaze_depth_model | AUTO | 8 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| gaze_depth_model | AUTO | 16 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| gaze_vertical_model | 1 | 1 | 12.06 | 81.98 | ok |  |
| gaze_vertical_model | 1 | 2 | 12.35 | 160.19 | ok |  |
| gaze_vertical_model | 1 | 4 | 12.11 | 324.55 | ok |  |
| gaze_vertical_model | 1 | 8 | 11.83 | 652.45 | ok |  |
| gaze_vertical_model | 1 | 16 | 11.71 | 1271.74 | ok |  |
| gaze_vertical_model | 2 | 1 | 12.12 | 81.37 | ok |  |
| gaze_vertical_model | 2 | 2 | 12.89 | 152.80 | ok |  |
| gaze_vertical_model | 2 | 4 | 13.01 | 301.65 | ok |  |
| gaze_vertical_model | 2 | 8 | 12.89 | 599.83 | ok |  |
| gaze_vertical_model | 2 | 16 | 12.90 | 1160.61 | ok |  |
| gaze_vertical_model | 4 | 1 | 12.10 | 81.54 | ok |  |
| gaze_vertical_model | 4 | 2 | 13.01 | 151.44 | ok |  |
| gaze_vertical_model | 4 | 4 | 12.84 | 305.43 | ok |  |
| gaze_vertical_model | 4 | 8 | 12.89 | 599.52 | ok |  |
| gaze_vertical_model | 4 | 16 | 12.88 | 1161.11 | ok |  |
| gaze_vertical_model | 8 | 1 | 12.22 | 80.86 | ok |  |
| gaze_vertical_model | 8 | 2 | 12.15 | 162.56 | ok |  |
| gaze_vertical_model | 8 | 4 | 12.31 | 319.20 | ok |  |
| gaze_vertical_model | 8 | 8 | 12.66 | 610.98 | ok |  |
| gaze_vertical_model | 8 | 16 | 12.29 | 1215.41 | ok |  |
| gaze_vertical_model | AUTO | 1 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| gaze_vertical_model | AUTO | 2 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| gaze_vertical_model | AUTO | 4 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| gaze_vertical_model | AUTO | 8 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| gaze_vertical_model | AUTO | 16 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| rtmpose_onnx | 1 | 1 | 0.98 | 1017.45 | ok |  |
| rtmpose_onnx | 1 | 2 | 1.86 | 1016.98 | ok |  |
| rtmpose_onnx | 1 | 4 | 3.57 | 1026.19 | ok |  |
| rtmpose_onnx | 1 | 8 | 6.90 | 1048.33 | ok |  |
| rtmpose_onnx | 1 | 16 | 13.37 | 1075.27 | ok |  |
| rtmpose_onnx | 2 | 1 | 0.98 | 1015.31 | ok |  |
| rtmpose_onnx | 2 | 2 | 1.85 | 1019.32 | ok |  |
| rtmpose_onnx | 2 | 4 | 3.58 | 1024.25 | ok |  |
| rtmpose_onnx | 2 | 8 | 6.97 | 1037.86 | ok |  |
| rtmpose_onnx | 2 | 16 | 13.48 | 1065.10 | ok |  |
| rtmpose_onnx | 4 | 1 | 0.98 | 1018.12 | ok |  |
| rtmpose_onnx | 4 | 2 | 1.86 | 1017.15 | ok |  |
| rtmpose_onnx | 4 | 4 | 3.59 | 1021.49 | ok |  |
| rtmpose_onnx | 4 | 8 | 6.98 | 1035.52 | ok |  |
| rtmpose_onnx | 4 | 16 | 13.53 | 1062.04 | ok |  |
| rtmpose_onnx | 8 | 1 | 0.99 | 1012.98 | ok |  |
| rtmpose_onnx | 8 | 2 | 1.86 | 1016.80 | ok |  |
| rtmpose_onnx | 8 | 4 | 3.58 | 1023.99 | ok |  |
| rtmpose_onnx | 8 | 8 | 6.95 | 1041.64 | ok |  |
| rtmpose_onnx | 8 | 16 | 13.47 | 1067.03 | ok |  |
| rtmpose_onnx | AUTO | 1 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| rtmpose_onnx | AUTO | 2 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| rtmpose_onnx | AUTO | 4 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| rtmpose_onnx | AUTO | 8 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| rtmpose_onnx | AUTO | 16 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| rtmpose_model | 1 | 1 | 0.99 | 1013.54 | ok |  |
| rtmpose_model | 1 | 2 | 1.86 | 1015.67 | ok |  |
| rtmpose_model | 1 | 4 | 3.57 | 1028.04 | ok |  |
| rtmpose_model | 1 | 8 | 6.91 | 1046.71 | ok |  |
| rtmpose_model | 1 | 16 | 13.36 | 1075.36 | ok |  |
| rtmpose_model | 2 | 1 | 0.98 | 1016.57 | ok |  |
| rtmpose_model | 2 | 2 | 1.86 | 1017.68 | ok |  |
| rtmpose_model | 2 | 4 | 3.58 | 1024.03 | ok |  |
| rtmpose_model | 2 | 8 | 6.99 | 1033.82 | ok |  |
| rtmpose_model | 2 | 16 | 13.56 | 1059.62 | ok |  |
| rtmpose_model | 4 | 1 | 0.99 | 1013.25 | ok |  |
| rtmpose_model | 4 | 2 | 1.86 | 1016.75 | ok |  |
| rtmpose_model | 4 | 4 | 3.58 | 1024.75 | ok |  |
| rtmpose_model | 4 | 8 | 6.96 | 1039.35 | ok |  |
| rtmpose_model | 4 | 16 | 13.56 | 1059.69 | ok |  |
| rtmpose_model | 8 | 1 | 0.99 | 1014.46 | ok |  |
| rtmpose_model | 8 | 2 | 1.86 | 1015.37 | ok |  |
| rtmpose_model | 8 | 4 | 3.57 | 1026.26 | ok |  |
| rtmpose_model | 8 | 8 | 6.95 | 1040.83 | ok |  |
| rtmpose_model | 8 | 16 | 13.48 | 1066.68 | ok |  |
| rtmpose_model | AUTO | 1 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| rtmpose_model | AUTO | 2 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| rtmpose_model | AUTO | 4 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| rtmpose_model | AUTO | 8 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
| rtmpose_model | AUTO | 16 | 0.00 | 0.00 | error | Exception from src\inference\src\cpp\compiled_model.cpp:128:
Check '!m_graphs.empty()' failed at src\plugins\intel_gpu\src\plugin\compiled_model.cpp:301:
[GPU] Model not loaded

 |
