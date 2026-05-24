# RTMPose TensorRT Conversion

Script: `backend/apps/pipeline/model_lifecycle/rtmpose_convert_tensorrt.py`

## Usage

```bash
python backend/apps/pipeline/model_lifecycle/rtmpose_convert_tensorrt.py \
  --onnx backend/models/rtmpose/rtmpose-m.onnx \
  --output-engine backend/models/rtmpose/tensorrt/rtmpose-m.engine
```

If `trtexec` is not in host PATH, the script falls back to `docker run rtmpose_exporter_lite:latest`.
