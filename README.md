# sql

`ecosystem::sql` defines a typed connection, query, cursor and transaction
contract. Its nested [`sqlite`](sqlite/README.md) package adapts the existing
`ecosystem::sqlite` backend; it does not replace the native driver or its
connection and transaction handling.

```toml
[dependencies]
"ecosystem::sql" = "0.1.0"
```

`Driver` opens a `Connection`; `Connection` starts a `Transaction`, and both
provide the `Executor` methods `execute`, `query`, `query_all`, `query_optional`
and `query_one`. `RowCursor` streams result rows and closes explicitly. All
errors are recoverable `sql::Error` values with a `kind`, message and optional
backend source text. SQLite error categories such as busy, cancelled, timeout,
no rows and result limits retain their category through the adapter.

`Query::new(statement, params)` checks a nonempty statement up to 1 MiB. Values
must be bound as `Params::positional` or `Params::named`; callers must not
construct SQL by concatenating untrusted values. The adapter preserves NULL,
INTEGER, REAL, valid UTF-8 TEXT, non-UTF-8 TEXT and BLOB storage classes. `Value`
and `FromValue`/`ToValue` provide strict typed conversion for `i64`, `f64`,
`bool`, `string`, `Bytes` and `Option[T]`. `Row` snapshots columns and values;
blob storage is copied on input and retrieval. `FromRow` maps a row to an
application record. Duplicate column names make `get_named` ambiguous rather
than silently choosing one.

`query_all` requires an explicit row cap of 0 through 1,048,576 and closes its
cursor on both success and failure. `query_one` requires exactly one row;
`query_optional` allows zero or one. Values and rows have no byte-total cap
beyond the backend's limits, so stream large results through `query` and close
the cursor. Transactions expose explicit `commit` and `rollback`; unfinished
transactions should be rolled back with `defer`. The SQLite adapter delegates
transaction isolation, cancellation and connection serialization to
`ecosystem::sqlite`.

`PoolPolicy` validates `max_open` (1..64), `max_idle` (0..max_open) and
`max_result_rows` (0..1,048,576). `sqlite::SqlitePool` opens connections lazily,
returns `Busy` when `max_open` is reached, reuses up to `max_idle` connections,
and closes excess idle connections. `with_connection(callback)` scopes one
lease; `execute`, `query_all`, `query_optional` and `query_one` are short
convenience leases, with `max_result_rows` applied to `query_all`. `stats()`
reports open, idle and in-use counts. `close()` closes idle connections and
marks the pool closed; active leases close when returned. Callbacks must close
their cursors, leave the connection open, and not retain its handle after
returning. On return, a connection with any live statement, cursor or
transaction is closed rather than reused; outstanding transactions roll back.
A callback that closed its connection also causes the pool to discard it.
`with_connection` returns a recoverable cleanup error describing the invalid
lease state, even when the callback itself succeeded. The pool then opens a
fresh connection for the next lease. Each lease also has a separate validity
token: a copied connection retained by the callback returns `Closed` for
queries, transactions and close after the callback returns, including while
the same physical connection serves a later lease.

The pool is serial-use policy, with no wait queue, acquisition timeout or
synchronization across parallel tasks. It refuses an exhausted pool
immediately. Applications needing concurrent sharing must provide their own
serialization or a future synchronized pool implementation.

The module and independent consumer require the native SQLite adapter mapping
and pinned Go dependencies described in the [SQLite README](../sqlite/README.md).
The repository's `go.mod` files use the local `../sqlite`/`../../sqlite`
replacement for verification. `(cd ../verification && just ecosystem-test sql)` formats, builds, tests
and runs the module and consumer with the published dependency interface.
