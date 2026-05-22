# RTMPose OpenVINO Conversion

Script: `backend/apps/pipeline/model_lifecycle/rtmpose_convert_openvino.py`

## Usage

```bash
python backend/apps/pipeline/model_lifecycle/rtmpose_convert_openvino.py \
  --onnx backend/models/rtmpose/rtmpose-m.onnx \
  --output-xml backend/models/rtmpose/openvino/rtmpose-m.xml
```

Generates `rtmpose-m.xml` and `rtmpose-m.bin`.
