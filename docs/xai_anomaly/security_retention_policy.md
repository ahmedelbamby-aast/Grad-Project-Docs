# XAI Security Retention Policy

**Last updated:** 2026-06-10

This document defines the current feature-owned privacy, access, audit, and
retention policy for Cycle 015 XAI/anomaly artifacts. It is a setup/governance
artifact only: it records the current enforced boundaries and the remaining gaps
that still block full production acceptance for later cycles.

## Source-of-truth references

| Kind | Reference |
|---|---|
| File | `backend/apps/anomalies/views.py` |
| File | `backend/apps/anomalies/serializers.py` |
| File | `backend/apps/behavior/views.py` |
| File | `backend/apps/video_analysis/scene/access.py` |
| File | `backend/apps/audit/services.py` |
| File | `backend/apps/behavior/models.py` |
| File | `.env.example` |
| Doc | `specs/015-xai-anomaly-score/plan.md` |
| Doc | `specs/015-xai-anomaly-score/data-model.md` |
| Doc | `specs/015-xai-anomaly-score/contracts/xai-api-ws-contract.md` |
| Doc | `specs/015-xai-anomaly-score/contracts/xai-evidence-contract.md` |

## Scope

This policy covers the following Cycle 015 artifact classes:

| Artifact class | Current surface |
|---|---|
| Review-priority score records | `apps.anomalies` score read endpoints |
| Explanation records | `apps.behavior` explanation list endpoint |
| Deep-XAI request/artifact/evaluation records | `apps.behavior` models and dataclass contracts |
| Future figure bundles and benchmark manifests | `docs/xai_anomaly/benchmark_evidence_layout.md` |

It does not redefine general platform data-retention law outside the feature. It
records how Cycle 015 artifacts must fit inside the existing governed access and
audit surfaces.

## Access policy

### Role gate

Current read access for reviewer-facing XAI surfaces is restricted to:

- authenticated `teacher` or `admin` roles; or
- `superuser` where the view explicitly allows it.

This is already enforced in the current score/explanation read paths:

- `apps.anomalies.views.AnomalyEventViewSet._enforce_preview_role`
- `apps.anomalies.views.JobXAIReviewPriorityScoreListView._enforce_role`
- `apps.behavior.views.BSILReadOnlyListView` (`IsAdminUser`) for the current
  explanation read model

### Job-scoped access grant

Role alone is not sufficient for score reads or preview aggregation. The caller
must also hold an active job-scoped access grant:

- `apps.video_analysis.scene.access.has_access(job_id, user_id)`

Current guarded read paths:

- session review-priority preview in `apps.anomalies.views.AnomalyEventViewSet.review_priority_aggregate_preview`
- persisted XAI score reads in
  `apps.anomalies.views.JobXAIReviewPriorityScoreListView.get`

### Current gap

`apps.behavior.views.BehaviorExplanationRecordListView` is still admin-only and
does not yet enforce the same job-scoped access-grant contract as the score read
path. That is a later implementation gap, not accepted maturity. Future Cycle
015 work must align explanation/deep-artifact retrieval with the same governed
job-scoped access model before production acceptance.

## Audit policy

Every reviewer-facing read or preview surface must be auditable.

Current enforced audit surface:

- `apps.audit.services.AuditService.log(...)`

Current XAI-related audit events already emitted:

| Surface | Current audit event |
|---|---|
| Review-priority aggregate preview | `review_priority_aggregate_preview` |
| Persisted score list read | `review_priority_scores_read` |

Deep-XAI artifact retrieval, artifact download, and explanation record reads do
not yet emit their own dedicated access-audit events. That is an explicit
remaining gap for later Cycle 015 security work and must not be described as
already implemented.

## Retention policy

### Configuration authority

The feature-level retention and review-access knobs are already reserved in
`.env.example`:

| Env key | Current purpose |
|---|---|
| `XAI_ARTIFACT_RETENTION_DAYS` | Retention window for deep/artifact-backed XAI outputs once later-cycle storage/export surfaces are live |
| `XAI_AUDIT_ACCESS_ENABLED` | Feature-owned toggle documenting that XAI access must stay auditable |
| `XAI_REVIEW_ROLE_ALLOWLIST` | Feature-owned reviewer role allowlist for future dedicated XAI surfaces |

These keys are configuration authority for the feature contract, even where the
later-cycle runtime enforcement is still pending.

### Record-retention rules

Current policy by artifact type:

| Artifact type | Current retention rule |
|---|---|
| Score/explanation/deep-XAI relational rows | PostgreSQL-authoritative, append-only, retained until a later governed sweeper is implemented |
| Raw figure/manifests once produced | Preserve manifest/digest lineage even if large binaries are later expired |
| Placeholder local helper output | Not authoritative evidence; may be cleaned up by helper-specific rollback scripts |

### Non-authoritative placeholder rule

Files emitted by placeholder helper writers in:

- `backend/apps/pipeline/model_lifecycle/explainability.py`
- `backend/apps/pipeline/model_lifecycle/visualizations.py`

must never be treated as authoritative XAI evidence. The current authenticity
gate (`scripts/ci/verify_xai_artifacts.py`) exists to keep those helpers outside
the governed XAI artifact path until they are replaced or retired by later
cycles.

## Privacy / minimization policy

Cycle 015 artifacts may contain classroom-derived identity and behavioral
evidence, so the feature policy is:

- keep deep-XAI artifacts job-scoped and authenticated;
- persist bounded summaries in relational records and keep large arrays/images as
  artifact references rather than embedding them inline;
- keep reviewer feedback operational only and never elevate it into anomaly
  ground truth or a training target; and
- expose only the governed summary/state vocabulary, never accusatory text.

These rules are already reflected in the current spec/data-model contracts and
must remain true for future API/WS and artifact-download surfaces.

## Accepted current state vs. remaining gaps

### Enforced now

- teacher/admin role gate for preview and score-read paths
- job-scoped access grant for preview and score-read paths
- audit logging for preview and score-read paths
- feature-owned retention/access env contract present in `.env.example`
- artifact authenticity verifier prevents placeholder artifact writers from
  becoming governed XAI evidence

### Still open

- explanation record reads must gain the same job-scoped access model as score
  reads
- deep-XAI artifact retrieval/download needs dedicated authenticated endpoints
  and access-audit events
- retention sweeper/export policy for deep-XAI artifacts must be implemented in
  a later cycle
- feature-owned `XAI_REVIEW_ROLE_ALLOWLIST` is documented but not yet wired as a
  dedicated runtime enforcement source

## Operational rule

No Cycle 015 task may be marked production-accepted based on artifact access or
retention behavior unless the current evidence proves:

1. the caller is authenticated;
2. the caller passes the governed role gate;
3. the caller holds the required job-scoped access grant for job-bound records;
4. the access is audited; and
5. any large artifact is referenced through a digest-backed lineage surface, not
   a silent ad-hoc file path.
