# OpenVINO v2 GPU Throughput Sweep Results

- Device: `GPU`
- Hint: `THROUGHPUT`
- Warmup: `2.0s`, Run: `8.0s` per case
- Generated at: `2026-05-22 19:40:31`

## Best Config Per Model

| Model | Best nstreams | Best nireq | FPS | Avg Latency (ms) |
|---|---:|---:|---:|---:|
| person_detector | - | - | 0.00 | 0.00 |
| posture_model | - | - | 0.00 | 0.00 |
| gaze_horizontal_model | - | - | 0.00 | 0.00 |
| gaze_depth_model | - | - | 0.00 | 0.00 |
| gaze_vertical_model | - | - | 0.00 | 0.00 |
| rtmpose_onnx | 2 | 16 | 13.53 | 1062.11 |
| rtmpose_model | 4 | 16 | 13.49 | 1065.13 |

## Full Sweep

| Model | nstreams | nireq | FPS | Avg Latency (ms) | Status | Note |
|---|---:|---:|---:|---:|---|---|
| person_detector | 1 | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | 1 | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | 1 | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | 1 | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | 1 | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | 2 | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | 2 | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | 2 | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | 2 | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | 2 | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | 4 | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | 4 | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | 4 | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | 4 | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | 4 | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | 8 | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | 8 | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | 8 | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | 8 | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | 8 | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | AUTO | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | AUTO | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | AUTO | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | AUTO | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| person_detector | AUTO | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | 1 | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | 1 | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | 1 | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | 1 | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | 1 | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | 2 | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | 2 | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | 2 | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | 2 | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | 2 | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | 4 | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | 4 | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | 4 | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | 4 | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | 4 | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | 8 | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | 8 | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | 8 | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | 8 | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | 8 | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | AUTO | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | AUTO | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | AUTO | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | AUTO | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| posture_model | AUTO | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | 1 | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | 1 | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | 1 | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | 1 | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | 1 | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | 2 | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | 2 | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | 2 | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | 2 | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | 2 | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | 4 | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | 4 | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | 4 | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | 4 | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | 4 | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | 8 | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | 8 | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | 8 | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | 8 | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | 8 | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | AUTO | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | AUTO | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | AUTO | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | AUTO | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_horizontal_model | AUTO | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | 1 | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | 1 | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | 1 | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | 1 | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | 1 | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | 2 | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | 2 | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | 2 | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | 2 | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | 2 | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | 4 | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | 4 | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | 4 | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | 4 | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | 4 | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | 8 | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | 8 | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | 8 | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | 8 | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | 8 | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | AUTO | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | AUTO | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | AUTO | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | AUTO | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_depth_model | AUTO | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | 1 | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | 1 | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | 1 | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | 1 | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | 1 | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | 2 | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | 2 | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | 2 | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | 2 | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | 2 | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | 4 | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | 4 | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | 4 | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | 4 | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | 4 | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | 8 | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | 8 | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | 8 | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | 8 | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | 8 | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | AUTO | 1 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | AUTO | 2 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | AUTO | 4 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | AUTO | 8 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| gaze_vertical_model | AUTO | 16 | 0.00 | 0.00 | error | Check 'backward_compatible_check // in_out_elements_equal' failed at src\core\shape_inference\include\reshape_shape_inference.hpp:375:
While validating node 'opset1::Reshape /model.6/m.0/m.0.0/attn/Reshape_4 (opset1::Transpose /model.6/m.0/m.0.0/attn/Transpose_5[0]:f32[4,1,8,32], opset1::Concat /model.6/m.0/m.0.0/attn/Concat_5_1[0]:i64[4]) -> (f32[?,1..,1..,256])' with friendly_name '/model.6/m.0/m.0.0/attn/Reshape_4':
Shape inference input shapes {[4,1,8,32],[4]}
Requested output shape [1,1,1,256] is incompatible with input shape
 |
| rtmpose_onnx | 1 | 1 | 0.99 | 1012.67 | ok |  |
| rtmpose_onnx | 1 | 2 | 1.85 | 1019.59 | ok |  |
| rtmpose_onnx | 1 | 4 | 3.57 | 1027.72 | ok |  |
| rtmpose_onnx | 1 | 8 | 6.93 | 1043.33 | ok |  |
| rtmpose_onnx | 1 | 16 | 13.37 | 1075.34 | ok |  |
| rtmpose_onnx | 2 | 1 | 0.99 | 1013.32 | ok |  |
| rtmpose_onnx | 2 | 2 | 1.86 | 1015.31 | ok |  |
| rtmpose_onnx | 2 | 4 | 3.59 | 1020.31 | ok |  |
| rtmpose_onnx | 2 | 8 | 7.00 | 1033.42 | ok |  |
| rtmpose_onnx | 2 | 16 | 13.53 | 1062.11 | ok |  |
| rtmpose_onnx | 4 | 1 | 0.98 | 1016.95 | ok |  |
| rtmpose_onnx | 4 | 2 | 1.85 | 1018.99 | ok |  |
| rtmpose_onnx | 4 | 4 | 3.60 | 1020.43 | ok |  |
| rtmpose_onnx | 4 | 8 | 6.99 | 1034.23 | ok |  |
| rtmpose_onnx | 4 | 16 | 13.52 | 1063.58 | ok |  |
| rtmpose_onnx | 8 | 1 | 0.98 | 1017.48 | ok |  |
| rtmpose_onnx | 8 | 2 | 1.85 | 1019.36 | ok |  |
| rtmpose_onnx | 8 | 4 | 3.58 | 1023.97 | ok |  |
| rtmpose_onnx | 8 | 8 | 6.96 | 1040.43 | ok |  |
| rtmpose_onnx | 8 | 16 | 13.53 | 1062.18 | ok |  |
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
| rtmpose_model | 1 | 1 | 0.99 | 1012.28 | ok |  |
| rtmpose_model | 1 | 2 | 1.86 | 1016.36 | ok |  |
| rtmpose_model | 1 | 4 | 3.56 | 1030.75 | ok |  |
| rtmpose_model | 1 | 8 | 6.93 | 1043.01 | ok |  |
| rtmpose_model | 1 | 16 | 13.38 | 1074.20 | ok |  |
| rtmpose_model | 2 | 1 | 0.98 | 1015.27 | ok |  |
| rtmpose_model | 2 | 2 | 1.86 | 1017.50 | ok |  |
| rtmpose_model | 2 | 4 | 3.58 | 1024.57 | ok |  |
| rtmpose_model | 2 | 8 | 6.97 | 1037.14 | ok |  |
| rtmpose_model | 2 | 16 | 13.44 | 1068.08 | ok |  |
| rtmpose_model | 4 | 1 | 0.99 | 1011.05 | ok |  |
| rtmpose_model | 4 | 2 | 1.86 | 1015.85 | ok |  |
| rtmpose_model | 4 | 4 | 3.59 | 1022.57 | ok |  |
| rtmpose_model | 4 | 8 | 6.97 | 1037.91 | ok |  |
| rtmpose_model | 4 | 16 | 13.49 | 1065.13 | ok |  |
| rtmpose_model | 8 | 1 | 0.99 | 1012.62 | ok |  |
| rtmpose_model | 8 | 2 | 1.86 | 1016.88 | ok |  |
| rtmpose_model | 8 | 4 | 3.59 | 1022.62 | ok |  |
| rtmpose_model | 8 | 8 | 6.96 | 1040.22 | ok |  |
| rtmpose_model | 8 | 16 | 13.45 | 1068.77 | ok |  |
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
