# Triton Stabilization Orchestrator Summary

- Run label: `verify_orch_ci`
- Mode: `dry-run`
- Status: `ok`
- Started: `2026-05-22T12:38:35.239684+00:00`
- Finished: `2026-05-22T12:38:35.239684+00:00`
- Steps: `4` total / `4` enabled

## Steps
- `phase0` | status=`planned` | enabled=`True`
  - command: `'E:\grad_project\.venv\Scripts\python.exe' 'E:\grad_project\scripts\ci\triton_phase0_benchmark_lock.py' --triton-url localhost:8000 --protocol http --measurement-interval-ms 10000 --gpu-window-sec 300 --report 'E:\grad_project\scripts\ci\artifacts\triton_stabilization\verify_orch_ci\phase0\triton_phase0_baseline_lock_report.md' --artifacts-dir 'E:\grad_project\scripts\ci\artifacts\triton_stabilization\verify_orch_ci\phase0\artifacts'`
  - outputs: `E:\grad_project\scripts\ci\artifacts\triton_stabilization\verify_orch_ci\phase0\triton_phase0_baseline_lock_report.md, E:\grad_project\scripts\ci\artifacts\triton_stabilization\verify_orch_ci\phase0\artifacts`
- `phase3` | status=`planned` | enabled=`True`
  - command: `'E:\grad_project\.venv\Scripts\python.exe' 'E:\grad_project\scripts\ci\triton_profile_jobs.py' 'scripts\ci\specs\triton_phase3_profile_jobs.yaml' --results-in 'scripts\ci\specs\triton_phase3_results_in_example.yaml' --results-out 'E:\grad_project\scripts\ci\artifacts\triton_stabilization\verify_orch_ci\phase3\triton_profile_filtered_results.json'`
  - outputs: `E:\grad_project\scripts\ci\artifacts\triton_stabilization\verify_orch_ci\phase3`
- `phase4` | status=`planned` | enabled=`True`
  - command: `'E:\grad_project\.venv\Scripts\python.exe' 'E:\grad_project\scripts\ci\run_triton_memory_pool_sweep.py' --spec 'E:\grad_project\scripts\ci\specs\triton_phase4_memory_pool_default.yaml' --artifacts-root 'E:\grad_project\scripts\ci\artifacts\triton_stabilization\verify_orch_ci' --run-label phase4_memory_pool --python-bin 'E:\grad_project\.venv\Scripts\python.exe'`
  - outputs: `E:\grad_project\scripts\ci\artifacts\triton_stabilization\verify_orch_ci\triton_phase4_memory_pool\phase4_memory_pool`
- `phase5` | status=`planned` | enabled=`True`
  - command: `'E:\grad_project\.venv\Scripts\python.exe' 'E:\grad_project\scripts\ci\protocol_decision_harness.py' --threshold-pct 10.0 --command-timeout-sec 600 --decision-report 'E:\grad_project\scripts\ci\artifacts\triton_stabilization\verify_orch_ci\phase5\protocol_decision_report.json' --summary-report 'E:\grad_project\scripts\ci\artifacts\triton_stabilization\verify_orch_ci\phase5\protocol_decision_summary.md' --http-artifact 'scripts\ci\specs\triton_phase5_http_artifact_example.json' --grpc-artifact 'scripts\ci\specs\triton_phase5_grpc_artifact_example.json'`
  - outputs: `E:\grad_project\scripts\ci\artifacts\triton_stabilization\verify_orch_ci\phase5\protocol_decision_report.json, E:\grad_project\scripts\ci\artifacts\triton_stabilization\verify_orch_ci\phase5\protocol_decision_summary.md`
