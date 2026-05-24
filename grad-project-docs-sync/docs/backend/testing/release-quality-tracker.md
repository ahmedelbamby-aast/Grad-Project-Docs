# Release Quality Tracker

**Feature**: Architecture Refactoring & Triton Transition (005)
**Template Version**: 1.0
**Updated**: 2026-05-07

## Purpose

Tracks quality metrics across releases to detect regressions early,
document historical baselines, and provide evidence for release gate decisions.

---

## Release Entry Template

Copy this template for each release cycle:

```markdown
### Release [VERSION] — [DATE]

**Branch**: [branch-name]
**CI Workflow Run**: [link]

#### Test Results

| Suite | Total | Passed | Failed | Skipped | Coverage |
|-------|-------|--------|--------|---------|----------|
| Unit | | | | | |
| Integration | | | | | |
| Contract | | | | | |
| System | | | | | |

#### Gate Status

| Gate | Status | Notes |
|------|--------|-------|
| Unit coverage ≥ 85% | ✅ / ❌ | |
| Integration tests PASS | ✅ / ❌ | |
| Contract tests PASS | ✅ / ❌ | |
| No previously-passing test regressed | ✅ / ❌ | |
| Dataset policy tests PASS | ✅ / ❌ | |

#### Inference Performance

| Metric | Value | Target | Status |
|--------|-------|--------|--------|
| P95 latency (ms) | | < 500ms | ✅ / ❌ |
| Error rate | | < 5% | ✅ / ❌ |
| Triton health | | ready | ✅ / ❌ |

#### Release Decision

- [ ] All CI gates PASS
- [ ] No open critical defects
- [ ] Release evidence uploaded to GitHub Actions artifacts
- [ ] Approved by: [name]

**Decision**: [ ] PROMOTE TO PRODUCTION  [ ] HOLD — INVESTIGATE
```

---

## Historical Releases

### Release 0.1.0 — 2026-05-07

**Branch**: `005-architecture-refactoring`
**Notes**: Initial Triton migration + CI pipeline bootstrap

#### Test Results

| Suite | Total | Passed | Failed | Skipped | Coverage |
|-------|-------|--------|--------|---------|----------|
| Unit | — | — | — | — | Pending |
| Integration | — | — | — | — | Pending |
| Contract | — | — | — | — | — |
| System | — | — | — | — | — |

> **Note**: Run `pytest` after activating the virtual environment to populate this table.
> See [onboarding-baseline.md](onboarding-baseline.md) for commands.

#### Gate Status

| Gate | Status | Notes |
|------|--------|-------|
| Unit coverage ≥ 85% | 🔄 Pending | Run full test suite |
| Integration tests PASS | 🔄 Pending | Run full test suite |
| Contract tests PASS | 🔄 Pending | Run full test suite |
| No regressions | 🔄 Pending | Compare against baseline |
| Dataset policy tests PASS | 🔄 Pending | DATASET_POLICY_BYPASS tests |

---

## Regression Tracking

| Issue | Detected | Fixed | Release | Root Cause |
|-------|----------|-------|---------|------------|
| — | — | — | — | — |

---

## Related Documents

- [onboarding-baseline.md](onboarding-baseline.md)
- [ci-policy.md](ci-policy.md)
- [real-data-test-policy.md](real-data-test-policy.md)
- [ARCHITECTURE.md](../../ARCHITECTURE.md)
- [COVERAGE_REPORT.md](../../../../COVERAGE_REPORT.md)
