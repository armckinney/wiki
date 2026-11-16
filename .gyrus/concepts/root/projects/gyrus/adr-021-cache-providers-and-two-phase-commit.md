---
title: Cache providers and synchronous two-phase commit
id: adr-021-cache-providers-and-two-phase-commit
type: adr
format: ""
category: architecture
version: 1
status: proposed
last_updated: 2026-11-16T06:31:13Z
last_modified_by: ""
tenant: root
tags:
    - phase-2
    - stability
    - cache
    - sqlite
    - redis
    - 2pc
    - locking
---

# Architecture Decision Record: Cache providers and synchronous two-phase commit

## Context

* **Author:** Antigravity, implementing the Phase 2 Stability roadmap (`prd-001`).
* **Date Proposed:** 2026-11-16
* **Deciders:** Repository owner.

### 1. The Problem

File-backed `localfs` and `git` stacks scan authoritative OKF files directly on every query, causing repeated remote clones on Git CLI invocations and repeated Markdown parsing on `localfs`. On shared database and remote repositories, read-heavy agent context loops (`gyrus_get`, `gyrus_suggest_context`) lack TTL-bounded document caching, while single-statement row/version checks do not provide leased write locks, crash-safe multi-step rollback, or atomic multi-source graph mutations. Additionally, the architecture diagram left open whether a hosted MCP server should maintain an in-process transaction queue or rely on storage/Redis coordination.

### 2. Objectives & Constraints

* Keep authoritative knowledge in the selected storage stack (`localfs`, `git`, or `postgres`) and preserve existing behavior (`cache.provider: none`) by default.
* Support selectable `none`, `sqlite`, and `redis` cache providers across stacks, using global `~/.gyrus/cache.db` by default for SQLite so code repositories stay clean and multi-repo workflows can share cached namespaces.
* Preserve immediate read-after-write consistency and Git initiative branch isolation.
* Provide synchronous Two-Phase Commit (2PC) and leased write locks via PostgreSQL (`postgres`, default on the PostgreSQL stack) and Redis (`redis`, available on PostgreSQL and Git stacks), while keeping `localfs` and `git` (without Redis) explicitly free of 2PC (`lock.provider: none`).

---

## Decision

We will implement:

1. **Read-Through & Write-Invalidating Cache Providers (`none`, `sqlite`, `redis`)**:
   * Add `gyrus.CacheProvider` (`pkg/gyrus`) and a `CachedStore` decorator (`internal/provider/cache`) that caches full `Concept` envelopes, schema overrides, and collection snapshots with a configurable TTL (`5m` default) and immediate write-through invalidation on mutations.
   * Default `sqlite` cache storage to global `~/.gyrus/cache.db` (overridable via `cache.path`), verify `(mtime, size)` file fingerprints on `localfs` reads/scans to detect external edits, and key `git` caches dynamically by `git:<repo_hash>:<active_branch>` to guarantee branch isolation.
   * Preserve `ConceptScanner` capability strictly from the wrapped store so PostgreSQL storage does not expose `ConceptScanner`.

2. **Synchronous Two-Phase Commit & Leased Write Locks (`none`, `postgres`, `redis`)**:
   * Resolve the architecture drawing's transaction-queue question in favor of synchronous storage/Redis-backed 2PC (`gyrus.TransactionCoordinator`) rather than an in-process MCP queue, preserving immediate read-after-write consistency across CLI and horizontally scaled MCP processes (leaving asynchronous execution queues to Phase 5 scalability work).
   * Execute mutations via Phase 1 (`Acquire` sorted resource locks + `Prepare` durable before-state snapshot) and Phase 2 (`Commit` or `Rollback` with cache invalidation) across concepts, history, index, graph edges (including multi-source `Engine.UpsertEdges`), and schema overrides.
   * Use a `30s` default lock TTL, bounded retry-with-backoff (`~2s`) on active lock contention returning `*gyrus.ConflictError` (CLI exit code `4`), and lazy rollback recovery of expired `PREPARED` transactions on subsequent lock acquisition.

### Key Rationale

* **Shared Read Acceleration Without Second Truth:** Caching full `Concept` documents and snapshots with TTL + write invalidation avoids secondary query divergence while eliminating redundant Git clones and database round-trips.
* **Read-After-Write Consistency:** Synchronous 2PC guarantees that consecutive agent calls (`gyrus_create` -> `gyrus_link` -> `gyrus_get`) always observe committed state without polling an asynchronous MCP queue.
* **Explicit Stack Boundaries:** Rejecting `lock=postgres` on file/Git stacks and `lock=redis` on `localfs` before opening resources avoids misleading transaction guarantees.

---

## Alternatives Considered

### Alternative 1: In-Process MCP Asynchronous Transaction Queue

* **Pros:** Smooths bursty writes inside one MCP server process.
* **Cons:** Fails to coordinate across CLI invocations or multiple hosted MCP instances and breaks immediate read-after-write semantics for agents.

### Alternative 2: Advisory Locks Without Prepared Before-State Rollback

* **Pros:** Simpler lock-only implementation without `prepared_transactions` state.
* **Cons:** A process crash mid-mutation (e.g., partial multi-source graph batch or uncommitted history/index updates) leaves dirty partial state when the lock TTL expires.

---

## Consequences

* **Positive / Gains:** Significant local `git` CLI and hosted MCP read latency reduction when `sqlite` or `redis` caching is enabled; deterministic deadlock-free locking, crash recovery, and atomic multi-source edge batches on PostgreSQL and Redis-coordinated stacks.
* **Negative / Trade-offs:** Enabling 2PC adds lock-table/Redis round-trips on writes; `sqlite` caches trade disk space in `~/.gyrus/cache.db` (bounded by TTL) for faster local queries.
* **Risks:** Clock skew or long-running mutations exceeding `lock.ttl` (`30s` default) could allow stale lock reclamation; operators can tune `lock.ttl` per workload.

---

## Compliance & Verification

* **Validation:** Automated unit, race-detector, and integration suites verify config defaults (`~/.gyrus/cache.db`), stack compatibility rejection, `sqlite`/`redis` TTL and `localfs` `(mtime, size)` invalidation, `git` branch isolation, `*gyrus.ConflictError` (exit code `4`) on lock contention, lazy rollback recovery after lock expiration, and atomic `Engine.UpsertEdges` rollback (`go test -race ./...`, `make test`, `make lint`). Live Redis and PostgreSQL tests are gated via `GYRUS_TEST_REDIS_URL` and `GYRUS_TEST_POSTGRES_URL`.
