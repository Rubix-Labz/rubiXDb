# rubiXDb User Manual

**Version described:** rubiXDb 0.1.0 (the build in this repository on 2026-10-20) · **Platform verified:** Windows 10 · **Companion document:** `docs/RUNBOOK.md` (operating, backing up, recovering, troubleshooting)

> **Status, stated plainly.** rubiXDb is a local, single-computer relational database. Its storage engine passed its own certifications, but **the whole product is not declared production ready** (`docs/PROJECT_STATE.md`). Real power loss and a real full disk have not been tested. Keep backups on another drive (section 9 of the runbook). Where this manual says "measured", it means measured on this build on 2026-10-20; the SQL surface is growing quickly, so check `docs/SQL_SUGGESTIONS_AND_PORT_PROMPT.md` and the editor's suggestions for the newest words.

---

## 1. What rubiXDb is

* A relational database that runs **on your own computer** as one program, `rubixdb`. It listens only on the loopback address (`127.0.0.1`), so nothing on the network can reach it.
* You use it three ways: the **web console** in your browser (`rubixdb gui`), the **SQL prompt** in a terminal (`rubixdb cli`), or **HTTP** from your own programs (`POST /v1/sql`).
* SQL dialect: mostly Snowflake-style (`QUALIFY`, `IFF`, `NVL`, `LISTAGG`, `VARIANT`), with PostgreSQL-style `ON CONFLICT`, `RETURNING`, `ILIKE` and `::` casts.
* One instance holds one database named `default` with schemas inside it (the first schema is `public`). A computer can have several instances.

**What it is not:** a server for other computers, a cluster, a replicated or partitioned system. There is no user/password login: access is by an API key kept in a file only you can read (section 8).

---

## 2. Starting rubiXDb

### 2.1 Double-click (Windows)

Double-click `rubixdb.exe`. A console window opens, an animated rubiXDb logo plays (any key skips it), and a menu asks:

```
How do you want to open rubiXDb?
  [1] GUI   the web console, in your browser (on this computer only)
  [2] CLI   a SQL prompt in this window
  [Q] quit
```

* **1, G or Enter:** starts the database and opens the console in your browser. The window stays open while it runs. **Closing the window stops the database cleanly.**
* **2 or C:** a SQL prompt in the same window. Closing the window also stops the database cleanly.
* **Q or Esc:** quit.

If the window is narrower than about 70 columns, or `NO_COLOR` is set, you get the logo without the animation. If something fails, the window waits for Enter so you can read the message. The menu appears only for a double-click; typing `rubixdb` in a terminal opens the SQL prompt as before. `rubixdb menu` shows the menu on request; `RUBIXDB_LAUNCHER=never` turns the automatic menu off, `=always` forces it.

### 2.2 From a terminal

| Command | What it does |
|---|---|
| `rubixdb gui` | find or start the `default` instance and open the console in your browser |
| `rubixdb gui --no-browser` | start (or attach) but do not open a browser; prints the address |
| `rubixdb gui --instance NAME` | use (or create) another instance |
| `rubixdb` or `rubixdb cli` | the interactive SQL prompt (starts an instance if none is running) |
| `rubixdb -c "SQL"` | run one statement, or `;`-separated statements, and exit |
| `rubixdb -f script.sql` | run every statement in a file and exit |
| `rubixdb menu` | the animated start menu |
| `rubixdb --version`, `--help` | version, help |

Every command finds the running instance, or starts one if none exists. Only one process can own an instance; a second `rubixdb gui` on the same name attaches to the first.

### 2.3 Ports and the "port is busy" question

The `default` instance uses port **302**. If that port is taken:

* by **another rubiXDb instance**, in a terminal you are asked: `[1]` use the next free port (303, 304 …) and keep the other running (default), or `[2]` stop the other instance gracefully and use the port. Only an explicit `2` stops anything.
* by **another program**, it is never touched; the next free port is used.
* when not in a terminal, it never asks: it takes the next free port.

`RUBIXDB_PORT_CONFLICT=ask|next|stop` answers in advance. The console address is printed when it starts (`rubiXDb instance "default" ready at http://127.0.0.1:302`).

### 2.4 Where your data lives

