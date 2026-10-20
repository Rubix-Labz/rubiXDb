# RubixDB implementation inventory

This inventory is the evidence register for the book `RUBIXDB — Architecture, Engineering & Technical Reference`. It describes repository revision `5e8ab125dabf1b77b82f687b88aaf194cf179e84` (branch `master`, observed 2026-10-20) and was assembled from source, manifests, tests, and project documentation. Source code and executable tests take precedence over design prose.

## Evidence status labels

- **Implemented** — present in the current source and covered by tests or an exercised binary path.
- **Implemented with limits** — present, but bounded or narrower than a general-purpose database feature.
- **Partial / phase-boundary** — present in a defined slice while another part is deliberately deferred.
- **Design intent** — specified in project documents but not established as current runtime behavior.
- **Unsupported / not certified** — explicitly unavailable or excluded by the current project decisions.
- **Unverified** — not claimed because the available evidence was insufficient.

## Workspace and product boundaries

| Area | Evidence | Status | Verified interpretation |
|---|---|---|---|
| Core Rust crate | `Cargo.toml`, `src/lib.rs` | Implemented | `rubixdb` owns the storage, catalog, relational row primitives, persistence, WAL, SSTable, LSM, operations, and compaction modules. |
| SQL crate | `sql/Cargo.toml`, `sql/src/lib.rs` | Implemented with limits | `rubixdb-sql` contains parsing, AST, binding, authorization resolution, planning, execution, functions, types, limits, and COPY/LOAD DATA. It depends on the core crate; the core does not depend on SQL. |
| HTTP API crate | `api/Cargo.toml`, `api/src/routes/mod.rs` | Implemented | Axum service with bearer-key authentication, role checks, SQL, KV, catalog, snapshot, metrics, observability, admin, health, and bulk-load routes. |
| CLI/product binary | `cli/src/main.rs`, `cli/src/host.rs` | Implemented | `rubixdb.exe` combines an HTTP SQL client role with local instance discovery/hosting and GUI/static-frontend integration. |
| Instance manager | `instance/src/lib.rs` and submodules | Implemented | Instance directory, OS lock, manifest, credentials, port binding, handshake, lifecycle, storage/relocation support. |
| Standalone API binary | `api/src/main.rs` / API configuration | Partial / not certified | `rubixdb-api.exe` exists as a separately built service shape and requires `RUBIXDB_DATA_DIR`; the embedded host path is the normal product path. |
| Frontend | `frontend/src`, `cli/src/embedded_frontend.rs` | Implemented | React/Vite console with typed fetch client and embedded/static serving path when hosted by the product binary. |
| Python client | `clients/python/rubixdb`, `pyproject.toml` | Implemented | Sync and asyncio HTTP clients, typed errors, transactions, pooling, retries, circuit breaker, streaming, bulk insert, integrations. Package metadata currently declares a proprietary license. |
| Node client | `clients/node/src`, `package.json` | Implemented | TypeScript ESM/CJS HTTP client built on undici, Express integration, typed parameters/errors, transactions, streaming, bulk insert, lifecycle. Requires Node >=20.18.1. |
| Framework examples | `examples/fastapi`, `examples/express`, `examples/django`, `examples/node`, `examples/python` | Implemented with limits | Runnable examples and tests use the local HTTP API. Django integration intentionally uses raw SQL; Django ORM-over-HTTP is unsupported. |

## Core storage and durability

- `src/wal`: framed WAL with `RBXWALv1` magic, 24-byte segment header, CRC32C records, sequence numbers, PUT/DELETE/checkpoint/group operations, segment rotation, recovery, retention, exclusive writer locking, group commit, and fail-closed corruption handling. Evidence: `src/wal/mod.rs`, `src/wal/format.rs`, `src/wal/recovery.rs`, WAL specifications and tests.
- `src/memtable`: mutable/immutable in-memory state used by the LSM write/read path. Evidence: module tests and `src/lsm/mod.rs`.
- `src/sstable`: RUBIC immutable sorted tables with data blocks, bloom filter, sparse index, footer, temporary-file cleanup, validation, and atomic publication. Evidence: `src/sstable/mod.rs`, `src/sstable/format.rs`, `RUBIC_SSTABLE_FORMAT_SPECIFICATION.md`.
- `src/manifest`: manifest framing/state and edits. Current LSM comments explicitly distinguish the implemented SSTable/WAL relationship from future or phase-bound manifest/compaction integration; the book will not describe a fully unified manifest-driven LSM unless verified.
- `src/lsm`: write batching, group commit, durable WAL boundary, memtable, immutable memtables, SSTable-derived reads, recovery, and bounded flush/pressure behavior. Automatic compaction is configurable and phase-bound; source comments state that the default is false and the trigger is not itself an automatic policy in all paths.
- `src/persist`: filesystem abstraction and test/simulation boundaries.
- `src/catalog`, `src/relational`: catalog metadata, table/row storage, typed relational values, key encoding, indexes, transactions, foreign keys, and table-store access over the engine.

## SQL pipeline

The verified conceptual pipeline is:

`HTTP/CLI SQL text -> parse -> AST -> bind/name resolution/type checks/authorization -> BoundStatement -> plan/validate/optimize -> physical operators -> execution over TableStore/IndexBuilder/Transaction -> typed result or typed error`.

Evidence: `sql/src/parse`, `sql/src/ast`, `sql/src/bind`, `sql/src/bound`, `sql/src/auth.rs`, `sql/src/plan`, `sql/src/exec`, `api/src/routes/sql.rs`, and the SQL test modules. The planner explicitly does not execute plans. The executor contains read operators and write execution modules; each statement class must be verified against the current executor/tests rather than inferred from a planner enum.

