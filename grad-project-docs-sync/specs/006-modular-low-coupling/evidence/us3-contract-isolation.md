# US3 Contract Isolation Evidence

## Scope

Representative module-swap and unrelated-workflow regression evidence for inference provider swaps, tracking output isolation, frontend API data-source adapters, and native Linux deployment separation.

## Verification

| Command | Result |
| --- | --- |
| `.venv\Scripts\python.exe -m pytest tests\integration\test_modular_inference_contract.py tests\unit\tracking\test_modular_contract_isolation.py tests\integration\test_modular_failure_boundaries.py tests\contract\test_native_linux_deployment_contract.py -q --tb=short` from `backend/` | Passed: 9 tests |
| `.venv\Scripts\python.exe -m compileall apps\detections apps\anomalies apps\pipeline apps\tracking apps\video_analysis` from `backend/` | Passed |
| `npm test -- tests\unit\api\backend-contracts.test.ts --run` from `frontend/` | Passed: 1 file, 2 tests |
| `npm run type-check` from `frontend/` | Passed |
| `python scripts\ci\verify_docs_diagrams.py` | Passed |

## Findings

The inference route table supports swapping Triton, mock, and future local providers behind `InferenceClientFactory`. Tracking results and rendering helpers now expose stable contract serializers. Frontend dashboard, detections, and video-analysis API modules normalize data-source responses before pages/stores consume them. Production Triton remains a native Linux service separate from Docker development topology.
