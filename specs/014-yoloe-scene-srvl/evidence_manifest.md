# Evidence Manifest — 014-yoloe-scene-srvl

**Date:** 2026-06-07  
**Status:** `staged_local_only` — local implementation staged; production benchmark, figure lineage, and rollback evidence pending
**Branch:** `014-yoloe-scene-srvl`

---

## 1. Requirements-to-Evidence Map (T116)

| Requirement | Tasks | Evidence file / location | Status |
|---|---|---|---|
| FR-001 Disabled-by-default | T001, T018, T037, T048, T094, T104, T105 | `config.py`, `scene_lane.py`, `prod_rollback_yoloe_scene.sh`, CI env block | ✅ Local disabled-path implementation and rollback helper implemented; T120 production command output pending |
| FR-002 Offline-only | T003, T020, T033, T034 | `live_guard.py`, `scene_lane.py` | ✅ Implemented |
| FR-003 Prompt-locked export | T020-T022, T033 | `prompts.py`, export scripts | ✅ Implemented |
| FR-004 RLE-Zstd masks | T011, T014, T020-T022, T033-T034, T040-T042 | `masks.py`, `artifacts.py` | ✅ Implemented |
| FR-010 Append-only contradiction | T055-T060 | `contradictions.py`, `test_scene_recovery_append_only.py` | ✅ Implemented |
| FR-012 Idempotency keys | T013, T045, T065 | `persistence.py`, SHA-256 `scene:{job_id}:{frame_number}` | ✅ Implemented |
| FR-013 No duplicate rows | T045, T065 | `persistence.py` get_or_create | ✅ Implemented |
| FR-020 Scene-map renderer | T026, T027, T081-T083, T085-T088 | `SceneMapRenderer.tsx`, `sceneMetrics.ts`, `visualization_queue.py` | ✅ Implemented |
| FR-022 ≥30 FPS gate | T004, T075, T087, T093, T102 | `sceneRendererBenchmark.test.ts`, `prod_benchmark_scene_renderers.sh` | ✅ Harness ready; production run pending |
| FR-023 Absent optional fields no crash | T023-T027, T049, T074, T084-T086 | `OverlayCanvas.tsx`, `BoundingBoxCanvas.tsx`, type tests | ✅ Implemented |
| FR-025 Production helper scripts | T032, T098-T105 | `tools/prod/prod_*.sh|ps1|py` | ✅ Implemented |
| FR-026 Unavailable metric reasons | T028, T029, T088, T100 | `prod_collect_yoloe_scene_metrics.py` | ✅ Implemented |
| FR-027 Docs updated | T005-T007, T030-T031, T089-T090, T108-T116 | `docs/`, `specs/`, `cycle_014_yoloe_scene_srvl.md` | ✅ Local/staging docs implemented; T106-T107 production evidence pending |
| FR-028 Constitution evidence | T005-T006, T094-T096, T108-T119 | This manifest, cycle doc §5, §6 | ✅ Local/staging evidence implemented; production acceptance evidence pending |
| FR-029 Rollback proof | T037, T094, T104-T105 | `prod_rollback_yoloe_scene.sh`, `test_yoloe_scene_rollback.py` | ✅ Local rollback helper/test implemented; T120 production rollback output pending |
| FR-031 No unintended side effects | T008, T023, T035, T039 | Disabled-by-default contract, system tests | ✅ Implemented |

---

## 2. No Hidden XFails / Unsupported Paths (T117)

| Check | Finding |
|---|---|
| Hidden `xfail` markers | None — all scene test files inspected. Any skip uses `pytest.mark.skip` with explicit reason. |
| Live enablement V1 | Blocked by `live_guard.assert_offline` in `scene_lane.py`. No test enables live path. |
| SQLite fallback | None — all DB operations use `SceneFrameSummary.objects.*` Django ORM against PostgreSQL. No SQLite pragma or `using='sqlite'`. |
| Untracked CI-required files | All files listed in `yoloe-scene-srvl.yml` test steps exist. |
| Mock acceptance artifacts | No mock FPS measurements. `sceneRendererBenchmark.test.ts` tests the harness structure only; actual SC-010 gate requires a production run. |
| Local-only production claims | No production benchmark has been run yet. Cycle doc §6 is explicitly pending evidence, not an acceptance claim. |

---

## 3. Async Job / Idempotency Declarations (T118)

| Concern | Declaration |
|---|---|
| Stage deadlines | Scene lane has no stage deadline; it runs synchronously within the Celery task. SRVL is skipped if `SRVL_RUN_EVERY_N_FRAMES` cadence not met. |
| Reconciler behavior | No reconciler — persistence uses `get_or_create` idempotency (SHA-256 key). |
| Idempotency keys | `scene:{job_id}:{frame_number}` SHA-256 — declared in `persistence.py` and `test_yoloe_scene_reconciliation.py`. |
| Error-ratio thresholds | Not declared for V1. FR-028 requires this before V2 acceptance. |
| Vector payload guards | SRVL input limited to `MAX_OBJECTS=128`; excess objects are truncated with `truncated_count` recorded. |
| Re-run tests | `test_yoloe_scene_reconciliation.py` (T095) verifies no duplicate rows on re-run. |

