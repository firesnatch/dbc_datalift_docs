# DBC DataLift RPM User Guide

This guide explains how to install the psql client and the `dbc_isam_fdw` RPM, where files are installed, how tables/columns are discovered, and how to operate configuration safely.

## 1. Install psql (PostgreSQL client)

`psql` is the command-line client used to verify connectivity, list foreign tables, inspect columns, and run troubleshooting queries.

Rocky Linux 8/9 (AppStream client):

```bash
sudo dnf install -y postgresql
psql --version
```

If your environment uses PGDG major-specific packages, install the matching client and use its full path:

```bash
sudo dnf install -y postgresql16
/usr/pgsql-16/bin/psql --version
```

Notes:

- The `dbc_isam_fdw` package can run sync jobs without an interactive user shell, but operators still need `psql` for validation and support workflows.
- If `psql` is installed in a major-specific location, either use the full path or add that directory to `PATH`.

## 1.1 Initialize PostgreSQL on a new Rocky host (first-time only)

Include this step in your instructions if the host does not already have an initialized PostgreSQL data directory.

Do not run initdb on an existing database host unless you intentionally want a brand-new empty cluster.

AppStream-style setup (generic `postgresql` service):

```bash
sudo dnf install -y postgresql-server
sudo postgresql-setup --initdb
sudo systemctl enable --now postgresql
```

PGDG or major-specific setup (example for PostgreSQL 16):

```bash
sudo dnf install -y postgresql16-server
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
sudo systemctl enable --now postgresql-16
```

Quick check:

```bash
sudo -u postgres psql -d postgres -c "select version();"
```

When this is required:

- New Rocky 8/9 VM or server with no existing PostgreSQL cluster.
- Lab/test environments being built from scratch.

When this is not required:

- Existing production host where PostgreSQL is already initialized and running.
- Managed PostgreSQL service where cluster lifecycle is handled externally.

## 2. Install the RPM

Install the package for your PostgreSQL major version:

```bash
sudo dnf install ./postgresql16-dbc_isam_fdw-0.1.0-1.el9.x86_64.rpm
```

Replace `16`/`el9` with the package that matches your system and PostgreSQL major version.

If Python is not already installed, `dnf` will install the required Python runtime dependency automatically from enabled repositories.

Immediately after install, verify the sync service started successfully:

```bash
sudo systemctl status dbc_isam_fdw_sync.service --no-pager -l
```

If the service is failed, inspect recent logs to find the exact startup error:

```bash
sudo journalctl -u dbc_isam_fdw_sync.service -n 200 --no-pager
```

## 3. Where files are installed

The RPM installs these components:

- Config file:
  - `/etc/dbc_isam_fdw/dbc_isam_fdw.conf`
- Generated SQL output directory:
  - `/var/lib/dbc_isam_fdw/generated`
- Sync and utility scripts:
  - `/usr/libexec/dbc_isam_fdw/tools/`
- DEF-to-DDL generator script:
  - `/usr/libexec/dbc_isam_fdw/sql/generate_fdw_tables_from_defs.sh`
- systemd units:
  - `/usr/lib/systemd/system/dbc_isam_fdw_sync.service`
  - `/usr/lib/systemd/system/dbc_isam_fdw_sync.timer`
- PostgreSQL extension artifacts (path depends on distro toolchain):
  - `dbc_isam_fdw.so` under your PostgreSQL `pkglibdir`
  - `dbc_isam_fdw.control` and `dbc_isam_fdw--0.1.0.sql` under your PostgreSQL `sharedir/extension`

Typical extension paths are one of:

- `/usr/pgsql-<major>/lib` and `/usr/pgsql-<major>/share/extension`
- `/usr/lib64/pgsql` and `/usr/share/pgsql/extension`

To see exact installed paths on a host:

```bash
rpm -ql postgresql16-dbc_isam_fdw
```

## 4. Connect and list available FDW tables

Use local admin access that works with default PostgreSQL host auth settings:

```bash
sudo -u postgres psql -d postgres
```

List only DataLift tables in `public`:

```bash
sudo -u postgres psql -d postgres -c "SELECT foreign_table_name FROM information_schema.foreign_tables WHERE foreign_table_schema = 'public' ORDER BY foreign_table_name;"
```

If tables are missing, run a manual sync:

```bash
sudo /usr/libexec/dbc_isam_fdw/tools/sync_fdw_tables.sh
```

## 5. Find columns in a table

For one table (example: `im_0001`):

```bash
sudo -u postgres psql -d postgres -c "SELECT column_name, data_type, ordinal_position FROM information_schema.columns WHERE table_schema = 'public' AND table_name = 'im_0001' ORDER BY ordinal_position;"
```

