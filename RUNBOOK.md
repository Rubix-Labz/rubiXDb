# rubiXDb Runbook

**Version described:** rubiXDb 0.1.0, repository state of 2026-10-20 · **Platform verified:** Windows 10 (Linux and macOS: not verified) · **Companion:** `docs/USER_MANUAL.md`

How to read the evidence tags: **[V]** = run and observed while this runbook was written (2026-10-20); **[T]** = covered by an automated test that passed; **[D]** = taken from a repository document and not re-run now (the document is named). Anything untagged is a plain instruction.

> **Read this first.** rubiXDb is **not declared production ready** (`docs/PROJECT_STATE.md`). What is claimed and measured: an acknowledged commit survives a **process** crash with the disk intact (23,386 acknowledged writes, 8 kill cycles, none lost) [D]. What is **not** tested: real power loss, a real full disk, Windows directory-entry durability, Linux and macOS. There is **no point-in-time recovery**. Your protection against losing the disk is a **verified backup on another drive**, taken by you on a schedule (section 6).

---

## 1. System at a glance

| Item | Fact |
|---|---|
| Program | one executable, `rubixdb` (`rubixdb.exe`), console embedded; no service, no installer |
| Process model | one process per running instance; the process that owns an instance holds an OS lock on `instance.lock`; a second one attaches |
| Listener | `127.0.0.1` only; default instance on **port 302**, others take the next free port (303, 304 …), recorded in `instance.json` [T] |
| Instance folder | Windows `%LOCALAPPDATA%\rubiXDb\instances\<name>\`; Linux `~/.local/share/rubiXDb/instances/<name>/` (or `$XDG_DATA_HOME`); macOS `~/Library/Application Support/rubiXDb/instances/<name>/`; `RUBIXDB_INSTANCES_ROOT` overrides |
| Inside | `instance.json` (identity, port, optional moved storage), `credentials.json` (admin API key), `instance.lock`, `security.log`, `disk-history.json`, `data/` (`wal/`, `sstables/`, `MANIFEST`, `DATA_FORMAT`, `WAL_CLEAN_STOP`, `wal-quarantine/` if used), `backups/`, `reserve.bin` |
| Moved storage | after `rubixdb instance move`, `data/`, `backups/` and `reserve.bin` live in the chosen folder together with `.rubixdb-storage.json`; the rest stays in the instance folder |
| Console cache | `…\rubiXDb\frontend\<hash>\` (the console unpacked from the program; recreated when missing) |
| Logs | `rubixdb gui` writes status lines to its **window / stderr**; the only log **file** is `security.log` (bounded, rotated, one JSON line per event, never contains SQL, values or keys) |
| Config | environment variables only (list in `docs/USER_MANUAL.md` section 9); validated before anything is opened |
| Auth | bearer API key per instance; one principal `local` (admin); `GET /healthz` needs no key |
| Dependencies | none to install (no database server, no runtime); a browser for the console |

---

## 2. Routine operations

### 2.1 Start

| Goal | Command | Notes |
|---|---|---|
| Console for the default instance | `rubixdb gui` | prints `rubiXDb instance "default" ready at http://127.0.0.1:302` [V] |
| Without opening a browser (scripts, services) | `rubixdb gui --no-browser` | |
| Another instance | `rubixdb gui --instance NAME` | created on first use |
| SQL prompt | `rubixdb cli` or `rubixdb` | starts an instance if none is running |
| Double-click | open `rubixdb.exe` | animated logo, then GUI / CLI menu [V] |
| Unattended | `rubixdb gui --no-browser` under a scheduler | set `RUBIXDB_PORT_CONFLICT=next` (not asking) and `RUBIXDB_LAUNCHER=never` |

A start that finds a **missing or foreign moved-storage folder refuses** and creates nothing [T]. A start that finds the preferred port busy picks the next free one [V].

### 2.2 Stop (always gracefully)

A graceful stop drains requests, flushes, stops the engine and writes the clean-stop record `data\WAL_CLEAN_STOP`. Use, in order of preference:

1. `rubixdb instance stop NAME` — authenticated shutdown through the instance's own API; waits up to 120 s until the lock is released; prints `instance "NAME" stopped cleanly` [V].
2. Close the console window, press Ctrl+C in the `rubixdb gui` window, or `\q` / Ctrl+D in the SQL prompt. **Closing either window is a clean stop** (fixed 2026-10-20; the SQL prompt used to be ended without the clean-stop record) [T] `cli/tests/window_close_integration.rs`.
3. `POST /v1/admin/shutdown` with `{"confirm":"NAME"}` and an admin key (what `instance stop` calls).

