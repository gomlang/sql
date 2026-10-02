# SQLite adapter

`SqliteDriver::new().connect(dsn)` opens a native-backed database, and
`SqliteDriver::new().memory()` opens an independent in-memory database.
`SqliteConnection` implements the generic `sql::Connection` and `Executor`
traits; `SqliteTransaction` implements `sql::Transaction` and `Executor`;
`SqliteRows` implements `sql::RowCursor`.

The adapter copies binary values at the boundary and maps SQLite errors into
the generic error categories while preserving the SQLite message in `source`.
It inherits the native backend's single physical connection per database,
serialized calls, explicit cursor closure, transaction modes and busy policy.
A query or cursor cannot outlive a closed connection. There is no separate
native or Go FFI implementation in this package.

`SqlitePool::new(driver, dsn, PoolPolicy)` provides synchronized bounded reuse
across parallel tasks. `max_open` bounds physical connections, `max_idle`
controls retained connections, and `max_result_rows` caps pooled collection.
`with_connection` preserves immediate `Busy` when exhausted;
`with_connection_with_context(ctx, callback)` waits for a lease with context
cancellation and deadlines. `execute_with_context`, `query_all_with_context`,
`query_optional_with_context` and `query_one_with_context` offer the same
acquisition behavior. Contexts govern acquisition only; native opening and SQL
execution are not interrupted. Waiting order is not guaranteed.

Callbacks must close cursors, finish child tasks using the lease, and leave the
connection open. On return, the pool waits for current calls, expires the lease,
and checks for live statements, cursors and transactions or a closed
connection. It closes and discards an invalid lease, rolls back unfinished
transactions, and returns a recoverable cleanup error. The next lease can open
a fresh connection. `close` wakes waiting acquisitions with `Closed` and closes
idle connections immediately; active callbacks finish and their connections
close when returned. Copied connection, cursor and transaction handles expire
with the lease and cannot affect a later lease's physical connection.

A `:memory:` data source is private to each physical connection. Keep
`max_open = 1` for one in-memory database, or use a file data source to share
data across connections. See the [module README](../README.md) for the
acquisition context example and full pool contract.