| Platform | Folder |
|---|---|
| Windows | `%LOCALAPPDATA%\rubiXDb\instances\<name>\` |
| Linux | `$XDG_DATA_HOME/rubiXDb/instances/<name>/` or `~/.local/share/rubiXDb/instances/<name>/` |
| macOS | `~/Library/Application Support/rubiXDb/instances/<name>/` |

`RUBIXDB_INSTANCES_ROOT` moves all instances. Inside an instance folder: `instance.json` (identity and port), `credentials.json` (the API key), `data/` (the database), `backups/`, `security.log`, `disk-history.json`, `instance.lock`. Instance names are 1–64 letters, digits, `-` or `_`. See `rubixdb instance list` and `rubixdb instance status [NAME]` (they show the data folder and the free space on its drive).

---

## 3. The web console

Open it with `rubixdb gui`. The left sidebar has these pages:

| Page | What it is for |
|---|---|
| **Home** | the instance at a glance: storage state, live SSTables, checkpoint, uptime, compaction, read/write activity; quick links (new SQL query, snapshots, backups) |
| **SQL Console** | worksheets of SQL and Markdown cells (section 3.1) |
| **Data integration** | load a CSV or JSON file into a table (section 3.2) |
| **Monitoring** (Health / Storage) | engine status, read and write metrics |
| **Catalog** (Data Explorer) | browse schemas, tables and indexes; key lookup and range query; schema design (ER diagram) |
| **Governance & security** (Settings) | connection and access information |
| **Compute** (Compaction) | automatic trigger, threshold, live SSTable count, cycles, most recent cycle |
| **Admin** (Operations) | instance, storage, write-ahead log, queries, resources; **backups, integrity check, maintenance** (see the runbook) |
| **Snapshots** | consistent read-only views of the data at a moment; release them when finished |

The profile menu at the bottom left has **Settings** (dark/light, the data directory), a one-time rename of the profile name, and **Create another instance** (with a confirmation). The **RAM card** shows memory, processor and thread use (hover or click for the detail panel) and now also **disk space** on the drive that holds the database. A banner appears above the page when that drive is running low (section 7).

### 3.1 The SQL Console

* **Worksheets** are listed in the left rail (open by default; the arrow button beside the path, or `Ctrl+B`, collapses it). Each worksheet is a list of **cells**: SQL cells and Markdown cells. They are kept in your browser session and autosaved.
* **Run** a statement with `Ctrl+Enter`. Several statements in a cell run one at a time. Results appear under the cell as a table, typed (numbers, text, dates, `DECIMAL`, `VARIANT` …). `Ctrl+Alt+E` downloads the focused cell's rows as CSV.
* **Transactions:** `BEGIN` … `COMMIT` / `ROLLBACK` across cells share one session.
* **Suggestions** appear as you type: tables and columns (click a table in the left tree to see its columns and types), keywords, **functions with their signature**, and column types where a type is expected. `Ctrl+Space` asks for them; `Tab` accepts. Keywords are turned to UPPERCASE as you finish them (never inside strings, comments, quoted names, or names the catalog knows). Only words the server really accepts are offered.
* **Errors** explain what went wrong and often what to do ("Did you mean SELECT?"); a failure shows a toast that leaves by itself.
* **History** and sessions are in the right panel (collapsed to an icon rail until you open it with `Ctrl+J`).

| Shortcut | Action |
|---|---|
| `Ctrl+Enter` | run the statement at the cursor |
| `Ctrl+Shift+Enter` | run all cells |
| `Ctrl+Alt+Enter` | run every cell above the focused one |
| `Ctrl+.` | stop running cells |
| `Ctrl+Alt+R` | clear all outputs |
| `Ctrl+Alt+N` or `Shift+Enter` | add a SQL cell below |
| `Ctrl+/` | toggle Markdown cell (in an editor: comment the line) |
| `Ctrl+Shift+C` (or `Ctrl+T`) | new worksheet |
| `Ctrl+Shift+D` | duplicate the worksheet |
| `Ctrl+P` | open a worksheet by name |
| `Ctrl+K` or `Ctrl+Shift+P` | command palette |
| `Ctrl+B` / `Ctrl+J` | toggle left rail / right panel |
| `Ctrl+Alt+D` | light / dark |
| `?` | list all shortcuts (when not typing) |

### 3.2 Data integration (CSV and JSON)

1. Open **Data integration**, drop a `.csv`, `.tsv`, `.txt`, `.json` or `.ndjson` file (UTF-8, UTF-16 and common Excel encodings; a byte-order mark and Excel's `sep=` line are handled).
2. Check the **preview**. Delimiter, header row and quoting are detected and can be changed.
3. Choose where it goes: a **schema**, then **a new table** (give a lowercase name; column names and **types are picked for you**: integer, bigint, `DECIMAL(p,s)`, double, boolean, date, time, timestamp, text) or **an existing table** (map file columns to table columns).
4. Choose what to do with duplicate keys (stop, skip, or update), review the **problems list**, and import. A new table gets a primary key (a detected unique column, or a generated `row_id`).
5. Small files load in **one transaction** (all rows or none); larger ones in confirmed batches of up to 1,000 rows with progress and **Cancel and undo**. A table that was created for a failed import and holds nothing is removed again.

Limits that come from the engine: about 9,990 rows per transaction, up to 1,600 columns (only 512 may be `NOT NULL`), one row at most 1 MiB. The details and every edge case are in `docs/FRONTEND_DATA_INTEGRATION.md`.

---

## 4. The SQL prompt (terminal)

```
rubixdb> SELECT 1 + 1 AS two;
```

* A statement ends with `;` (one line is also accepted as one statement). `BEGIN`/`COMMIT`/`ROLLBACK` span lines.
* Meta-commands: `\l` databases, `\ls` schemas, `\lt` tables, `\d TABLE` describe, `\di` indexes, `\du` your authorization, `\conninfo` connection, `\c` reconnect, `\help`, `\q` quit.
* Output is made safe for the terminal: control characters in stored data are escaped.
* `rubixdb -c "…"` and `-f file.sql` run non-interactively and exit with a non-zero code on error.
* You never type the API key: the prompt reads it from the instance folder. To reach another server instead, set `RUBIXDB_API_URL` and `RUBIXDB_API_KEY` in the environment (never as arguments).

---

## 5. SQL reference

*(Measured by sending statements to the real server on 2026-10-20. "Accepted" means the server ran it. Anything not listed here is probably not supported yet; unsupported statements are refused with a clear `UNSUPPORTED` message and change nothing.)*

### 5.1 Reading data

* `SELECT [DISTINCT]`, aliases, `*`, expressions, `FROM`, `WHERE`, `GROUP BY` (including `ROLLUP`, `CUBE`, `GROUPING SETS`, `GROUP BY ALL`, `GROUPING()`), `HAVING`, `ORDER BY` (name, alias or position; `ASC`/`DESC`; `NULLS FIRST/LAST`), `LIMIT … OFFSET`, `FETCH FIRST n ROWS ONLY`, `QUALIFY`, named `WINDOW`, `TABLESAMPLE`, `VALUES (…)` as a table, `EXPLAIN`.
* **Joins:** `INNER`, `LEFT`, `RIGHT`, `FULL [OUTER]`, `CROSS`, `NATURAL` (also left/right/full), `USING (…)`, `LATERAL` (also `CROSS/LEFT JOIN LATERAL`).
* **Subqueries and CTEs:** scalar, `IN`, `EXISTS`, `NOT EXISTS`, correlated, `ANY`/`ALL`/`SOME`, derived tables, `WITH`, `WITH RECURSIVE` (depth limit 1,000).
* **Set operations:** `UNION [ALL] [BY NAME]`, `INTERSECT [ALL]`, `EXCEPT`/`MINUS [ALL]`.
* **Predicates and operators:** `=`, `<>`, `<`, `BETWEEN`, `IN`, `LIKE`, `ILIKE`, `RLIKE`/`REGEXP`, `IS [NOT] NULL`, `AND OR NOT`, `||`, `+ - * / %`, `CASE`, `CAST(x AS t)`, `TRY_CAST`, `x::t`.

### 5.2 Window functions and aggregates

* **Window:** `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `PERCENT_RANK`, `CUME_DIST`, `NTILE`, `LAG`, `LEAD`, `FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE`, and aggregates with `OVER (PARTITION BY … ORDER BY … ROWS|RANGE|GROUPS BETWEEN …)`, `IGNORE|RESPECT NULLS`.
* **Aggregates:** `COUNT` (also `COUNT(DISTINCT a, b)`), `SUM`, `AVG`, `MIN`, `MAX`, `MEDIAN`, `STDDEV`/`_POP`/`_SAMP`, `VARIANCE`/`VAR_POP`/`VAR_SAMP`, `LISTAGG(… ) WITHIN GROUP (ORDER BY …)`, `STRING_AGG`, `ARRAY_AGG`, `COUNT_IF`, `ANY_VALUE`, `BOOL_AND`/`BOOL_OR`, `MIN_BY`/`MAX_BY`, `RATIO_TO_REPORT`; `FILTER (WHERE …)`.