---

## 4. Concurrency Acceptance Guard (T119)

No worker/thread/concurrency candidate in 014 is accepted without governed production benchmark evidence. The following concurrency-adjacent components are staged only:

| Component | Concurrency concern | Gate |
|---|---|---|
| `VisualizationQueue` | Multiple producers pushing frames | SC-011: benchmark required before frontend acceptance |
| SRVL NPZ atomic write | Concurrent readers / writers | Uses `os.replace` atomic rename — safe without additional lock |
| Django Channels WS broadcast | `async_to_sync` call in sync Celery | No load benchmark yet — acceptance requires SC-012 evidence |

**No acceptance claim is made for any of the above until production measurements confirm the gates.**

---

## 5. Final Rollback and Disabled-Path Proof (T120)

### Rollback command sequence (to be run on production server)

```bash
# Step 1: verify current state
echo "YOLOE_SCENE_ENABLED=${YOLOE_SCENE_ENABLED:-unset}"
echo "SRVL_ENABLED=${SRVL_ENABLED:-unset}"

# Step 2: dry-run rollback
bash tools/prod/prod_rollback_yoloe_scene.sh --dry-run

# Step 3: apply rollback
bash tools/prod/prod_rollback_yoloe_scene.sh --restart-workers

# Step 4: verify flags are 0
grep -E "YOLOE_SCENE_ENABLED|SRVL_ENABLED" .env
```

Expected output after rollback:
```
YOLOE_SCENE_ENABLED=0
SRVL_ENABLED=0
```

### Disabled-path proof

The system test `test_yoloe_scene_rollback.py::TestSceneRollback` verifies:
1. `run_scene_frame_lane` returns early without error when `YOLOE_SCENE_ENABLED=False`
2. No `SceneFrameSummary` rows are created
3. `_run_srvl_lane` is not called
4. `_broadcast_scene_event` is not called

Production rollback output: **PENDING** — must be run and linked here before acceptance.

---

## 6. Backend Test Suite Status (T113)

| Suite | Test file | Status |
|---|---|---|
| Unit / SRVL math | `test_srvl_math.py` | ✅ Implemented |
| Unit / SRVL vectorized | `test_srvl_vectorized_backend.py` | ✅ Implemented |
| Unit / SRVL artifacts | `test_srvl_artifacts.py` | ✅ Implemented |
| Unit / visualization queue | `test_scene_visualization_queue.py` | ✅ Implemented |
| Integration / detector assist | `test_scene_detector_assist.py` | ✅ Implemented |
| Integration / append-only | `test_scene_recovery_append_only.py` | ✅ Implemented |
| Integration / scene map video | `test_scene_map_video.py` | ✅ Implemented |
| System / rollback | `test_yoloe_scene_rollback.py` | ✅ Implemented |
| System / reconciliation | `test_yoloe_scene_reconciliation.py` | ✅ Implemented |
| System / evidence package | `test_yoloe_scene_evidence_package.py` | ✅ Implemented |
| Contract / production helpers | `test_yoloe_scene_production_helper_contract.py` | ✅ Implemented |

Actual test run on production server: **PENDING** — must produce passing output before acceptance.

---

## 7. Frontend Test Suite Status (T114)

| Test | File | Status |
|---|---|---|
| Scene types | `sceneTypes.test.ts` | ✅ Implemented |
| Renderer metrics | `sceneRendererMetrics.test.ts` | ✅ Implemented |
| Renderer benchmark harness | `sceneRendererBenchmark.test.ts` | ✅ Implemented |
| Playwright e2e scene map | `e2e/scene-map.spec.ts` | ✅ Implemented |

SC-010 PixiJS FPS gate: **PENDING** — requires production run.

---

## 8. Shell / PowerShell / Python Helper Checks (T115)

| Script | Syntax check | DRY_RUN | No-sudo | Status |
|---|---|---|---|---|
| `prod_run_yoloe_scene_benchmark.sh` | `bash -n` | ✅ | ✅ EUID | ✅ |
| `prod_benchmark_scene_renderers.sh` | `bash -n` | ✅ | ✅ EUID | ✅ |
| `prod_rollback_yoloe_scene.sh` | `bash -n` | ✅ | ✅ EUID | ✅ |
| `prod_run_yoloe_scene_benchmark.ps1` | `[switch]$DryRun` | ✅ | ✅ no sudo | ✅ |
| `prod_benchmark_scene_renderers.ps1` | `[switch]$DryRun` | ✅ | ✅ no sudo | ✅ |
| `prod_rollback_yoloe_scene.ps1` | `[switch]$DryRun` | ✅ | ✅ no sudo | ✅ |
| `prod_collect_yoloe_scene_metrics.py` | Python import | ✅ | ✅ geteuid | ✅ |
| `prod_generate_yoloe_scene_figures.py` | Python import | ✅ | ✅ geteuid | ✅ |

Automated checks: `test_scene_prod_helper_shell_syntax.py` (T089), `test_scene_prod_helper_powershell_syntax.py` (T090), `test_yoloe_scene_production_helper_contract.py` (T097).
