# SQLite migrations

`ecosystem::sql::migration` adds versioned, forward-only migrations to the
existing SQLite adapter. It has no additional module or native dependencies.
The public entry points explicitly accept `sqlite::SqliteConnection`: the
metadata SQL, transaction behavior and tests target SQLite. Other SQL dialects
need their own adapter before this runner can support them.

```goml
use ecosystem::sql::{Connection, Driver, Error};
use ecosystem::sql::sqlite::{SqliteDriver};
use ecosystem::sql::migration::{Migration, Migrator, Report};

fn upgrade(path: string) -> Result[Report, Error] {
    let database = SqliteDriver::new().connect(path)?;
    defer { let _ = database.close(); };
    let initial = Migration::new(1, "create jobs", Vec::from_array([
        "CREATE TABLE jobs(id INTEGER PRIMARY KEY, title TEXT NOT NULL)",
    ]))?;
    let state = Migration::new(2, "add completion state", Vec::from_array([
        "ALTER TABLE jobs ADD COLUMN complete INTEGER NOT NULL DEFAULT 0",
        "CREATE INDEX jobs_complete ON jobs(complete)",
    ]))?;
    let plan = Migrator::new(Vec::from_array([state, initial]))?;
    plan.migrate_sqlite(database)
}
```

`Migration::new(version, name, statements)` snapshots the statement vector.
`version()`, `name()`, `statements()` and `checksum()` inspect the definition;
the returned statement vector is also a copy. Versions are positive `i64`
integers, and names contain 1 through 256 UTF-8 bytes with at least one
non-whitespace character. Each migration contains 1 through 1,024 nonblank SQL
statements, each at most 1 MiB and together at most 16 MiB. Each string must
contain one complete SQLite statement; a trigger body belongs in one string.
The existing SQLite adapter rejects multiple statements and explicit
transaction control. SQL validity is checked by SQLite during migration.

`Migrator::new(definitions)` snapshots and sorts up to 10,000 definitions by
numeric version, rejects duplicates, and caps total SQL at 64 MiB. Versions
need not be consecutive. Keep every previously applied definition in the plan,
and append new versions above the greatest applied version.

`plan.status_sqlite(connection)` returns `Status { applied, pending }` from a
single read transaction. Each `Applied` record has `version`, `name` and
`checksum`; pending entries are `Migration` values. This operation does not
create the metadata table or apply SQL. An empty database has no applied
entries. Status checks the same history integrity rules as migration, so
changed, missing or reordered applied definitions return an error.

`plan.migrate_sqlite(connection)` returns
`Report { previously_applied, applied }`. The `applied` vector contains only
entries committed by this call. A second call with the same plan returns an
empty vector. The runner reserves `main._goml_migrations` with `version INTEGER
PRIMARY KEY`, `name TEXT NOT NULL` and `checksum TEXT NOT NULL`; applications
must not write this table. Reads and writes qualify `main` so a temporary table
cannot shadow the ledger. History collection is capped at 10,000 rows and
4,000,000 bytes of value payload.

The SHA-256 input begins with the ASCII string `goml.sql.migration.v1:` and
then frames the version's decimal text, name, statement count's decimal text,
and each statement in order. Each field is encoded as decimal UTF-8 byte
length, a colon, then its exact bytes. Checksums include whitespace, comments,
names and statement boundaries. They detect accidental edits; this is not a
database authentication or tamper-proof log. A changed checksum/name, unknown
applied version or applied history that is not a prefix of the sorted plan
returns `ErrorKind::Argument` before pending SQL runs. Restore the original
applied definition and add a new migration to change schema or data.

The runner acquires `BEGIN IMMEDIATE` before creating or reading history and
keeps one transaction through all pending statements, history inserts and final
validation. A failure rolls back the entire pending batch, including metadata
creation on a fresh database, while preserving previously committed migrations.
Statement errors retain the backend kind and source and identify the version,
name and one-based statement number. Commit errors trigger rollback; rollback
errors are appended without replacing the original error category. Panic
unwinding also attempts rollback. A failed pending migration may be corrected
and retried because it was never recorded as applied.

SQLite's write lock coordinates runners on separate connections and processes.
A competing runner on a normal file-backed database waits according to the
connection's busy timeout or returns `Busy`; the runner does not add retries.
Shared-cache lock waits have different backend behavior and may exceed that
timeout. Use ordinary file-backed connections for concurrent migrations;
SQLite also [discourages shared-cache mode](https://www.sqlite.org/sharedcache.html).
Retrying after the other transaction
commits rechecks history before executing SQL. Use a separate connection or a
pool lease per caller, and finish any existing cursors/transactions before
calling the runner. `pool.with_connection(|connection|
plan.migrate_sqlite(connection))` scopes acquisition and cleanup. A pool's
context acquisition timeout does not cancel SQL after checkout.

Migration SQL is trusted application code and should contain transactional
schema/data changes in the main database. Do not change connection settings,
attach/detach databases, write the migration table, invoke functions with
external side effects or use nontransactional maintenance operations. Set any
required PRAGMAs before running migrations; `journal_mode=OFF` is rejected
because SQLite cannot promise rollback with journaling disabled. Foreign-key
enforcement and other connection policies remain those configured by the
SQLite adapter. The runner does not split SQL files, infer a baseline for an
existing schema, run down migrations, offer interactive repair, or support
callbacks and nontransactional migrations. To start tracking an existing
database, define the changes that should run from its known schema and retain
those definitions thereafter.

The transaction and locking contract follows SQLite's
[transaction documentation](https://www.sqlite.org/lang_transaction.html).
The journal and foreign-key limitations follow its
[PRAGMA documentation](https://www.sqlite.org/pragma.html).
The independent downstream executable in
[`testdata/downstream/native`](../testdata/downstream/native/) exercises an
existing-database upgrade, repeat application, drift rejection and rollback.
