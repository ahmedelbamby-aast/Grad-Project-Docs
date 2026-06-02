# Triton Strategy Gap Analysis Matrix

**Last updated:** 2026-05-15

## Objective
Compare recommended Triton optimization strategies against what is currently **Implemented**, **Partially Implemented**, or **Missing** in this codebase, then classify each into:
- **Must be Implemented**
- **Need to be Implemented**
- **Can be Implemented**
- **Optional**

---

## Current State Summary (Evidence-Based)

- Triton runtime integration and policy gating are implemented:
  - [runtime_policy.py](E:\grad_project\backend\apps\pipeline\services\runtime_policy.py)
  - [tasks.py](E:\grad_project\backend\apps\video_analysis\tasks.py)
- Triton deployment and metrics endpoints are implemented:
  - [docker-compose.dev.yml](E:\grad_project\docker-compose.dev.yml)
- Model configs are present, but **batching/instances/warmup/cache sections are absent**:
  - [person_detector config](E:\grad_project\backend\models\triton_repository\person_detector\config.pbtxt)
  - [posture_model config](E:\grad_project\backend\models\triton_repository\posture_model\config.pbtxt)
  - [gaze_horizontal config](E:\grad_project\backend\models\triton_repository\gaze_horizontal_model\config.pbtxt)
  - [gaze_vertical config](E:\grad_project\backend\models\triton_repository\gaze_vertical_model\config.pbtxt)
  - [gaze_depth config](E:\grad_project\backend\models\triton_repository\gaze_depth_model\config.pbtxt)
- Client protocol is HTTP-based Triton v2 (no gRPC client path yet):
  - [triton_client.py](E:\grad_project\backend\apps\pipeline\services\triton_client.py)

---

## Decision Matrix (Mermaid)

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart LR
  subgraph MUST["Must be Implemented (Critical)"]
    M1["Dynamic Batching Profiles<br/>Status: Missing<br/>Impact: Highest"]
    M2["Instance Group Sizing (per model/live-offline)<br/>Status: Missing<br/>Impact: High"]
    M3["Perf Analyzer + Model Analyzer Tuning Loop<br/>Status: Missing<br/>Impact: High"]
    M4["Live vs Offline Scheduling Policy in model configs<br/>Status: Missing<br/>Impact: High"]
  end

  subgraph NEED["Need to be Implemented (Important)"]
    N1["Model Warmup<br/>Status: Missing<br/>Impact: Medium-High"]
    N2["Rate Limiter + Cross-Model Priorities<br/>Status: Missing<br/>Impact: Medium-High"]
    N3["Explicit Memory Pool Tuning (Pinned/CUDA)<br/>Status: Missing<br/>Impact: Medium"]
    N4["Protocol Benchmarking (HTTP vs gRPC streaming)<br/>Status: Partial (HTTP only)"]
  end

  subgraph CAN["Can be Implemented (Context-dependent)"]
    C1["Response Cache<br/>Status: Missing<br/>Fit: Duplicate offline requests"]
    C2["Sequence Batcher for Stateful Temporal Models<br/>Status: Missing<br/>Fit: Stateful live pipelines"]
    C3["Separate Triton repos/config sets for Live/Offline<br/>Status: Partial (runtime policy only)"]
  end

  subgraph OPTIONAL["Optional (Scale/Infra-specific)"]
    O1["Multi-GPU placement optimization<br/>Status: Missing"]
    O2["MIG partition strategy<br/>Status: Missing"]
    O3["Autoscaling/K8s HPA by Triton metrics<br/>Status: Missing"]
  end
```

---

## Implementation Coverage Matrix

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TD
  A["Strategy Coverage"]
  A --> B1["Implemented"]
  A --> B2["Partially Implemented"]
  A --> B3["Missing"]

  B1 --> I1["Triton runtime routing + policy gates"]
  B1 --> I2["Triton metrics endpoint exposed"]
  B1 --> I3["TensorRT plan deployment in repo"]

  B2 --> P1["Workload differentiation (live/offline) at app layer<br/>No per-model scheduler configs yet"]
  B2 --> P2["Protocol path: HTTP implemented<br/>gRPC streaming not implemented"]

  B3 --> X1["dynamic_batching + preferred_batch_size + queue_delay"]
  B3 --> X2["instance_group sizing / per-GPU placement"]
  B3 --> X3["model_warmup blocks"]
  B3 --> X4["response_cache server/model config"]
  B3 --> X5["rate limiter policy config"]
  B3 --> X6["analyzer-driven performance optimization pipeline"]
  B3 --> X7["explicit pinned/cuda memory pool tuning"]
```

---

## Critical Decision Path (What to do first)

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'primaryColor': '#7C3AED', 'lineColor': '#A78BFA'}}}%%
flowchart TB
  S["Start: Need lower latency + higher throughput + better GPU utilization"] --> D1{"Are models configured for batching?"}
  D1 -- "No (current)" --> A1["Implement dynamic_batching + preferred_batch_size + queue_delay"]
  D1 -- "Yes" --> D2{"GPU underutilized?"}

  A1 --> D2
  D2 -- "Yes" --> A2["Tune instance_group counts and placement"]
  D2 -- "No" --> D3{"p95/p99 still high?"}

  A2 --> D3
  D3 -- "Yes" --> A3["Add model_warmup + rate limiter priorities"]
  D3 -- "No" --> D4{"Need further optimization?"}

  A3 --> D4
  D4 -- "Yes" --> A4["Run perf_analyzer + model_analyzer sweeps and lock best configs"]
  D4 -- "No" --> E["Stable operating point"]
```

---

## Strategy-by-Strategy Classification

| Strategy | Current Status | Classification | Why |
|---|---|---|---|
| Dynamic batching (`dynamic_batching`) | Missing | Must | Biggest throughput/utilization lever; currently `max_batch_size: 0` configs without batch scheduler sections |
| Per-model queue delay tuning | Missing | Must | Required to balance live latency vs offline throughput |
| Instance groups (`instance_group`) | Missing | Must | Needed for concurrency scaling and better GPU occupancy |
| Perf Analyzer + Model Analyzer loop | Missing | Must | No empirical config search workflow currently enforced |
| Live/Offline config split at model scheduler level | Partial | Must | App-level split exists, scheduler-level split absent |
| Model warmup (`model_warmup`) | Missing | Need | Reduces cold-start latency spikes |
| Rate limiter priorities | Missing | Need | Protects critical live models under contention |
| Explicit memory pool tuning | Missing | Need | Better transfer stability and memory behavior at scale |
| HTTP vs gRPC streaming benchmarking | Partial | Need | HTTP exists; no measured protocol decision for live streaming |
| Response cache | Missing | Can | Useful only when repeated inputs are common |
| Sequence batcher | Missing | Can | Valuable if stateful temporal inference is introduced |
| Separate Triton repos for profile-specific optimization | Partial | Can | Runtime policy exists; per-profile Triton configs do not |
| Multi-GPU placement/MIG/autoscaling | Missing | Optional | Infra-scale dependent, not mandatory for single-GPU setups |

---

## Recommended Immediate Execution Order

1. Implement dynamic batching + queue policies for each model.
2. Add instance groups and benchmark 1..N per model.
3. Run Perf Analyzer/Model Analyzer and lock latency/throughput SLO profiles for:
   - Live profile (low queue delay, low p99)
   - Offline profile (higher throughput, acceptable p99)
4. Add warmup and rate limiter.
5. Evaluate gRPC streaming for live workload.
