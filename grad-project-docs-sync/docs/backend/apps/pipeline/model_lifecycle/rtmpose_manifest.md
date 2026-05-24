# RTMPose Manifest

Module: `backend/apps/pipeline/model_lifecycle/rtmpose_manifest.py`

Tracks canonical artifact locations for RTMPose pipeline assets:
- `.pth`
- `.onnx`
- OpenVINO `.xml/.bin`
- TensorRT `.engine`

Use `write_manifest(model_dir)` to emit `rtmpose_manifest.json`.