## 6. Config file location and settings

Primary config file:

- `/etc/dbc_isam_fdw/dbc_isam_fdw.conf`

Runtime scripts load config in this order:

1. `DBC_FDW_CONFIG_FILE` (if set)
2. `/etc/dbc_isam_fdw/dbc_isam_fdw.conf`
3. `/etc/sysconfig/dbc_isam_fdw`

Environment variables override config file values.

### 6.0 Config syntax and quoting rules

The config file uses shell-style `KEY=VALUE` assignments.

- Quotes are optional for simple values with no spaces or separators.
- Quotes are recommended as a safe default and are used in this guide.
- Use quotes when values contain spaces or separator characters.

Examples:

```ini
# Works without quotes (simple value)
DBC_SYNC_APPLY=1

# Recommended style (consistent and safe)
DBC_SYNC_APPLY="1"

# Quote values containing separators
DBC_DEF_DIRS="/ppro/src/pdfp:/opt/customer/defs"
```

Important: if you put spaces around `=`, shell-style parsing can break. Use `KEY=VALUE`, not `KEY = VALUE`.

### 6.1 Why DEF settings matter

DEF files are the product metadata source of truth. They define field layouts and mappings that drive generated foreign-table DDL.

- Which foreign tables exist is determined by DEF/data discovery.
- Which columns and data types each table has are determined by DEF metadata parsing.

If DEF selection is wrong, tables/columns in PostgreSQL will be missing or incorrect. Most production issues start with these settings.

Sync workflow summary:

1. Discover targets by pairing DEF metadata with matching data files.
2. Generate per-table SQL files in `DBC_GENERATED_SQL_DIR`.
3. Optionally apply SQL in PostgreSQL to make foreign tables match generated definitions.

The settings below control each stage of that workflow.

### 6.2 Detailed settings reference

Each setting below includes purpose, whether it is required, default value, when it is used, and an example.

#### `DBC_DEF_DIRS`

- Purpose: Directories to scan for `*DEF.TXT` metadata files.
- Required: Conditionally required. You must set it if your DEF files are not in the default location.
- Default: `/ppro/src/pdfp`
- When used: Every discovery/generation/sync run.
- Example:

```ini
DBC_DEF_DIRS="/ppro/src/pdfp:/opt/customer/defs"
```

- Operational note: The parser accepts `:`, `;`, or `,` as separators.
- Detailed behavior:
  - Each directory is scanned for files matching `*DEF.TXT`.
  - DEF names are normalized case-insensitively; first discovered file name wins if duplicates exist across folders.
  - If no DEF targets can be paired with data files, sync fails with a non-zero status.
- When to change it:
  - Set it whenever you want to include other DEF files that live outside `/ppro/src/pdfp`. For example, accounting data in `/ppro/src/arfp`.
  - When you are working on a copied system.  For example `/ppro/copies/test/src/pdfp`.

#### `DBC_DEF_FILES`

- Purpose: Limits processing to a specific DEF file subset.
- Required: No.
- Default: empty (process all discovered DEF files in `DBC_DEF_DIRS`).
- When used: Discovery/generation/sync runs where you want narrower scope.
- Example:

```ini
DBC_DEF_FILES="IMDEF.TXT,PMDEF.TXT"
```

- Operational note: This is the safest way to stage rollout by table families. If omitted, all eligible DEF files are read.
- Detailed behavior:
  - This is an allow-list, not a deny-list.
  - If you specify a DEF file here and it does not exist in `DBC_DEF_DIRS`, sync exits with an error.
  - Omitted/empty means auto-discover all `*DEF.TXT` files.
- Product context:
  - DEF files determine table and column definitions, so this setting directly controls which tables can appear in PostgreSQL.
  - Use it to safely expose only selected modules during initial rollout.

#### `DBC_DATA_DIR`

- Purpose: Directory containing source data files (`*.TXT`) used by generated foreign tables.
- Required: Yes, in practice. The default only works if your data is in the expected standard location.
- Default: `/ppro/data`
- When used: Target discovery and generated table options for data file access.
- Example:

```ini
DBC_DATA_DIR="/srv/dbc/data"
```

- Detailed behavior:
  - Target discovery looks for data files shaped like `<PREFIX><COMPANY>.TXT`, where `<PREFIX>` comes from each DEF file name and `<COMPANY>` is a 4-digit company suffix.
  - A table is generated only when both DEF metadata and a matching data file are found.
  - If the data directory does not exist, sync exits with an error.
- Why this matters:
  - Wrong path means zero or incomplete table discovery even if DEF files are correct.

#### `DBC_ISAM_DIR`