**Last resort only:** killing the process. Data recovers from the log on the next start [D], but the next start prints `no clean-shutdown attestation …` and a damaged log tail can no longer be told from an ordinary crash. After any kill: start it, then run `rubixdb check` (section 5).

### 2.3 Status and health

| Check | Command | Healthy answer |
|---|---|---|
| Is it running? | `rubixdb instance status [NAME]` | `status: running`, with data folder, port and free space [V] |
| All instances | `rubixdb instance list` | name, port, id |
| Liveness | `GET http://127.0.0.1:PORT/healthz` (no key) | `{"status":"ok"}` [V] |
| Readiness | `GET /readyz` (key) | `{"ready":true,"storage_state":"Healthy","index_recovery":"complete","sstable_verification":"complete"}` [V] |
| Detail | `rubixdb status` (`--json`) | instance, WAL, compaction, queries, resources, disk |
| System | `rubixdb status --system` | health, CPU, memory, disk I/O, rates, and the **disk watch** line |
| Storage by table | `rubixdb storage` | size per table and index |
| Integrity | `rubixdb check` | `result: 0 error(s)` |

`storage_state` is `Healthy`, `StoragePressure` (a flush is retrying "no space"; writes still accepted) or `StorageFull` (writes refused with 507). `sstable_verification` is the background data-block check (`complete`, running, or `disabled` when `RUBIXDB_LOCAL_SSTABLE_VERIFY_MIB_PER_SEC=0`).

### 2.4 Monitoring endpoints (all need the key except `/healthz`)

`/v1/metrics` (engine read/write and per-route request metrics), `/v1/metrics/system` (CPU, memory, disk, process I/O, throughput, latency percentiles, WAL, compaction, background work, **`disk.forecast`**) and `/v1/metrics/system/timeseries`, `/v1/observability/{queries,sessions,events,version}`, `/v1/compaction/{status,metrics}`. A value the platform cannot measure is `null`/`-`, never 0 [D]. Log lines and metric labels never contain SQL text, paths or keys.

### 2.5 Routine schedule (suggested; the product schedules nothing itself)

| When | What |
|---|---|
| Every day (or as often as you accept losing data) | `rubixdb backup create nightly-YYYYMMDD`, then `rubixdb backup verify nightly-YYYYMMDD`, then **copy the file to another drive or machine** (section 6) |
| Weekly | `rubixdb check`; look at `rubixdb status --system` disk line; `rubixdb backup list` and delete old backups you no longer need (`backup delete NAME --confirm NAME`) |
| After any abrupt stop | `rubixdb check` |
| Monthly | restore the latest backup into a scratch instance and check it (section 6.4) — an untested backup is not a backup |
| Before any upgrade | backup + verify (section 8) |

---

## 3. Capacity and disk space

### 3.1 What the system tells you

The server samples the free space of the drive holding the data every 10 minutes (history kept in `disk-history.json`), fits a trend over up to 7 days (needs at least 6 samples over 2 hours) and reports **level**, days until full and advice in `GET /v1/metrics/system → disk.forecast` [T] [V].

| Level | Any of | Console |
|---|---|---|
| ok | otherwise | face of the RAM card shows nothing |
| watch | under 20 % free, or forecast under 30 days | badge "Disk shrinking" |
| **warning** | under 10 % free, or forecast under 14 days | **banner + toast** |
| **critical** | under 5 % or under 1 GiB free, forecast under 3 days, or the engine reports `StoragePressure`/`StorageFull` | banner as alert |
| unknown | free space not measurable (Linux/macOS today) | none |

**These thresholds are provisional**: chosen, not measured against real workloads. They measure the whole drive, including what else on it grows.

### 3.2 Actions by level

1. **watch / warning:** find what is growing. `rubixdb storage` lists the tables; `rubixdb maintenance purge-orphans` (dry run) shows data left by dropped tables; delete old backups; `DROP` unused tables then purge. If the drive is shared with other programs, free it there.
2. **Plan the move** if the drive cannot be freed (section 3.3). Do it at the *warning* level, not at *critical*.
3. **critical / 507 `STORAGE_EXHAUSTED`:** writes are refused, reads work. Free space first (delete a backup file, empty the recycle bin), then run the move. The **reserve** file (16–256 MiB) is released automatically to let a stuck flush finish [T unit; real full disk **NOT TESTED**].

