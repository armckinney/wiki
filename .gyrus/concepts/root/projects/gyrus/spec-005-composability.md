---
title: Provider Composability, Test Profiles & Capability Matrix
id: spec-005-composability
type: specification
format: markdown
category: technical
version: 8
status: active
last_updated: 2026-11-16T06:31:49Z
last_modified_by: ""
tenant: root
---

# Supported provider stacks

Gyrus supports three complete storage stacks plus orthogonal cache and lock/2PC providers, validated before resource construction:

| Profile | Storage | Index | Graph | Search | Default Lock / 2PC |
| --- | --- | --- | --- | --- | --- |
| local | localfs | okf | okf | okf | none |
| git | git | okf | okf | okf | none |
| postgres | postgres | postgres | postgres | postgres_fts | postgres |

Storage defaults to localfs. Omitted secondary providers follow storage. Hybrid storage/index/graph/search combinations and cloud/blob providers are unsupported; removed names have no aliases.

## Cache and lock/2PC composability

- `cache.provider`: `none` (default), `sqlite` (defaulting to `~/.gyrus/cache.db`), or `redis` may be enabled with any of the three supported storage stacks (`localfs`, `git`, or `postgres`).
- `lock.provider`:
  - `localfs` accepts `none` only (`localfs` relies on local cooperating-process `flock` and atomic file replacement; distributed 2PC is not enabled).
  - `git` defaults to `none` and optionally accepts `redis` for distributed Redis write locking and 2PC.
  - `postgres` defaults to `postgres` (PostgreSQL `concept_locks` / `prepared_transactions` 2PC) and also accepts `redis` or `none`.
- Incompatible combinations (`lock=postgres` on `localfs`/`git`, or `lock=redis` on `localfs`) and non-positive `cache.ttl`/`lock.ttl` durations fail configuration validation before opening resources.

The application constructs one storage instance and injects it into the stack. OKF adapters scan canonical local/Git concepts and source-owned links. PostgreSQL uses database-backed storage/index/graph and native FTS. Both expose qualified names, explicit project/reference scope, and consistent query limits. Existing-concept arguments require `tenant/reference/id` or `tenant/projects/project/id`.

Git sessions select `direct` or `branching`. Branching isolates all knowledge operations on a runtime initiative branch; database-backed hybrid indexes are excluded to prevent branch leakage. See the [Git specification](spec-201-git-storage-driver.md) and [standard configurations](tech-ref-002-config-schema.md).