The type surface includes relational numeric, boolean, text, blob, date, time, timestamp, decimal, JSON/variant-related values, and exact conversion rules. The supported SQL surface is broad and evolves through phase-specific tests: DDL/DML, joins, aggregates, windows, subqueries/CTEs, set operations, recursive queries, indexes, foreign keys, RETURNING, conflict handling, functions, temporal operations, and phase-specific compatibility features. The book will cite the exact test/module evidence for each group and will label unsupported or phase-bound syntax.

## HTTP route families

Evidence: `api/src/routes/mod.rs`, route modules, API integration tests, and frontend `ApiClient`.

- Public liveness: `/healthz`.
- Authenticated readiness/status/identity: `/readyz`, `/v1/whoami`, `/v1/status`, `/v1/metadata`.
- SQL and bulk loading: `POST /v1/sql`, `POST /v1/copy`.
- KV/range/snapshots: `/v1/kv`, `/v1/range`, `/v1/snapshots` and item routes.
- Catalog: database/schema/table/index/authz endpoints, including REST DDL routes where present.
- Metrics and observability: `/v1/metrics`, `/v1/metrics/system`, system timeseries, compaction metrics/status, sessions, queries, events, and version.
- Operator/admin: `/v1/admin/*` backup, check, purge, shutdown, and related operations as configured.
- Static frontend: hosted only when the embedded product startup supplies a frontend distribution directory.

Authentication is bearer API-key based. Configuration parses named `name:role:key` entries into `reader` and `admin` principals. `GET`/`HEAD` generally require reader access; mutating routes and admin routes require admin access; SQL is gated at the API reader boundary and statement authorization is delegated to the SQL authorization context. API keys are redacted from debug output and security logs.

## Bulk loading

`sql/src/copy` implements CSV and JSON input, server-side import-file validation, field/row/input limits, column resolution, typed conversion, batching, cancellation, and transactional batch sinks. The source documentation states: one batch and one record are held in memory; the first failing batch stops the load; earlier batches remain committed. The current feature must not be described as PostgreSQL `COPY TO`, cloud staging, arbitrary client streaming, or an all-or-nothing transaction unless a specific route/test proves it.

## Clients and framework integrations

- Python package: `rubixdb.Client` and `rubixdb.asyncio.AsyncClient`, environment configuration, typed placeholders, transaction context managers, error taxonomy, retries only for safe/rejected operations, breaker, timeouts, metrics/log hooks, stream and bulk helpers.
- Node package: `@rubixdb/client`, typed parameter conversion, ESM/CJS exports, `Client.transaction`, typed errors, retry/breaker/timeouts, streams and bulk inserts; `@rubixdb/client/express` provides attach, transaction middleware, error handling, NDJSON, probes, and graceful shutdown.
- FastAPI: `rubixdb.integrations.fastapi`, lifespan/client dependencies, transaction dependency, typed error mapping, NDJSON, health/readiness.
- Django: `rubixdb.integrations.django`, process client, transactional view helper, middleware, NDJSON and probes; ORM is explicitly unsupported on this HTTP path.

## Frontend evidence

`frontend/src/api/client.ts` is the typed fetch boundary. It adds `Authorization: Bearer <session key>`, sends JSON, maps 401 to session handling, and exposes SQL, catalog, status, metrics, observability, KV, snapshots, and administrative calls. `frontend/src/api/types.ts` mirrors the wire shapes. The Data Integration section is in `frontend/src/integration` and has Import files, Connect from code, API keys, and Connection status tabs. The UI honestly reports unavailable key-management routes because the current server does not expose create/reveal/rotate/revoke APIs. Focused Data Integration tests previously passed 47/47; the frontend typecheck and production build passed during the book inventory session.

## Security and operations

Verified controls include bearer authentication, role mapping, API-key redaction, bounded request/body/SQL/session/rate limits, explicit CORS allow-listing without wildcarding, security headers, no-store API responses, route-pattern-based audit events, bounded observability registries, path/import validation, instance OS locking, and fail-closed storage corruption handling. Known limitations include one static configured key model per configured entry set rather than a complete key-management lifecycle, standalone API certification status, local/loopback deployment assumptions, and platform-specific durability/testing gaps recorded in `OPEN_ITEMS.md`.

## Toolchain and publication state

Observed tools: Rust/Cargo 1.98.1, Node 24.21.0, npm 11.19.0, Python 3.14.7. Pandoc, Typst, TeX/Tectonic, Graphviz, Mermaid CLI, Chromium, and wkhtmltopdf were not available in PATH. The book build therefore includes a pure-Python/stdlib HTML generator and a self-contained PDF fallback where feasible; generated-format limitations will be recorded rather than hidden.

No root `LICENSE`, `COPYING`, or `NOTICE` file was found in the initial root inventory. Existing package metadata uses `Proprietary`/`UNLICENSED` for the client packages. The book will not invent a software license and will distinguish repository licensing evidence from the book output's copyright statement.

## Unverified or deliberately omitted claims

- No external distributed-cluster, replication, sharding, PostgreSQL wire protocol, MySQL wire protocol, or Django DB-API backend claim.
- No claim that every design specification is current runtime behavior.
- No new benchmark figures beyond existing project measurements that identify workload and environment; methodology is provided when evidence is insufficient.
- No claim that GitHub workflows or publishing infrastructure have run successfully unless repository evidence demonstrates it.
- No claim that missing key-management routes, rotation grace, or reveal/revoke audit APIs exist.
