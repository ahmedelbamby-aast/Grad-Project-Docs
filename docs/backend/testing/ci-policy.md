# CI/CD Policy: Staged Gating Strategy
# Feature: Architecture Refactoring & Triton Transition
# Spec: /specs/005-architecture-refactoring/spec.md (FR-010, FR-023)

## Overview

This document defines the three-stage CI gating policy for the Exam Monitor backend.
The policy rolls out incrementally from zero pipelines to full blocking gates to avoid
disrupting delivery while building confidence in test coverage.

## Stages

### Stage 1 — Bootstrap (Informational)

**Trigger**: All pull requests on any branch  
**Blocks merge**: No  
**Purpose**: Establish CI baseline; catch obvious failures early  
**Workflow file**: `.github/workflows/ci-bootstrap.yml`

Tests included:
- All unit tests (`backend/tests/unit/`)
- All integration tests (`backend/tests/integration/`)

Pass criteria: Informational only. Results are reported but do not gate merge.

---

### Stage 2 — Blocking Gate (Merge Protection)

**Trigger**: Pull requests targeting protected branches (`main`, `release/*`)  
**Blocks merge**: Yes — merge is rejected if any test fails or coverage drops below threshold  
**Purpose**: Enforce quality bar on protected branches  
**Workflow file**: `.github/workflows/ci-blocking.yml`

Tests included:
- All unit tests (`backend/tests/unit/`)
- All integration tests (`backend/tests/integration/`)

Pass criteria:
- All unit and integration tests must pass
- Code coverage must be ≥ 85%
- No coverage regression > 5% from previous baseline

---

### Stage 3 — Staging System Tests (Release Gate)

**Trigger**: Release branches (`release/*`) and manual dispatch  
**Blocks release promotion**: Yes  
**Purpose**: Validate end-to-end behavior in a staging environment with real model serving  
**Workflow file**: `.github/workflows/ci-staging-system.yml`

Tests included:
- All system tests (`backend/tests/system/`)
- Contract alignment tests (`backend/tests/contract/`)

Pass criteria:
- All system and contract tests must pass
- Release gate evidence must be verified by `scripts/ci/verify_release_gate.py`
- No raw dataset present in staging/prod environments (DatasetPolicy enforcement)

---

## Branch Protection Configuration

| Branch pattern | Required stages | Who can bypass |
|----------------|-----------------|----------------|
| `main`         | Stage 1 + Stage 2 | Tech lead (emergency only) |
| `release/*`    | Stage 1 + Stage 2 + Stage 3 | None |
| `005-*`, feature branches | Stage 1 (informational) | Author |

---

## Coverage Thresholds

| Scope | Minimum | Block on failure |
|-------|---------|-----------------|
| Unit tests | 85% | Yes (Stage 2+) |
| Integration tests | 70% | Yes (Stage 2+) |
| System tests | Execution pass | Yes (Stage 3) |

---

## Dataset Policy Enforcement

Raw video test datasets are FORBIDDEN in production and staging-release environments.  
The `DatasetPolicy` guard fixture (`backend/tests/conftest.py`) must run on every test session.  
The release gate verifier (`scripts/ci/verify_release_gate.py`) checks that no raw dataset artifacts
are present in the deployment package before promotion.

---

## References

- [spec.md](../../specs/005-architecture-refactoring/spec.md) — FR-010, FR-023, FR-024
- [quickstart.md](../../specs/005-architecture-refactoring/quickstart.md) — CI Rollout Path
- [contracts/ci-test-gates-contract.md](../../specs/005-architecture-refactoring/contracts/ci-test-gates-contract.md)
