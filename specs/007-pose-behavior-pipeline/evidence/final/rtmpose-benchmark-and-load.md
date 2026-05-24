# RT103 Benchmark and Load

Status: Complete.

## Executed Commands
1. `k6 run --vus 50 --duration 5m load-tests/baseline.js`
2. `k6 run load-tests/scenarios/stress.js`
3. `locust -f load-tests/locustfile.py --headless -u 200 -r 20 -t 10m --processes -1`

## Results Summary
- k6 baseline passed: 297,192 iterations, 0 failed checks, p95 iteration duration `50.8371ms`.
- k6 stress passed: 421,480 iterations, 0 failed checks, p95 iteration duration `20.8552ms`.
- Locust headless multiprocess run completed with exit code `0` and clean shutdown at run-time limit (`600s`).

## Artifacts
- `backend/tests/system/artifacts/load/k6-baseline.log`
- `backend/tests/system/artifacts/load/k6-baseline-summary.json`
- `backend/tests/system/artifacts/load/k6-stress.log`
- `backend/tests/system/artifacts/load/k6-stress-summary.json`
- `backend/tests/system/artifacts/load/locust-headless.log`
- `load-tests/baseline.js`
- `load-tests/scenarios/stress.js`
- `load-tests/locustfile.py`

## Environment Notes
- `k6` was executed from a portable binary at `tools/k6/k6-v2.0.0-windows-amd64/k6.exe`.
- Windows native `locust --processes -1` is unsupported; the exact command was executed in a Linux container to preserve required flags and behavior.
