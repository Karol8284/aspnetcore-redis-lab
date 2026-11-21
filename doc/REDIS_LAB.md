# Redis Lab

## Status

**Gate 0: blocked by missing concurrency evidence. Gate 1 is not started.**

Decision reviewed: **2026-11-21**.

Redis is not part of the application, Compose topology, or V8 starter. This is
intentional: the existing V6 measurements do not yet prove that a Redis dependency
would provide a user-visible or database-load benefit.

This document is the canonical plan and decision record for the Redis Lab. Work
must proceed one gate at a time. A later gate cannot be treated as complete
before the preceding gate has an evidence report and a reproducible test.

## Gate 0 — pilot selection

### Selected read path

The single candidate is:

```text
GET /api/projects/{projectId}/dashboard
```

Actor: an authenticated project participant.

Expected result: task metrics, overdue and upcoming tasks, and recent project
activity visible to that participant.

Data owner:

- `ProjectTasks` owns the task snapshot;
- `Projects` owns access checks and recent activity;
- PostgreSQL remains the source of truth.

The application read is represented by `GetProjectDashboardQuery`. The
Infrastructure handler currently obtains data through
`IGetProjectDashboardStore` and `IProjectTaskDashboardReader`. Any future cache
must enter through a small Application port; controllers and Domain must not
call Redis.

The dashboard is a better first candidate than authentication, balances,
refresh tokens, or session state. Those sensitive and transactional paths are
explicitly excluded. A project dashboard is non-transactional and can tolerate a
short, documented delay if authorization is still checked for every request.

### Existing baseline

The existing 10,000-task fixture and opt-in test provide a repeatable
PostgreSQL-backed baseline. On the post-migration repeat:

| Scenario | Success | p50 | p95 | p99 |
|---|---:|---:|---:|---:|
| `project.dashboard` | 30/30 | 26.203 ms | 29.267 ms | 30.929 ms |

The dashboard uses five PostgreSQL commands per request:

1. project access;
2. task statistics;
3. overdue tasks;
4. upcoming tasks;
5. recent activity.

The pre-migration detailed timing recorded 25.275 ms EF p95 and 35.870 ms HTTP
p95. This makes a database round-trip reduction a plausible hypothesis, but it
does not prove a production bottleneck. The evidence and comparison limitations
are recorded in:

- [`V6_LARGE_FIXTURE_API_BASELINE.md`](V6_LARGE_FIXTURE_API_BASELINE.md);
- [`V6_QUERY_PLAN_FINDINGS.md`](V6_QUERY_PLAN_FINDINGS.md);
- [`V6_EF_SQL_TIMING.md`](V6_EF_SQL_TIMING.md).

The existing report is a sequential test with concurrency one. It is not a load
test and must not be used to claim capacity, saturation, or Redis benefit.

### Hypothesis and cheapest falsification test

Hypothesis: under the same authenticated 10,000-task fixture, a warm dashboard
cache lowers PostgreSQL work and concurrent-read p95 without increasing errors
or exposing another user's data.

The cheapest test that can falsify the hypothesis is a controlled k6 comparison
of the dashboard with the required ramp from 10 to 100 virtual users followed by
the same steady state and duration for every scenario:

1. run PostgreSQL-only baseline;
2. run the Redis-enabled warm path;
3. compare p50/p95/p99, throughput, error rate, PostgreSQL commands, and
   PostgreSQL query time;
4. reject Redis unless p95 and PostgreSQL work improve enough to justify a new
   failure domain, and the error rate does not regress.

No exact acceptance threshold should be invented. The issue's required
benchmark dimensions are still missing:

- representative concurrent usage and steady-state duration;
- host and Docker resource limits for the API, PostgreSQL, and Redis;
- a declared minimum p95/throughput or database-load improvement;
- the dashboard update frequency in the target workload;
- the application version and repeat command for the concurrent baseline.

Until those dimensions are supplied and a concurrent baseline is recorded,
Gate 0 cannot pass and Redis must not be added.

### Proposed cache contract

If Gate 0 passes, keep one cache entry per project:

```text
redis-lab:project-dashboard:v1:{projectId}
```

The key contains only a namespace, version, entity type, and opaque identifier.
It must not contain user identity, titles, authorization claims, or secrets.
Every request must perform the existing project-access check before reading the
shared dashboard entry. The access check failure continues to return the
current not-found response.

Proposed controls:

- TTL: 30 seconds as a starting experiment, not a production commitment;
- jitter: 10% positive randomization, producing 30–33 seconds;
- invalidation: after a committed task create/update/delete/status change and
  after a committed project- or activity-affecting mutation;
- fallback: Redis timeout or connection failure reads PostgreSQL and must not
  change the public API response;
- disabled mode: a configuration flag bypasses Redis completely;
- stale-data limit: normally zero delay after successful invalidation, bounded
  by the jittered TTL if an invalidation request is lost.

Successful database writes must remain the invalidation transaction boundary's
entry point. Cache population must never write business state and PostgreSQL
must remain the sole source of truth.

These values are a testable proposal only. They are not approved until the
concurrent baseline and write workload demonstrate that the invalidation scope
is complete and the stale-data risk is acceptable.

### Exclusions

The first pilot does not include:

- Redis Pub/Sub or Streams;
- distributed locks;
- Redis sessions, refresh tokens, or authorization data;
- replacing PostgreSQL;
- caching multiple endpoints;
- Kafka, brokers, or separate services;
- multi-instance scaling assumptions.

## Gate decision record

| Gate | Decision | Evidence / remaining blocker |
|---|---|---|
| Gate 0 | **Blocked** | A candidate and sequential baseline exist; concurrent fixture, load profile, threshold, and stale-data tolerance still need evidence. |
| Gate 1 | Not started | Gate 0 has not passed. No Redis package, service, cache port, or runtime registration may be added yet. |
| Gate 2 | Not started | Requires the Gate 1 pilot and identical load fixtures. |
| Gate 3 | Not started | Requires a real Redis integration to observe and operate. |
| Gate 4 | Not started | No Redis pattern may enter the V8 starter before Gates 1–3 pass. |

## Rollback and removal

While Redis is not implemented, rollback is the no-op state: keep PostgreSQL
reads and the existing Compose/runtime configuration unchanged.

If a future Gate 1 pilot is accepted, rollback must be a feature-flag/config
change that disables cache reads and writes. The API contract, PostgreSQL reads,
and local Compose startup must continue to work without requiring manual code
rewrites. Removal then consists of deleting the Redis infrastructure adapter,
cache port usage, Redis tests/configuration, and the Compose service only after
the Redis-disabled configuration has been verified.