### 3.3 Move the database to another drive (runbook)

Preconditions: the destination is a **local** drive folder that is new or empty, not a network or cloud-sync folder, with free space at least the database size + 5 % (minimum 64 MiB). Check with `rubixdb instance status NAME` (data size) and your file manager.

| Step | Command | Check |
|---|---|---|
| 1. Verified backup first | `rubixdb backup create pre-move` then `rubixdb backup verify pre-move`; copy the file elsewhere | verify prints OK |
| 2. Move (stops the instance gracefully with `--stop`) | `rubixdb instance move NAME --to D:\rubixdb --stop` | prints files, bytes, "comparing every copied byte", "integrity check", "switching", then `moved N file(s)` |
| 3. Start and test | `rubixdb gui --instance NAME`; run a `SELECT COUNT(*)` on your tables; `rubixdb instance status NAME` shows `data: D:\rubixdb\data` | counts as before |
| 4. Integrity | `rubixdb check --instance NAME` | 0 errors |
| 5. Free the old drive | `rubixdb instance move NAME --remove-old --confirm NAME` | prints bytes freed |

Behaviour to know: the name, port, API key and URL do not change; the old files are **kept until step 5**; a copy that is interrupted (Ctrl+C, power cut, destination full) leaves the old files in use and **the same command resumes** it (files already complete are not copied again); a second move is refused until step 5. Verified [V] on a 67 MiB database E:→D: and a killed-and-resumed move; large databases (multi-GB) and power loss during a move are **NOT TESTED**. If the destination drive is not connected at a later start, the instance **refuses to start** and creates nothing; reconnect it. `rubixdb instance drop NAME` removes the moved data too (only folders proven to be this instance's). Moving back to the default location is **not implemented**; to return, restore a backup into a new instance.

---

## 4. Instance management

| Task | Command |
|---|---|
| New instance | `rubixdb gui --instance NAME` (first use creates it) |
| List / details | `rubixdb instance list`, `rubixdb instance status NAME` |
| Stop | `rubixdb instance stop NAME` |
| Rotate the API key (offline only) | `rubixdb instance rotate-credential NAME --confirm NAME`; prints the file path, never the key; old key stops working at next start |
| Delete an instance **and its data** | `rubixdb instance drop NAME --confirm NAME` (refused while it runs; irreversible; also removes moved storage and old files of a move) |
| Where is the key? | the `credentials.json` named in `rubixdb instance status` output (`directory:`); never pass it on a command line |

---

## 5. Integrity checking and maintenance

`rubixdb check [--instance NAME | --data-dir DIR] [--json]`. **Online** (instance running): one consistent snapshot, the database keeps serving. **Offline** (stopped): physical checks before any recovery, then a logical pass. It never repairs anything. Exit codes: **0** clean, **1** warnings only, **2** errors, **3** incomplete, **4** could not run.

| Finding (code) | Meaning | Action |
|---|---|---|
| `ORPHAN_TABLE_DATA` / `ORPHAN_INDEX_DATA` (warning) | rows left by `DROP TABLE` / `DROP INDEX` | harmless; reclaim: `rubixdb maintenance purge-orphans` (dry run), then `… --apply --expect N` with exactly the N shown [V] |
| `INDEX_ENTRY_MISSING` / `_STALE` / `_DANGLING` / `INDEX_UNIQUE_VIOLATION` | an index disagrees with its table | `DROP INDEX` then `CREATE INDEX` (proven repair) [D] |
| `ROW_*`, `CATALOG_*` (other than the note below) | damaged rows or catalog | restore the latest verified backup into a new instance (section 6.3) |
| `SSTABLE_MISSING` / `_CORRUPT` / `_SIZE_MISMATCH`, `MANIFEST_*`, `WAL_CORRUPT`, `WAL_UNREADABLE` | damaged engine files | restore a verified backup; do not start over it |
| `WAL_TORN_TAIL`, `WAL_TAIL_*` | an unfinished last log record (normal after a crash) / a tail kept in `wal-quarantine/` | informational unless the start refuses (incident I-6) |
| `CHECK_INCOMPLETE` | the check did not finish | rerun; look at the reason |

**Note (fixed 2026-10-20):** older builds reported `CATALOG_ROW_UNDECODABLE system.columns[…]` and `unknown system table id 8` on databases that had tables created before `ALTER TABLE` support, and failed to verify backups of them (`freshly written backup failed read-back verification`) and refused the reclaim dry run (`a precondition of the operation no longer holds`). The data was intact; the checker was out of date. Run the current build. [V] `docs/CATALOG_ROWS_FROM_OLDER_BUILDS.md`.

Admin page equivalents: **Operations → Integrity check / Backups / Maintenance**.

---

## 6. Backup, restore and disaster recovery

### 6.1 What a backup is

`rubixdb backup create [NAME]` makes a **consistent online** backup (writers are not blocked) of the whole database into `backups/NAME.rbxbackup` (name: 1–64 letters, digits, `.` `_` `-`; never overwrites; a failed one leaves no partial file). It holds data and catalog, **no credentials, no paths**. `rubixdb backup list`, `rubixdb backup verify NAME` (checksums, key order, catalog), `rubixdb backup verify --file PATH` (any file, no instance needed), `rubixdb backup delete NAME --confirm NAME`. Measured [D] `PHASE_RUBIXDB_DISASTER_RECOVERY.md`: 121 MB backup in 2.2 s; writers never blocked.

### 6.2 Recovery objectives (measured on one SATA-SSD machine; yours will differ)

| Failure | Recovery | Data loss |
|---|---|---|
| process crash or kill, disk intact | start it; `rubixdb check` | **0 acknowledged commits** (process kill only) [D] |
| power loss / OS crash | same path | **NOT TESTED** |
| disk or folder lost or damaged | restore the newest verified backup into a **new** instance | everything after that backup (RPO = age of the newest verified backup; ≈163 acknowledged rows/s lost in the measured workload) |
| a bad `DELETE`/`DROP` by a person | restore a backup into a **new** instance and copy the rows back | everything since the backup, for what you restore |
| point in time ("as of 14:03") | **not implemented** (the log is not archived) | – |

RTO measured [D]: crash restart 0.52 s to first query at 1 M rows; restore of a 1 M-row database (78.6 MB backup) 72 s, end to end 74 s; ~21–26 k entries/s, so it grows linearly with size (a second 3 M-entry database took 145 s, half of it the mandatory post-restore integrity check).

### 6.3 Restore procedure

Restore **never overwrites** an existing database and is offline by design.

1. Find the backup file; verify it: `rubixdb backup verify --file "C:\path\NAME.rbxbackup"`.
2. Restore into a **new** instance name: `rubixdb restore --from "C:\path\NAME.rbxbackup" --instance NEW` (or `--data-dir DIR` for a plain folder). It verifies, loads, runs the integrity check and refuses a non-empty destination (`DESTINATION_NOT_EMPTY`) [V][T].
3. Start it: `rubixdb gui --instance NEW`. The restored instance **generates its own API key** (backups hold none); update any client. Its first start prints the `no clean-shutdown attestation` note (a restored database has never been stopped cleanly yet); that is expected [V].
4. `rubixdb check --instance NEW` → 0 errors; spot-check your tables.
5. Switch over: keep using `NEW`, or, to reuse the old name, `rubixdb instance drop OLD --confirm OLD` and restore once more under the old name (do this only when the restored copy is good).

### 6.4 Backup drill (monthly)

Verify the latest file, restore it into `drill`, run `rubixdb check --instance drill`, query a table, then `rubixdb instance drop drill --confirm drill`. A backup kept only on the same drive as the data is **not** disaster recovery: always keep one copy on another drive or machine.

---

## 7. Incident playbooks

Each: **Symptom → Check → Action → Verify.** Never delete `instance.lock`, never copy files out of a running `data/`, never edit `instance.json` or `data/` by hand.

**I-1 "Port … is in use"** — `Port 302 is in use by a program that is not a rubiXDb instance, which is left alone. Starting "default" on port 303.` is normal; use the printed address. To keep 302 for rubiXDb, stop the other program, or `rubixdb instance stop OTHER`, or set `RUBIXDB_PORT_CONFLICT=stop` (stops the other **rubiXDb** instance gracefully). A foreign program is never touched.

**I-2 "instance … is locked by another process that did not answer a real health/identity check"** — the owner may still be starting or stopping. Wait 10–30 s and retry; `rubixdb instance status NAME` shows `locked (…)`. If it persists, find the `rubixdb.exe` process in Task Manager and end it (a last-resort kill; then see I-4). The OS releases the lock when that process exits. Do not delete the lock file.

**I-3 "Access is denied" at start** (`cannot use the instance directory …`) — rubiXDb's folder is owned by another account (typically it was once started from an "administrator" prompt). In a normal terminal rubiXDb explains and offers a one-time Windows approval step that gives your account ownership again; accept it, or run the same from a prompt you type into. Never run rubiXDb elevated routinely. `RUBIXDB_NO_ELEVATE=1` removes the offer. For a moved database the folder is the moved one: fix its permissions yourself. [T] `docs/ACCESS_DENIED_AND_ADMIN.md`.

**I-4 `no clean-shutdown attestation …`** — informational: the previous run ended abruptly. Check: nothing; Action: `rubixdb check`; make sure future stops are graceful (section 2.2). Verify: the note does not return after a graceful stop. The note shows **once more** on an instance whose last stop was abrupt.

**I-5 Instance will not start: `UNSUPPORTED_DATA_FORMAT`** — the data was made by a newer build; the folder is untouched. Use the newer build. Downgrade is not supported; the supported route between builds is backup → restore into a new instance of the target build [D].

**I-6 Start refused: `WAL_CORRUPT` or `WAL_TAIL_DAMAGED`** — the startup guard found a damaged log and **changed nothing**.
* `WAL_CORRUPT`: restore the newest verified backup (6.3). There is no override.
* `WAL_TAIL_DAMAGED` (the log ends before the last recorded clean stop; the message gives the number of acknowledged records missing): first `rubixdb check --data-dir <data>`; prefer restoring a verified backup. **Only if you accept losing those records:** start once with `RUBIXDB_ALLOW_TRUNCATE_CORRUPT_WAL=1` (exactly `1`), then unset it; the removed tail is kept in `wal-quarantine/` and the event is logged to `security.log` [D] `PHASE_ITEM_F07_*`.

**I-7 Start refused: missing table file / engine open failed** — `SSTABLE_MISSING` and similar: restore a verified backup into a new instance. A missing moved-storage folder: reconnect the drive (nothing was created).

**I-8 Start is slow, then `server did not signal readiness within 30s`** — a big recovery or a slow disk. The process may keep waiting up to `RUBIXDB_STARTUP_GRACE_MS` (default 60 s), then exits with code 1; the data folder is safe to restart (equivalent to a kill during start) [D] `PHASE_ITEM_F11_ADR.md`. Raise the grace (up to 3,600,000 ms) and start again; check disk speed and size of `data\wal`.

**I-9 Writes fail with 507 `STORAGE_EXHAUSTED`** — section 3.2 (critical). Free space, then writes resume by themselves when a flush succeeds; confirm `/readyz` shows `Healthy`.

**I-10 401 / "API key was not accepted"** — keys are per instance. Use the key of the instance you are talking to; `rubixdb gui` signs the console in by itself. After `rotate-credential` clients must read the new key from the file. For HTTP clients: `Authorization: Bearer <key>`.

**I-11 429 `RATE_LIMITED` / `TOO_MANY_SESSIONS`** — rate limit (default very high) or more than 50 open transactions on one principal. Commit or roll back open transactions; `GET /v1/observability/sessions` lists them; idle ones close after 300 s.

**I-12 `RESOURCE_LIMIT`, `TIMEOUT`, 413** — the statement is bigger than a limit (manual section 5.9) or ran over 30 s. Narrow it, add an index, or split it. Known slow shapes: a join on an **indexed** key over tables above ~40,000 rows (the planner uses an index nested loop; it measured 40 s at 200,000 rows while the unindexed hash join took 1.2 s), and `SELECT DISTINCT` on tens of thousands of distinct rows (quadratic) [D] `docs/audits/RELEASE_BASELINE_2026-10-20.md` §5. Use `GROUP BY` instead of `DISTINCT`; there is no spill to disk.

**I-13 `CONFLICT_ERROR`** — two transactions changed the same rows; retry the loser.

**I-14 A table was dropped by mistake** — `DROP TABLE` is immediate and not undone by `ROLLBACK`. Restore the newest backup into a new instance and copy the table back (6.3). Do **not** run `purge-orphans --apply` before you have recovered what you need: it deletes the dropped table's leftover rows.

**I-15 Integrity check shows errors** — section 5 table. Never "fix" by deleting files.

**I-16 Lost or exposed API key** — stop the instance, `rubixdb instance rotate-credential NAME --confirm NAME`, update clients. `credentials.json` for the end-to-end test instance is **tracked in git** (`frontend/.e2e-crossbrowser-data/default/credentials.json`), already published: treat that key as burnt and remove the file (open release blocker, section 10).

**I-17 Console shows a blank page or old pages** — close the tab and re-run `rubixdb gui`; the console is served from the program (cache folder `…\frontend\<hash>`); `RUBIXDB_FRONTEND_DIST` overrides it (unset it).

**I-18 SQL prompt or console cannot connect** — `rubixdb instance status NAME`; if `not running`, any `rubixdb` command starts it; if `locked`, I-2.

---

## 8. Upgrade, downgrade and release

### 8.1 Upgrade (new `rubixdb.exe` over an old one)

1. `rubixdb backup create pre-upgrade` → `rubixdb backup verify pre-upgrade` → copy it off the drive.
2. `rubixdb instance stop NAME` for every instance (Windows cannot replace a running `rubixdb.exe`; the same applies when rebuilding in `target\release`).
3. Replace the executable (the console is inside it).
4. `rubixdb gui` → `rubixdb check` → open the console, run a known query.

Supported by test [D]: upgrading data written by the two previous builds tested (Increments 17 and 18). Earlier builds are not claimed. Data folders carry a `DATA_FORMAT` marker; a newer format is refused untouched; **downgrade is unsupported** (best effort only): move data between builds by backup → restore into a new instance.

### 8.2 Build and package (for whoever produces the release)

`scripts/release.ps1`, end to end [D] `PHASE_RUBIXDB_FINAL_SINGLE_NODE_RELEASE.md`: `cargo fmt --check`; `cargo clippy --workspace --all-targets --all-features -D warnings`; `cargo test --release --workspace`; `npm ci` and `npm run build` in `frontend/`; `cargo build --release --locked` with reproducible-build flags (two clean builds gave identical SHA-256); package `rubixdb.exe`, `VERSION`, `SHA256SUMS`; smoke test of the **packaged** copy (version, start, DDL, DML, backup, verify, check, console served, clean stop). By hand: `cd frontend && npm run build && cd .. && cargo build --release -p rubixdb-cli` (output `target\release\rubixdb.exe`, about 22.7 MiB; the console is embedded at build time). `cargo audit` and `cargo deny` were clean when last run [D].

Known state of the release gates (2026-10-20 audit [D] `docs/audits/RELEASE_BASELINE_2026-10-20.md` §3): the default workspace test run had **2 deterministic failures** (`repo_hygiene`: the tracked credentials file) and 3 load-sensitive flaky tests (the WAL throughput gate M1.3, a recursive-CTE snapshot test that fails when run alone, an override-value test); the crash and power-loss suites are feature-gated and **not** in the default run.

---

## 9. Security operations

| Task | How |
|---|---|
| Who can connect | only processes on this computer (loopback). Anyone who can read `credentials.json` has administrator rights on that instance: keep the instance folder private |
| Audit | `security.log` (+ rotated `.1`, `.2` …) in the instance folder: authentication failures (rate-bounded), `/v1/admin/*` actions, `CREATE`/`DROP`/`ALTER`, start/stop, credential replacement, instance moves; bounded size; no secrets or SQL |
| Rotate key | section 4 |
| Remove a user's access | there are no accounts; rotate the key |
| Review exposure | `netstat -ano | findstr 302` must show only `127.0.0.1`; CORS has no allowed origins by default |
| Backups | contain data, not keys; protect them like the data |
| Dependencies | `cargo audit`, `cargo deny` before each release |

---

## 10. Known limitations and open blockers (do not rely on these)

* **Not production ready** by the project's own rule (`CLAUDE.md`: every mandatory certification row must be PASS). Open: the **tracked credentials file** in git (release blocker, `repo_hygiene` fails); the WAL throughput gate M1.3 is load-sensitive; **real power loss, a real full disk, Windows directory durability and the class-C crash matrix are not tested**; lifecycle findings F-11 / F-18; see `OPEN_ITEMS.md`.
* **No point-in-time recovery**, no WAL archiving, no replication, no clustering.
* **DDL is not transactional:** `CREATE TABLE`/`ALTER` inside `BEGIN` take effect at once and survive `ROLLBACK`.
* **No spill to disk:** statements needing more than ~1,000,000 rows or ~256 MiB of intermediate state fail with `RESOURCE_LIMIT`.
* **No statistics or cost-based join order**; large indexed joins and `DISTINCT` can be slow (I-12).
* **Disk-space forecasts and thresholds are provisional**; free space is not measured on Linux/macOS (level `unknown`).
* **Moving a database** is tested up to 67 MiB; not for multi-GB, not across a power cut.
* **Downgrade** unsupported. `instance.json` has no format version: an older `rubixdb` ignores the moved-storage setting and would open an empty database in the instance folder — never run an older build against a moved instance.
* The standalone `rubixdb-api` binary is **unsupported** for v1; use `rubixdb gui`.
* Linux and macOS: the program builds for them, but nothing in this runbook was verified there (the double-click menu, clean stop on window close, access-denied step and drive checks are Windows-specific).

---

## 11. Command cheat-sheet

```
rubixdb gui [--instance NAME] [--no-browser]    start/attach, open console
rubixdb cli | rubixdb                           SQL prompt          (-c "SQL" | -f file.sql)
rubixdb menu                                    animated start menu
rubixdb instance list | status [NAME] | stop [NAME]
rubixdb instance move [NAME] --to FOLDER [--stop] [--allow-synced-folder]
rubixdb instance move [NAME] --remove-old --confirm NAME
rubixdb instance drop NAME --confirm NAME
rubixdb instance rotate-credential NAME --confirm NAME
rubixdb status [--system] [--json]              storage | check [--instance NAME | --data-dir DIR] [--json]
rubixdb backup create [NAME] | list | verify NAME | verify --file PATH | delete NAME --confirm NAME
rubixdb restore --from FILE (--instance NEW | --data-dir DIR)
rubixdb maintenance purge-orphans [--apply --expect N]
```

Exit codes: `check` 0 clean · 1 warnings · 2 errors · 3 incomplete · 4 could not run. Other commands: 0 success, non-zero failure (2 = refused or bad usage).

## 12. Evidence index

| Topic | Source |
|---|---|
| Operations layer, interfaces, safety rules | `PHASE_RUBIXDB_PRODUCTION_OPERATIONS_ARCHITECTURE.md`, `…_RESULTS.md`, `PHASE_RUBIXDB_MAINTENANCE_ARCHITECTURE.md`, `PHASE_RUBIXDB_INTEGRITY_ARCHITECTURE.md` |
| Backup/restore/DR measurements | `PHASE_RUBIXDB_BACKUP_RESTORE_ARCHITECTURE.md`, `PHASE_RUBIXDB_DISASTER_RECOVERY.md` |
| Durability promises and gaps | `docs/DURABILITY_CONTRACT.md`, `docs/PERSISTENCE_MODEL.md`, `docs/CRASH_CLASS_MATRIX.md` |
| Release engineering, formats, upgrade | `PHASE_RUBIXDB_FINAL_SINGLE_NODE_RELEASE.md` |
| Security review | `PHASE_RUBIXDB_FINAL_SINGLE_NODE_SECURITY.md`, `PHASE_RUBIXDB_INSTANCE_SECURITY.md`, `PHASE_RUBIXDB_SECURITY_GAP_ANALYSIS.md` |
| Startup, shutdown, configuration | `PHASE_RUBIXDB_CONFIGURATION_STARTUP_SHUTDOWN_CERTIFICATION.md`, `PHASE_ITEM_F11_*`, `PHASE_ITEM_F07_*`, `PHASE_ITEM_F08_*` |
| Observability | `PHASE_RUBIXDB_FULL_OBSERVABILITY_ARCHITECTURE.md` |
| This build's disk space, move, start menu, clean stop, access, catalog fix | `docs/DISK_SPACE_AND_MOVE.md`, `docs/START_MENU_AND_DOUBLE_CLICK.md`, `docs/CLEAN_STOP_ON_WINDOW_CLOSE.md`, `docs/ACCESS_DENIED_AND_ADMIN.md`, `docs/CATALOG_ROWS_FROM_OLDER_BUILDS.md` |
| Backlog and release audit | `docs/audits/RELEASE_BASELINE_2026-10-20.md`, `OPEN_ITEMS.md` |
