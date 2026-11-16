---
title: 'GYRUS-203: PostgreSQL Enterprise Index & Storage Driver'
id: spec-203-postgres-storage-index-driver
type: specification
format: markdown
category: technical
version: 3
status: completed
last_updated: 2026-11-16T06:31:52Z
last_modified_by: ""
tenant: root
tags:
    - phase-2.1
    - storage-driver
    - postgres
---

# PostgreSQL storage contract

The PostgreSQL profile uses the storage, index, graph, and FTS adapters together. Hybrid file-backed/database stacks are rejected before construction. A configured connection string opens the existing pgx-backed pools and initializes required tables/indexes.

Storage implements `ConceptStore`, `SchemaStore`, and `RestoreConcept` for 2PC rollback recovery. Core tables include `concepts` (JSONB frontmatter and content), `concept_edges`, `concepts_history`, and `schemas`. When PostgreSQL 2PC is active (`lock.provider: postgres`, the default for the `postgres` stack), `concept_locks` and `prepared_transactions` tables coordinate leased write locks (`30s` default TTL), bounded contention retry returning `*gyrus.ConflictError` (CLI exit code `4`), and lazy rollback recovery of expired `PREPARED` transactions. Qualified names identify concepts and relationships. Positive expected versions are checked under row locks; stale writes return typed conflicts and roll back without history writes.

The metadata index and graph use PostgreSQL; search uses [native FTS](spec-204-postgres-fts-search-driver.md). Runtime Git branch selection does not apply to this stack. Redis cache/2PC coordination (`cache.provider: redis`, `lock.provider: redis`) may also be paired with PostgreSQL; asynchronous ingestion queues and hosted deployment remain future roadmap work.

Provider mocks cover CRUD, version conflicts, schemas, relationships, queries, and PostgreSQL/Redis 2PC lock contention and crash recovery. `TestIntegration_PostgresLive` and `TestIntegration_PostgresLive2PC` are opt-in through `GYRUS_TEST_POSTGRES_URL` pointing to an isolated database (`GYRUS_TEST_REDIS_URL` enables live Redis validation). See [configuration examples](tech-ref-002-config-schema.md).
