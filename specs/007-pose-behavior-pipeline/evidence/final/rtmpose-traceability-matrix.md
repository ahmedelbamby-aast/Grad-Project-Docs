# RT107 Traceability Matrix

Status: Updated with final evidence for RT102-RT108.

| RT ID | Code Path | Test | Evidence |
|---|---|---|---|
| RT001 | n/a | n/a | `rtmpose-scope-lock.md` |
| RT002 | n/a | n/a | `rtmpose-task-evidence-index.md` |
| RT003 | `backend/apps/pipeline/results.py` baseline contract | n/a | `rtmpose-baseline-runtime-payloads.md` |
| RT004 | n/a | n/a | `rtmpose-baseline-backend-tests.md` |
| RT005 | n/a | n/a | `rtmpose-baseline-frontend-tests.md` |
| RT067 | `backend/apps/pipeline/utils.py::temporal_stability_score` | `backend/tests/system/test_temporal_stability_score.py` | `rtmpose-test-results.md` TODO |
| RT068 | `backend/apps/pipeline/utils.py::moving_average_denoise`, `noise_reduction_score` | `backend/tests/system/test_temporal_noise_reduction_score.py` | `rtmpose-test-results.md` TODO |
| RT069 | `backend/apps/pipeline/utils.py::temporal_continuity_score` | `backend/tests/system/test_temporal_continuity_score.py` | `rtmpose-test-results.md` TODO |
| RT070 | `backend/apps/pipeline/results.py::aggregate_scores` | `backend/tests/system/test_multi_person_long_session_continuity.py` | `rtmpose-test-results.md` TODO |
| RT071 | `backend/apps/pipeline/results.py::export_contrastive_tensor_sample` | `backend/tests/system/test_contrastive_tensor_export_sanity.py` | `rtmpose-test-results.md` TODO |
| RT072 | `backend/apps/pipeline/utils.py` | Covered by RT067-RT069 tests | `rtmpose-scientific-validation-method.md` |
| RT073 | `backend/apps/pipeline/results.py` | Covered by RT070-RT071 tests | `rtmpose-scientific-validation-method.md` |
| RT074 | n/a | n/a | `rtmpose-scientific-validation-method.md` |
| RT102 | `load-tests/baseline.js` | `k6 run --vus 50 --duration 5m load-tests/baseline.js` | `backend/tests/system/artifacts/load/k6-baseline.log`, `backend/tests/system/artifacts/load/k6-baseline-summary.json` |
| RT103 | `load-tests/scenarios/stress.js` | `k6 run load-tests/scenarios/stress.js` | `backend/tests/system/artifacts/load/k6-stress.log`, `backend/tests/system/artifacts/load/k6-stress-summary.json`, `rtmpose-benchmark-and-load.md` |
| RT104 | `load-tests/locustfile.py` | `locust -f load-tests/locustfile.py --headless -u 200 -r 20 -t 10m --processes -1` | `backend/tests/system/artifacts/load/locust-headless.log`, `rtmpose-benchmark-and-load.md` |
| RT105 | `rtmpose-engineering-validation-report.md` | n/a | RT105 closure note in `rtmpose-engineering-validation-report.md` (only addressed `⚠️` allowed for flip policy) |
| RT106 | `rtmpose-engineering-validation-report.md` | n/a | RT106 closure note in `rtmpose-engineering-validation-report.md` confirms `❌` unchanged |
| RT108 | n/a | n/a | `rtmpose-release-readiness.md` |
