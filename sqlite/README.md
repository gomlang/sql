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

`SqlitePool::new(driver, dsn, PoolPolicy)` provides bounded serial reuse of
these connections. `max_open` refuses nested acquisition once full, `max_idle`
controls retained connections, and `max_result_rows` caps pooled collection.
`with_connection` requires its callback to close cursors and leave the
connection open for return to the pool. On return, the pool checks for live
statements, cursors and transactions or a closed connection. It closes and
discards an invalid lease, rolls back any unfinished transaction, and returns
a recoverable cleanup error. The next lease opens a fresh connection. `close`
rejects new acquisitions and closes idle connections immediately; active
leases close when returned. Copied connection handles retained by a callback
expire when its lease ends; they cannot query, start a transaction or close a
later lease's physical connection. There is no wait queue or synchronization
for parallel task access.
