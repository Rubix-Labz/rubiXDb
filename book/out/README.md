<div align="center">

<img src="assets/rubixdb-logo.png" alt="RubixDB logo" width="720"/>

# RubixDB

### A Rust-based analytical database with Snowflake-inspired SQL

**Explore data. Run SQL. Build with confidence.**

[![Core](https://img.shields.io/badge/Core-Rust-000000?style=flat-square&logo=rust)](#features)
[![SQL](https://img.shields.io/badge/SQL-Snowflake--inspired-4B6F8A?style=flat-square)](#sql-capabilities)

</div>

> **Before publishing:** Replace `WINDOWS_DOWNLOAD_URL` with a real, verified release URL. Do not present a placeholder as a working download.

---

## Contents

- [What is RubixDB?](#what-is-rubixdb)
- [Features](#features)
- [SQL capabilities](#sql-capabilities)
- [Semi-structured data and regex](#semi-structured-data-and-regex)
- [Interfaces](#interfaces)
- [Download for Windows](#download-for-windows)
- [Build and validate](#build-and-validate)
- [Current status and limitations](#current-status-and-limitations)
- [Project documentation](#project-documentation)
- [Reporting issues](#reporting-issues)
- [Security](#security)
- [License](#license)

## What is RubixDB?

**RubixDB is a Rust-based, single-node analytical database project.** It provides a SQL engine with support for common analytical queries and a growing set of Snowflake-inspired SQL features.

The project brings together a database engine and developer-facing tools, including a command-line interface (CLI), an HTTP SQL API, and a browser-based SQL IDE.

RubixDB aims to provide useful SQL compatibility, predictable behavior, and explicit limits when an operation cannot safely or reliably complete. It is **not fully Snowflake-compatible**, and its performance, scalability, and durability should not be assumed to match a distributed cloud data warehouse.

## Features

- **SQL query engine:** execute familiar SQL statements.
- **Analytical queries:** filtering, sorting, grouping, aggregates, joins, CTEs, subqueries, set operations, and window functions.
- **Data modification:** supported forms of `INSERT`, `UPDATE`, `DELETE`, and `MERGE`.
- **Schema management:** supported schema and table DDL operations.
- **Snowflake-inspired functions:** date/time, string, mathematical, conditional, and aggregate functions.
- **Semi-structured data foundations:** basic `VARIANT` and `ARRAY` types, JSON functions, and path access.
- **Bounded regular expressions:** resource limits help prevent excessive regex processing.
- **CLI:** interact with the system from a terminal.
- **HTTP SQL API:** submit SQL requests over HTTP.
- **Browser SQL IDE:** write and execute SQL in a browser.
- **Automated tests:** coverage for SQL behavior, API and CLI flows, and compatibility cases.

Feature support varies by statement and expression. Check the compatibility documentation before depending on a particular syntax or edge case.

## SQL capabilities

The following is a high-level summary, not a promise that every clause combination or Snowflake edge case is supported.

### Queries and analytics

- `SELECT`, `WHERE`, `ORDER BY`, `LIMIT`, and `DISTINCT`
- Aggregates including `COUNT`, `SUM`, `AVG`, `MIN`, and `MAX`
- `GROUP BY`, `HAVING`, and supported grouping extensions
- Common table expressions (CTEs), including recursive CTEs
- Scalar and supported correlated subqueries
- Set operations such as `UNION`, `INTERSECT`, and `EXCEPT`
- Window functions such as `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `LAG`, and `LEAD`
- Supported window frames and `QUALIFY`
- Supported join forms, including inner, left, right, full, cross, and natural joins

Execution strategies and resource limits depend on the query. Do not assume all operators can spill to disk or that the engine has comprehensive cost-based query optimization.

### DML and DDL

Supported forms include:

- `INSERT`, multi-row `INSERT`, and `INSERT ... SELECT`
- `UPDATE` and `DELETE`
- Supported `RETURNING` forms
- `ON CONFLICT DO NOTHING` and `ON CONFLICT DO UPDATE`
- `MERGE INTO` with supported matched and not-matched actions
- `UPDATE ... FROM` and `DELETE ... USING`
- `CREATE SCHEMA`, supported table-alter operations, table rename, and `DROP SCHEMA`
- Supported constraints and transaction statements

Exact syntax and restrictions matter. Consult the compatibility matrix rather than assuming all possible combinations are accepted.

### Transactions and diagnostics

- `BEGIN`, `COMMIT`, and `ROLLBACK`
- Snapshot isolation and read-your-own-writes behavior
- `EXPLAIN` for supported query plans
- SQL access through the CLI and HTTP API

Known transaction limitations include no savepoints, limited DDL transaction behavior, and possible write-skew scenarios. Review the engineering documentation before relying on specific isolation guarantees.

## Semi-structured data and regex

Recent development added foundational support for:

- `VARIANT` and `ARRAY` column types stored as canonical JSON text
- `PARSE_JSON`, `TRY_PARSE_JSON`, and `TO_JSON`
- `GET_PATH`, `ARRAY_SIZE`, and `TYPEOF`
- Path expressions such as `v:a.b[0]`
- Date/time functions: `DATE_TRUNC`, `LAST_DAY`, `ADD_MONTHS`, `NEXT_DAY`, and `TO_CHAR`
- Math/string functions: `POWER`, `SQRT`, `TRUNC`, `LEFT`, and `RIGHT`

Regular-expression processing has explicit bounds for pattern size, subject size, compiled program size, and materialized replacement output.

**These features are not complete Snowflake semi-structured-data compatibility.** `OBJECT`, `ARRAY_CONSTRUCT`, some `VARIANT` behaviors such as ordering, and other functions remain incomplete. Frontend display support for the new types also requires verification.

## Interfaces

| Interface | Purpose |
|---|---|
| CLI | Run database-related commands from a terminal |
| HTTP SQL API | Submit SQL requests over HTTP |
| Browser SQL IDE | Write and execute SQL in a browser |
| Windows distribution | Intended download-and-launch experience; verify the package and release link before advertising availability |

Document exact commands, API endpoints, configuration options, and supported operating systems alongside the relevant source code or release package.

## Download for Windows

<div align="center">

### Get RubixDB for Windows

<a href="WINDOWS_DOWNLOAD_URL">
  <img src="https://img.shields.io/badge/Download-RubixDB%20for%20Windows-111111?style=for-the-badge&logo=windows&logoColor=white" alt="Download RubixDB for Windows"/>
</a>

</div>

**Before publishing:** replace `WINDOWS_DOWNLOAD_URL` with a verified release page or direct installer URL. If a Windows installer is not available yet, say so clearly and link to the official release page instead.

### Installation

Once an official package is available:

1. Download it from the official RubixDB release page.
2. Verify that the release and package match the version you intend to install.
3. Open the installer or follow the release-specific instructions.
4. Launch RubixDB using the installed application or documented command.
5. Consult the troubleshooting guide if startup fails.

Do not promise a particular installer format, automatic updates, or Start-menu shortcuts unless the actual package provides them.

### System requirements

Publish only requirements that have been tested. Document:

- Supported Windows versions and CPU architecture
- Minimum and recommended memory
- Required disk space
- Runtime or other dependencies
- Whether the package includes the database engine or requires a separate process

## Build and validate

RubixDB is primarily implemented in Rust and uses Cargo for its workspace. Contributors should use the Rust toolchain version specified by the repository and follow the current contributor instructions.

General workspace validation commands used by the project include:

```bash
cargo check --workspace --all-targets --all-features
cargo fmt --all -- --check
cargo clippy --workspace --all-targets --all-features -- -D warnings
```

These commands are validation checks, not a guarantee that every test passes on every machine. Use the repository's current instructions for exact build prerequisites and package-specific test commands.

## Current status and limitations

RubixDB has a growing automated test suite covering SQL behavior, API and CLI behavior, and compatibility cases. Recent development added tests for regex, semi-structured data, date/time functions, and math/string functions.

Read the latest compatibility matrix and engineering reports for the current status. Test results can vary by build mode and machine load. Known workspace issues have included repository-hygiene findings, timing-sensitive performance thresholds, and failures associated with concurrent worktree changes; track these separately from feature-specific failures.

### Important limitations

- **Incomplete Snowflake compatibility:** some features are partial, unimplemented, architecturally blocked, or intentionally not planned.
- **Semi-structured SQL gaps:** `OBJECT`, additional array/object constructors, and some `VARIANT` behavior remain incomplete.
- **Frontend support:** the browser IDE needs verification and/or updates for `VARIANT` and `ARRAY`.
- **Execution memory:** not all sorts, joins, windows, or aggregations have spill-to-disk support. Some operations are bounded and may fail rather than consume unbounded resources.
- **Query optimization:** do not assume comprehensive cost-based join ordering or cardinality estimation.
- **Transactions:** savepoints are not supported; DDL transaction behavior is limited; write skew may be possible.
- **Durability verification:** the recent development report does not certify power-loss behavior.
- **Release readiness:** flaky tests, repository-hygiene findings, platform-specific behavior, and real-browser interactions need continued verification.

This is not a complete list. Consult the current compatibility matrix and open-items tracker before using RubixDB for production workloads.

## Project documentation

Keep the links below aligned with files that actually exist in the repository:

- `PROGRESS.md` — implementation progress and current status
- `CHANGELOG.md` — notable changes by development phase
- `OPEN_ITEMS.md` — known gaps and follow-up work
- Snowflake SQL compatibility document — supported syntax, test status, limitations, and feature matrix
- Architecture documentation — storage, execution, transaction, and recovery design
- Release notes — installation steps, supported platforms, and version-specific issues

Add working relative links to the actual files before publishing. Do not leave documentation labels pointing to files that do not exist.

## Reporting issues

When reporting a problem, include:

1. RubixDB version or commit hash
2. Operating system and version
3. Installation method and relevant configuration
4. The smallest SQL statement or command that reproduces the issue
5. Expected behavior and actual behavior
6. Error output and relevant logs
7. Whether the issue reproduces consistently

Add the official issue-tracker or support URL here once it is confirmed.

## Security

Do not commit passwords, API keys, tokens, or other secrets. Avoid including secrets in issue reports. Add a private vulnerability-reporting channel when one has been configured.

## License

Add the actual license name and link to the repository's license file. Until it is confirmed, do not assume that redistribution is permitted.

---

<div align="center">

**RubixDB**

*An evolving analytical database, built with Rust.*

</div>