### 5.3 Changing data

* `INSERT INTO t [(cols)] VALUES (…), (…)`, `INSERT … SELECT`, `INSERT … DEFAULT VALUES` (when every column has a default), `ON CONFLICT [(cols)] DO NOTHING | DO UPDATE SET … = EXCLUDED.…`, `RETURNING`.
* `UPDATE t SET … [FROM other] [WHERE …] [RETURNING …]`, `DELETE FROM t [USING other] [WHERE …] [RETURNING …]`.
* `MERGE INTO t USING s ON … WHEN MATCHED [AND …] THEN UPDATE SET … | DELETE  WHEN NOT MATCHED [AND …] THEN INSERT (…) VALUES (…)`. (Not supported: `RETURNING` on `MERGE`, `INSERT ROW`, `WHEN NOT MATCHED BY SOURCE`.)
* `TRUNCATE [TABLE] t`.
* Not supported: `INSERT OR REPLACE`, `INSERT OVERWRITE`, MySQL's `REPLACE INTO` / `ON DUPLICATE KEY UPDATE`.

### 5.4 Defining things

* `CREATE TABLE [IF NOT EXISTS] t (col type [NOT NULL] [DEFAULT x] [UNIQUE] [REFERENCES …] [CHECK (…)], …, PRIMARY KEY (…), FOREIGN KEY …)`. **Every table needs a primary key** (key columns are `NOT NULL`). `CREATE TABLE … AS SELECT` is recognised, but the new table must still be given a primary key (the server's message says how).
* `DROP TABLE [IF EXISTS] t`, `CREATE [UNIQUE] INDEX [IF NOT EXISTS] i ON t (cols)`, `DROP INDEX [IF EXISTS] i ON t`.
* `CREATE SCHEMA [IF NOT EXISTS] s`, `DROP SCHEMA [IF EXISTS] s [CASCADE | RESTRICT]`, `USE SCHEMA s`. (One database, `default`; `CREATE DATABASE` is refused.)
* `ALTER TABLE t`: `ADD [COLUMN] [IF NOT EXISTS] …`, `DROP COLUMN`, `RENAME COLUMN a TO b`, `RENAME TO new`, `ALTER COLUMN c SET|DROP NOT NULL`, `SET|DROP DEFAULT`, `ADD CONSTRAINT … CHECK | UNIQUE | FOREIGN KEY`, `ADD FOREIGN KEY | CHECK | UNIQUE`, `DROP CONSTRAINT [IF EXISTS]`.
* Foreign keys: `ON DELETE`/`ON UPDATE` `CASCADE`, `SET NULL`, `SET DEFAULT`, `RESTRICT`, `NO ACTION`; always checked immediately (no `DEFERRABLE`).
* Not supported: views, sequences / identity / auto-increment, temporary tables, `CREATE OR REPLACE`, changing a column's type, adding a primary key later, `COMMENT ON`, triggers, stored procedures, `CREATE USER` / `GRANT`.

### 5.5 Transactions

`BEGIN` (or `START TRANSACTION`), `COMMIT`, `ROLLBACK`. A transaction sees a consistent snapshot; if two transactions change the same row, the second to commit is refused with `CONFLICT_ERROR` and must be retried. **`CREATE TABLE`, `ALTER` and other schema changes inside a transaction take effect at once and are not undone by `ROLLBACK`.** Limits: 50 open transactions (sessions) per principal; a transaction idle for 300 s or open for 1,800 s is closed; a statement may run at most 30 s. No `SAVEPOINT`, no isolation-level setting, no row locks (`SELECT … FOR UPDATE`).

### 5.6 Types

| Type | Notes |
|---|---|
| `INTEGER` (`INT`), `BIGINT` | 32 / 64-bit whole numbers (`SMALLINT`, `TINYINT` are also accepted as of the current build) |
| `REAL`, `DOUBLE` (`FLOAT`) | floating point |
| `DECIMAL(p,s)` / `NUMERIC` | exact, up to 38 digits |
| `TEXT`, `VARCHAR`, `CHAR`, `STRING` | text (no length limit enforced; a row is at most 1 MiB) |
| `BOOLEAN` (`BOOL`) | |
| `DATE`, `TIME`, `TIMESTAMP` (`DATETIME`, `TIMESTAMP_NTZ`) | no time zones; ISO 8601 spellings |
| `BLOB`, `BINARY`, `VARBINARY` | |
| `VARIANT`, `ARRAY`, `OBJECT` | semi-structured JSON values; read with `v:key.sub[0]` or `GET_PATH`; cast with `::` |
| `UUID`, `NUMBER` | accepted by the current build (they were refused by a build made a few hours earlier, so confirm with a `CREATE TABLE` on yours) |
| Not supported | `JSON` as a column type, `INTERVAL`, `TIMESTAMP_TZ`, `TIMESTAMP_LTZ`, geometry |

### 5.7 Built-in functions

About 100 functions. Names are case-insensitive. The editor shows the signature of each as you type.

* **Text:** `UPPER`, `LOWER`, `LENGTH`/`LEN`, `TRIM`/`LTRIM`/`RTRIM`/`BTRIM`, `LPAD`, `RPAD`, `SUBSTR`/`SUBSTRING`, `LEFT`, `RIGHT`, `REPLACE`, `CONCAT`, `CONCAT_WS`, `POSITION`/`CHARINDEX`, `REVERSE`, `INITCAP`, `REPEAT`, `SPLIT_PART`, `SPLIT`, `TRANSLATE`, `ASCII`, `CHR`, `STARTSWITH`, `ENDSWITH`, `CONTAINS`.
* **Pattern matching:** `REGEXP_LIKE` (`RLIKE`), `REGEXP_COUNT`, `REGEXP_REPLACE`, `REGEXP_SUBSTR`, `REGEXP_MATCH`.
* **Numbers:** `ABS`, `ROUND`, `FLOOR`, `CEIL`/`CEILING`, `MOD`, `SQRT`, `POWER`/`POW`, `TRUNC`/`TRUNCATE`, `SIGN`, `LN`, `EXP`, `LOG`, `DIV0`, `TO_NUMBER`, `TRY_TO_NUMBER`, `TO_BOOLEAN`.
* **Dates and times:** `CURRENT_DATE`, `CURRENT_TIME`, `CURRENT_TIMESTAMP`, `NOW`, `SYSDATE`, `GETDATE`, `LOCALTIMESTAMP`, `DATEADD`, `DATEDIFF`, `TIMESTAMPADD`, `TIMESTAMPDIFF`, `DATE_PART`, `EXTRACT`, `DATE_TRUNC`, `LAST_DAY`, `ADD_MONTHS`, `NEXT_DAY`, `DATE_FROM_PARTS`, `YEAR`, `MONTH`, `DAY`, `TO_DATE`, `TO_TIME`, `TO_TIMESTAMP`, `TO_CHAR`/`TO_VARCHAR`, and the `TRY_` forms.
* **Conditional:** `COALESCE`, `NULLIF`, `NVL`/`IFNULL`, `NVL2`, `IFF`, `DECODE`, `GREATEST`, `LEAST`, `ZEROIFNULL`.
* **Semi-structured:** `PARSE_JSON`, `TRY_PARSE_JSON`, `TO_JSON`, `GET_PATH`, `GET`, `ARRAY_SIZE`, `ARRAY_CONSTRUCT`, `ARRAY_CONTAINS`, `OBJECT_CONSTRUCT`, `OBJECT_KEYS`, `IS_NULL_VALUE`, `TYPEOF`, `TO_OBJECT`, and the `FLATTEN` table function (`LATERAL FLATTEN(INPUT => …) f`).
* Not available yet (measured refused): `MD5`/`HASH`, `RANDOM`, `UUID_STRING`, `DAYNAME`/`MONTHNAME`, `MODE`, `CORR`, `PERCENTILE_CONT`, `APPROX_COUNT_DISTINCT`, `CONVERT_TIMEZONE`, `EQUAL_NULL`, `NULLIFZERO`, `TIME_FROM_PARTS`.

### 5.8 Not supported (always answered with `UNSUPPORTED` and nothing is changed)

`CREATE VIEW`, `CREATE SEQUENCE`, `CREATE TEMPORARY TABLE`, `CREATE OR REPLACE`, `ALTER COLUMN … TYPE`, `ADD PRIMARY KEY`, `SHOW …`, `DESCRIBE`, `SET`, `ANALYZE`, `EXPLAIN ANALYZE`, `SAVEPOINT`, `GRANT`/`REVOKE`, `CALL`, `COMMENT ON`, `SELECT TOP`, `DISTINCT ON`, `LIKE ANY`, `LIKE … ESCAPE`, `SIMILAR TO`, `IS [NOT] DISTINCT FROM`, `IS TRUE`, `INTERVAL`, `SELECT * EXCLUDE`, `ORDER BY ALL`, `MATERIALIZED` CTEs, `SEARCH`/`CYCLE`, window `EXCLUDE`, `ASOF JOIN`, `INTERSECT/EXCEPT BY NAME`, `CREATE DATABASE`/`USE DATABASE`, `DEFERRABLE` constraints, `VALIDATE CONSTRAINT`, `NOT VALID`, `AUTOINCREMENT`/identity columns.

### 5.9 Limits you can hit

| Limit | Value |
|---|---|
| Statement size | 1 MiB |
| Identifier length | 255 characters |
| Parameters per statement | 10,000 |
| Columns per table | 1,600 (512 may be `NOT NULL`) |
| Items in `IN (…)`, rows in `VALUES` | 10,000 |
| Statements per script | 1,000 |
| Expression nesting / subquery nesting / subqueries per statement | 128 / 32 / 256 |
| Row size | 1 MiB |
| Rows written in one transaction | about 10,000 (a transaction's write set may be 16 MiB) |
| Rows or bytes held by one sort / aggregate / join | 1,000,000 rows / 256 MiB (the statement **fails** with `RESOURCE_LIMIT`; nothing spills to disk) |
| Statement run time | 30 s |
| Request body | rejected above the server's limit with HTTP 413 |

---

## 6. Using it from your own programs (HTTP)

The server speaks JSON over HTTP on `http://127.0.0.1:<port>`. Every route except `GET /healthz` needs the header `Authorization: Bearer <API key>`. The key is the 64-character value in `credentials.json` (never put it on a command line you share, in a URL, or in a script you commit).

```
POST /v1/sql        {"sql": "SELECT id, name FROM people WHERE id = $1",
                     "params": [{"type":"integer","value":1}], "session_id": null}
```

Verified replies:

```
{"session_id":null,"result":{"kind":"rows","columns":[{"name":"id","type":"integer","nullable":false},
 {"name":"name","type":"text","nullable":true}],"rows":[[{"type":"integer","value":1},{"type":"text","value":"Ann"}]],"row_count":1}}
{"session_id":null,"result":{"kind":"write","statement":"INSERT","rows_affected":1,"message":"INSERT completed: 1 row affected."}}
{"session_id":null,"result":{"kind":"ddl","statement":"CREATE TABLE","object":"default.public.people","changed":true,"message":"Table default.public.people created."}}
{"error":{"code":"PARSE_ERROR","message":"the SQL text could not be parsed","detail":"Expected: an SQL statement, found: SELEC …","hint":"Did you mean SELECT?"}}
```

* Always pass values as **`params`** (typed values: `integer`, `text`, `decimal`, `date` …) instead of building SQL text: it is both safe and exact.
* A transaction: send `BEGIN`; the reply carries a `session_id`; send it with every following statement; end with `COMMIT` or `ROLLBACK`.
* Other routes: `GET /readyz`, `/v1/whoami`, `/v1/status`, `/v1/metadata`, `/v1/metrics`, `/v1/metrics/system`, `/v1/catalog/{databases,schemas,tables,indexes}`, `/v1/snapshots`, `/v1/kv/…`, `/v1/range`, and the administrator routes `/v1/admin/*` (backups, check, storage, status, maintenance, shutdown). The statements `POST /v1/sql` accepts are the SQL of section 5.
* Errors have a stable `code` (`PARSE_ERROR`, `UNSUPPORTED`, `NOT_FOUND`, `ALREADY_EXISTS`, `CONFLICT_ERROR`, `FOREIGN_KEY_VIOLATION`, `CHECK_VIOLATION`, `RESOURCE_LIMIT`, `TIMEOUT`, `UNAUTHORIZED` 401, `FORBIDDEN` 403, `RATE_LIMITED` 429, `TOO_MANY_SESSIONS` 429, `STORAGE_EXHAUSTED` 507 …), a message, optional detail and a hint. Errors never contain file paths or keys.

---

## 7. Disk space and moving the database

* rubiXDb watches the free space of the drive that holds the database, keeps a short history, and **forecasts the days until it is full**. From the *warning* level (under 10 % free, or a forecast under 14 days) the console shows a **banner** (with the exact command and a Copy button) and a toast; the resource panel has a **Disk space** section; `rubixdb status --system` and `rubixdb instance status` print the same.
* A **reserve** file (16–256 MiB, 1 % of the drive) next to the data is handed back to the database when it reports storage pressure, so a stuck write can finish. When the drive is truly full, writes are refused with **507 `STORAGE_EXHAUSTED`**; reads still work.
* **Move the database to another drive** (command line only), with the instance stopped:

```
rubixdb instance move default --to D:\rubixdb --stop
rubixdb instance move default --remove-old --confirm default     (after checking it works)
```

The data and backups are copied, compared byte for byte, integrity-checked, and only then is the instance switched. The name, port, API key and URL do not change. The old files stay until you confirm. Refused: a non-empty folder, a link, a network drive, a cloud-sync folder (`--allow-synced-folder` overrides), a folder inside another instance, too little space. If the moved folder is missing at the next start (for example the drive is unplugged) the instance **refuses to start** instead of creating an empty database. Full details: `docs/DISK_SPACE_AND_MOVE.md`.

---

## 8. Access and security

* **Local only:** the server listens on `127.0.0.1`; there is no way to reach it from the network.
* **One key per instance:** the API key is created on first start and kept in `credentials.json`, readable only by you (on Windows only your account and the system). The console signs you in by itself (the key is passed in the address fragment, which is never sent to a server). Backups never contain the key.
* **Rotate the key:** stop the instance, then `rubixdb instance rotate-credential NAME --confirm NAME`. Every client still using the old key gets 401 and must read the new one from the file.
* **Roles:** the local instance has one principal, `local`, with the administrator role. (Read-only keys exist in the API model, not in the single-user local setup.)
* **Safe by construction:** values you store are shown as inert text, never as HTML; the terminal prompt escapes control characters; one process owns an instance (a second attaches); request size, statement size, nesting, concurrent transactions and request rate are bounded.
* **Audit log:** `security.log` in the instance folder records sign-in failures, administrator actions, schema changes (create/drop), start/stop, credential replacement and instance moves, as bounded, rotated one-line JSON; it never contains SQL text, values or keys.
* **If you are denied access to rubiXDb's own folder** (for example it was once started from an "administrator" prompt), rubiXDb says which folder and offers, once and only if you agree, a Windows approval step that gives your account its ownership back. rubiXDb itself never runs as administrator. `RUBIXDB_NO_ELEVATE=1` switches the offer off.
* Do not share `credentials.json`. If a copy of it was ever committed or sent, rotate the key.

---

## 9. Settings (environment variables)

Set these in the environment before starting (a bad value stops startup with a message naming it; unset or empty means the default).

| Variable | Meaning (default) |
|---|---|
| `RUBIXDB_INSTANCE_NAME` | instance used when `--instance` is not given (`default`) |
| `RUBIXDB_INSTANCES_ROOT` | where instances live (the folders of section 2.4) |
| `RUBIXDB_PORT_CONFLICT` | `ask` (terminal default), `next`, `stop` |
| `RUBIXDB_LAUNCHER` | `always` / `never` for the start menu |
| `RUBIXDB_NO_ELEVATE` | `1` switches off the offer of the one-time Windows approval step |
| `RUBIXDB_FRONTEND_DIST` | serve the console from this folder instead of the one built into the program |
| `RUBIXDB_LOCAL_RATE_LIMIT_RPS` / `_BURST` | requests per second per principal (100,000) / burst (200,000) |
| `RUBIXDB_INSTANCE_RETRY_BUDGET_MS` | how long to wait for a running instance to answer, 0–600,000 (10,000) |
| `RUBIXDB_STARTUP_GRACE_MS` | extra wait after a slow start before exiting, 5,000–3,600,000 (60,000) |
| `RUBIXDB_LOCAL_MAX_BLOCKING_THREADS` | cap on threads running SQL, 16–512 (512) |
| `RUBIXDB_LOCAL_SSTABLE_VERIFY_MIB_PER_SEC` | background data check speed, 0 (off) – 1024 (64; unvalidated default) |
| `RUBIXDB_ALLOW_TRUNCATE_CORRUPT_WAL` | `1` accepts losing log records after a damaged log (runbook, incident I-6) — **last resort** |
| `RUBIXDB_API_URL`, `RUBIXDB_API_KEY` | use another server instead of the local instance (key from the environment only) |

---

## 10. Messages you may see, and what to do

| You see | It means | Do this |
|---|---|---|
| `Port 302 is in use by a program that is not a rubiXDb instance … Starting "default" on port 303.` | something else holds 302; nothing was touched | nothing: use the printed address |
| `An instance named "default" is already running at …` | a second start attached to the first | continue, or `rubixdb instance stop default` first |
| `rubixdb: no clean-shutdown attestation (first start of this build, or the last stop was not graceful) …` | the previous run ended abruptly (kill, crash, power cut). Data was recovered from the log. | nothing; stop it gracefully next time (close its window, `\q`, or `rubixdb instance stop`) |
| `That API key was not accepted` (console sign-in) | wrong instance's key or a mistyped one | open the console with `rubixdb gui` (it signs in itself); keys are per instance |
| `the storage folder … is not available (is the drive connected?)` | the database was moved to a drive that is not there | reconnect the drive; nothing was created |
| `cannot use the instance directory … Access is denied` | Windows denies access to rubiXDb's folder | follow the offer on screen, or give your account ownership of the folder |
| Banner "The drive holding your database is running low / almost full" | see section 7 | free space, or move the database |
| HTTP 507 `STORAGE_EXHAUSTED` | the drive is full | free space on it, or move the database; reads still work |
| `RESOURCE_LIMIT` | the statement is too large or builds too much in memory | narrow it (`WHERE`, `LIMIT`) or split it |
| `UNSUPPORTED …` | the statement uses something rubiXDb does not do (section 5.8) | the message says what; nothing was changed |
| `CONFLICT_ERROR` | another transaction changed the same rows first | roll back and retry |

More, with procedures: `docs/RUNBOOK.md`.

---

## 11. Where to read more

| Topic | Document |
|---|---|
| Operating it | `docs/RUNBOOK.md` |
| SQL coverage with evidence | `docs/SNOWFLAKE_SQL_COMPATIBILITY.md`, `docs/audits/RELEASE_BASELINE_2026-10-20.md` |
| What is guaranteed after a crash | `docs/DURABILITY_CONTRACT.md`, `PHASE_RUBIXDB_DISASTER_RECOVERY.md` |
| Disk space and moving | `docs/DISK_SPACE_AND_MOVE.md` |
| Data import page | `docs/FRONTEND_DATA_INTEGRATION.md` |
| Start menu, clean stop, access denied | `docs/START_MENU_AND_DOUBLE_CLICK.md`, `docs/CLEAN_STOP_ON_WINDOW_CLOSE.md`, `docs/ACCESS_DENIED_AND_ADMIN.md` |
| Security review | `PHASE_RUBIXDB_FINAL_SINGLE_NODE_SECURITY.md`, `PHASE_RUBIXDB_INSTANCE_SECURITY.md` |
| Architecture | `ARCHITECTURE.md`, `PHASE_RUBIXDB_GUI_ARCHITECTURE.md`, `PHASE_RUBIXDB_INSTANCE_ARCHITECTURE.md`, `PHASE_API_ARCHITECTURE.md` |
| Open problems | `OPEN_ITEMS.md`, `docs/PROJECT_STATE.md` |
