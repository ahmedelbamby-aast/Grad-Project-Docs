# Triton Stabilization Orchestrator Summary

- Run label: `local_smoke`
- Mode: `dry-run`
- Status: `ok`
- Started: `2026-05-22T11:59:59.854582+00:00`
- Finished: `2026-05-22T11:59:59.855113+00:00`
- Steps: `4` total / `3` enabled

## Steps
- `phase0` | status=`planned` | enabled=`True`
  - command: `'E:\grad_project\.venv\Scripts\python.exe' 'E:\grad_project\scripts\ci\triton_phase0_benchmark_lock.py' --triton-url localhost:8000 --protocol http --measurement-interval-ms 10000 --gpu-window-sec 300 --report 'E:\grad_project\scripts\ci\artifacts\triton_stabilization\local_smoke\phase0\triton_phase0_baseline_lock_report.md' --artifacts-dir 'E:\grad_project\scripts\ci\artifacts\triton_stabilization\local_smoke\phase0\artifacts'`
  - outputs: `E:\grad_project\scripts\ci\artifacts\triton_stabilization\local_smoke\phase0\triton_phase0_baseline_lock_report.md, E:\grad_project\scripts\ci\artifacts\triton_stabilization\local_smoke\phase0\artifacts`
- `phase3` | status=`planned` | enabled=`True`
  - command: `'E:\grad_project\.venv\Scripts\python.exe' 'E:\grad_project\scripts\ci\triton_profile_jobs.py' 'E:\grad_project\scripts\ci\specs\triton_phase3_profile_jobs.yaml'`
  - outputs: `E:\grad_project\scripts\ci\artifacts\triton_stabilization\local_smoke\phase3`
- `phase4` | status=`planned` | enabled=`True`
  - command: `'E:\grad_project\.venv\Scripts\python.exe' 'E:\grad_project\scripts\ci\run_triton_memory_pool_sweep.py' --spec 'E:\grad_project\scripts\ci\specs\triton_phase4_memory_pool_default.yaml' --artifacts-root 'E:\grad_project\scripts\ci\artifacts\triton_stabilization\local_smoke' --run-label phase4_memory_pool --python-bin 'E:\grad_project\.venv\Scripts\python.exe'`
  - outputs: `E:\grad_project\scripts\ci\artifacts\triton_stabilization\local_smoke\triton_phase4_memory_pool\phase4_memory_pool`
- `phase5` | status=`skipped` | enabled=`False`
  - command: `'E:\grad_project\.venv\Scripts\python.exe' 'E:\grad_project\scripts\ci\protocol_decision_harness.py' --threshold-pct 10.0 --command-timeout-sec 600 --decision-report 'E:\grad_project\scripts\ci\artifacts\triton_stabilization\local_smoke\phase5\protocol_decision_report.json' --summary-report 'E:\grad_project\scripts\ci\artifacts\triton_stabilization\local_smoke\phase5\protocol_decision_summary.md'`
  - outputs: `E:\grad_project\scripts\ci\artifacts\triton_stabilization\local_smoke\phase5\protocol_decision_report.json, E:\grad_project\scripts\ci\artifacts\triton_stabilization\local_smoke\phase5\protocol_decision_summary.md`