- Purpose: Directory containing index files (`*.ISI`) used for indexed reads.
- Required: Yes, in practice.
- Default: `/ppro/isam`
- When used: During generated option output and runtime indexed access behavior.
- Example:

```ini
DBC_ISAM_DIR="/srv/dbc/isam"
```

- Detailed behavior:
  - Generated table definitions include options that inform indexed/keyed access paths.
  - At runtime, mismatched data/index roots can cause poor performance or failed keyed lookups.
- Why this matters:
  - `DBC_DATA_DIR` and `DBC_ISAM_DIR` should point to the same dataset version (same snapshot/cut) to avoid key/data skew.

#### `DBC_GENERATED_SQL_DIR`

- Purpose: Output directory for generated `CREATE FOREIGN TABLE` SQL files.
- Required: No, if default location is acceptable and writable.
- Default: `/var/lib/dbc_isam_fdw/generated`
- When used: Generation and apply phases.
- Example:

```ini
DBC_GENERATED_SQL_DIR="/var/lib/dbc_isam_fdw/generated"
```

- Operational note: Keep this directory persistent and writable by the service user (`postgres`) because systemd sync jobs run as `postgres`.
- Detailed behavior:
  - Generation removes old `*.sql` files in this directory and writes the current set from DEF/data discovery.
  - Drift checks compare freshly generated SQL to files in this directory.
  - Apply mode includes these files into PostgreSQL using `psql` (`\i <file>`).
- Why this matters:
  - If this path is not writable, sync cannot generate or refresh table definitions.
  - If it is ephemeral, every restart appears as drift and can trigger unnecessary churn.

#### `DBC_SYNC_MODE`

- Purpose: Chooses whether sync should verify state only, refresh unconditionally, or refresh only when drift is detected.
- Required: No.
- Default: `auto`
- Allowed values:
  - `auto`: detect drift and apply only when needed.
  - `force`: regenerate and apply every run.
  - `check`: validate only; no generation/apply.
- When used: At sync script startup.
- Example:

```ini
DBC_SYNC_MODE="check"
```

- Detailed behavior:
  - `check`: compares expected generated state and verifies expected foreign tables exist; returns non-zero if out of sync.
  - `auto`: runs check first, then regenerates/applies only when differences or missing foreign tables are found.
  - `force`: skips drift decision and always regenerates; applies when `DBC_SYNC_APPLY="1"`.

#### `DBC_SYNC_APPLY`

- Purpose: Controls whether the sync job only generates SQL files, or also executes SQL that updates PostgreSQL foreign tables.
- Required: No.
- Default: `1`
- Allowed values:
  - `1`: execute SQL changes in PostgreSQL.
  - `0`: do not execute SQL changes; generation/check only.
- When used: After drift/missing-table detection.
- Example:

```ini
DBC_SYNC_APPLY="0"
```

- Detailed behavior:
  - With `DBC_SYNC_APPLY="1"`, sync builds an apply script that:
    - ensures the extension/server exist,
    - drops existing foreign tables for discovered targets,
    - re-includes generated SQL files to recreate the current table definitions.
  - With `DBC_SYNC_APPLY="0"`, sync still discovers targets and can regenerate files, but PostgreSQL catalog objects are not changed.
  - This setting is useful for staged rollouts:
    - generation host: `DBC_SYNC_APPLY="0"`
    - database host: `DBC_SYNC_APPLY="1"`

#### `DBC_SYNC_LOCK_WAIT_SECONDS`

- Purpose: Maximum wait time to acquire the sync lock that prevents two sync jobs from running at the same time.
- Required: No.
- Default: `30`
- When used: Before sync begins; lock acquisition phase.
- Example:

```ini
DBC_SYNC_LOCK_WAIT_SECONDS="120"
```

- What is locked:
  - The script creates a lock directory at:
    - `/tmp/dbc_isam_fdw_sync.lock` (or `${TMPDIR}/dbc_isam_fdw_sync.lock` if `TMPDIR` is set).
  - If that lock already exists, another sync process is active (or left stale lock state).

- What is being synchronized:
  - DEF/data discovery to identify target tables.
  - SQL generation into `DBC_GENERATED_SQL_DIR`.
  - Optional PostgreSQL apply step (drop/recreate foreign tables to match generated definitions).

- Why the lock exists:
  - Prevents race conditions where two runs both write generated SQL and/or both apply DDL at the same time.
  - Avoids partial or conflicting table state caused by overlapping DROP/CREATE sequences.

- How timeout behaves:
  - If lock is acquired before timeout, run proceeds.
  - If timeout is reached, sync exits with an error instead of starting concurrently.

#### `DBC_SYNC_DB_URL`

- Purpose: Full PostgreSQL connection URL used by sync jobs.
- Required: Optional, but recommended when local defaults are not correct.
- Default: empty.
- When used: Any script step that runs SQL via `psql`.
- Example:

```ini
DBC_SYNC_DB_URL="postgresql://postgres@localhost:5432/postgres"
```

- Operational note: If set, this is used as the primary connection target.
- Detailed behavior:
  - When set, the sync script calls `psql <DBC_SYNC_DB_URL> ...` and still appends `DBC_SYNC_PSQL_ARGS` unless you clear them.
  - Default `DBC_SYNC_PSQL_ARGS` is `-U postgres -d postgres`; if your URL already includes user/database, set `DBC_SYNC_PSQL_ARGS=""` to avoid ambiguity.
- Recommended pattern:

```ini
DBC_SYNC_DB_URL="postgresql://sync_user@dbhost:5432/postgres"
DBC_SYNC_PSQL_ARGS=""
```

#### `DBC_SYNC_PSQL_ARGS`

- Purpose: Additional `psql` CLI args when URL is not sufficient or not used.
- Required: Conditionally required. Needed when default local args do not match your environment.
- Default: `-U postgres -d postgres`
- When used: Every internal `psql` invocation.
- Example:

```ini
DBC_SYNC_PSQL_ARGS="-h 127.0.0.1 -p 5432 -U postgres -d postgres"
```

- Detailed behavior:
  - Parsed as shell-style words and passed directly to `psql` for every internal SQL call.
  - Controls host/port/user/database when `DBC_SYNC_DB_URL` is not used.
- Practical guidance:
  - Local peer-auth systems: `-U postgres -d postgres` is usually enough.
  - Remote DB host: include `-h` and `-p` explicitly.
  - If you use `DBC_SYNC_DB_URL`, clear this setting unless you intentionally want additional flags.

#### `DBC_SYNC_PSQL_BIN`

- Purpose: Path to `psql` binary for non-standard installations.
- Required: No.
- Default: `/usr/bin/psql`
- When used: Every script SQL execution.
- Example:

```ini
DBC_SYNC_PSQL_BIN="/usr/pgsql-16/bin/psql"
```

- Detailed behavior:
  - This controls which `psql` executable is used by sync jobs launched from systemd.
  - Useful when `/usr/bin/psql` is missing, or when you require a specific major-version client path.
- Failure mode:
  - If this path is wrong/non-executable, sync fails before SQL checks or apply steps run.

#### `DBC_SYNC_PG_CTL_BIN`

- Purpose: Path override for `pg_ctl` used by the optional startup wrapper script.
- Required: No.
- Default: `/usr/bin/pg_ctl`
- When used: Only when using `tools/start_with_fdw_sync.sh` with `DBC_SYNC_START_POSTGRES=1`.
- Example:

```ini
DBC_SYNC_PG_CTL_BIN="/usr/pgsql-16/bin/pg_ctl"
```

- Detailed behavior:
  - Not required for the default systemd sync service path.
  - Required only if you use the startup wrapper and PostgreSQL binaries are not on `PATH`.

#### `DBC_SYNC_PG_ISREADY_BIN`

- Purpose: Path override for `pg_isready` used by the optional startup wrapper script.
- Required: No.
- Default: `/usr/bin/pg_isready`
- When used: Only when using `tools/start_with_fdw_sync.sh`; it checks whether PostgreSQL is already running before trying to start it.
- Example:

```ini
DBC_SYNC_PG_ISREADY_BIN="/usr/pgsql-16/bin/pg_isready"
```

- Detailed behavior:
  - Not required for normal sync timer/service usage.
  - Useful in custom launch scripts where PostgreSQL may need to be started before sync.

### 6.3 Example production config

```ini
DBC_DEF_DIRS="/ppro/src/pdfp"
DBC_DEF_FILES=""
DBC_DATA_DIR="/ppro/data"
DBC_ISAM_DIR="/ppro/isam"
DBC_GENERATED_SQL_DIR="/var/lib/dbc_isam_fdw/generated"

DBC_SYNC_MODE="auto"
DBC_SYNC_APPLY="1"
DBC_SYNC_LOCK_WAIT_SECONDS="30"

DBC_SYNC_DB_URL=""
DBC_SYNC_PSQL_ARGS="-U postgres -d postgres"
DBC_SYNC_PSQL_BIN="/usr/bin/psql"
DBC_SYNC_PG_CTL_BIN="/usr/bin/pg_ctl"
DBC_SYNC_PG_ISREADY_BIN="/usr/bin/pg_isready"
```

## 7. Optional: check sync status

```bash
DBC_SYNC_MODE=check /usr/libexec/dbc_isam_fdw/tools/sync_fdw_tables.sh
```

Exit status is non-zero if drift or missing foreign tables are detected.
