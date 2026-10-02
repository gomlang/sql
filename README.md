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
cursor exactly once on success, failure and panic unwinding. If cursor closure
also fails after a read, row-limit or conversion error, the original kind and
source are retained and the cleanup failure is appended to its message. A close
failure after otherwise successful collection is returned directly. `query_one`
requires exactly one row;
`query_optional` allows zero or one.

`query_all_with_limits[T](query, CollectionLimits { max_rows, max_bytes })`
adds a cumulative value payload budget. The row cap is 0 through 1,048,576;
the byte cap is any nonnegative `isize`. Limits are validated before opening
a cursor. Each row is charged before `FromRow` runs: NULL costs zero, INTEGER
and REAL cost eight bytes, and TEXT, non-UTF-8 TEXT and BLOB cost their byte
length. An exact fit succeeds; exceeding either cap returns `Limit` and closes
the cursor with the same error preservation rules. Empty results and zero-byte
values can fit a zero byte budget. With a pool, call the method on the executor
inside `with_connection` or `with_connection_with_context`.

These budgets bound collected input payload, not peak process memory: backend
row materialization, column metadata, container overhead, snapshot copies and
allocations in a custom `FromRow` are outside the budget. The existing
`query_all` remains row-bounded only. Stream large results through `query` and
close the cursor. Transactions expose explicit `commit` and `rollback`; unfinished
transactions should be rolled back with `defer`. The SQLite adapter delegates
transaction isolation, cancellation and connection serialization to
`ecosystem::sqlite`.

`PoolPolicy` validates `max_open` (1..64), `max_idle` (0..max_open) and
`max_result_rows` (0..1,048,576). `sqlite::SqlitePool` is safe to share between
parallel tasks. It opens connections lazily, reserves capacity before opening,
reuses up to `max_idle` connections, and closes excess idle connections. Calls
to the backend do not hold the pool's bookkeeping lock. `stats()` reports open,
idle and in-use counts; connections being opened or closed count as in use
until that work finishes. A `:memory:` data source creates a separate database
for each physical connection, so use `max_open = 1` for a single in-memory
database or a shared file data source for a multi-connection pool.

`with_connection(callback)` scopes one lease and returns `Busy` immediately
when capacity is exhausted, preserving its existing behavior. `execute`,
`query_all`, `query_optional` and `query_one` use the same immediate acquisition;
`max_result_rows` applies to `query_all`.

`with_connection_with_context(ctx, callback)` waits for capacity until a lease
is available, the context is cancelled (`Cancelled`), its deadline expires
(`Timeout`), or the pool closes (`Closed`). Returning or discarding a connection
wakes waiting tasks. Waiting order is not guaranteed. The corresponding
`execute_with_context`, `query_all_with_context`, `query_optional_with_context`
and `query_one_with_context` take `(ctx, query)`. The context controls pool
acquisition, including checks before and after opening a connection; it does
not interrupt native connection opening or SQL execution after checkout.
Use `std::context::with_timeout` or `with_deadline` to bound a wait. A nested
waiting acquisition needs enough pool capacity or a deadline to avoid waiting
for the outer callback to release its own connection.

```goml
use ecosystem::sql::sqlite::SqlitePool;
use ecosystem::sql::{Error, Query, Execution};
use std::context;
use std::time;

fn execute_when_available(pool: SqlitePool, query: Query) -> Result[Execution, Error] {
    context::with_timeout(
        context::Context::background(),
        time::Duration::from_seconds(1),
        |ctx, _| pool.execute_with_context(ctx, query),
    )
}
```

`close()` rejects and wakes new or waiting acquisitions, closes idle
connections, and lets active callbacks finish. Active leases close when
returned. Callbacks must close their cursors, leave the connection open, and
finish any child tasks using the lease before returning. On return, a
connection with a live statement, cursor or transaction is closed rather than
reused; outstanding transactions roll back. A callback that closed its
connection also causes the pool to discard it. Cleanup errors remain
recoverable and are appended to an existing callback error without replacing
its category. The pool can open a fresh connection for the next lease.

A lease's validity is synchronized with its connection, cursor and transaction
calls. Return waits for a call already in progress, then expires the lease
before checking resources or reusing the physical connection. Retained handles
return `Closed` after expiration and cannot act on a later lease. This runtime
check also covers copied handles shared between tasks; it does not make
retaining a lease beyond the callback a supported usage pattern.

The module and native downstream fixture use the native SQLite adapter mapping
and pinned Go dependencies described in the [SQLite README](../sqlite/README.md).
The root `go.mod` uses the local `../sqlite` replacement; the fixture
uses `../../../../sqlite` from `testdata/downstream/native`. `(cd ../verification && just ecosystem-test sql)` formats, builds, tests
and runs the module and fixture through the isolated registry snapshot.

## Development and downstream checks

Requires GoML 0.1.56 or newer. The independent native fixture is in `testdata/downstream/native/`; it retains a separate manifest and Go module for native dependencies. From the library root, run:

```sh
goml test
goml verify --timeout 300s
```

`goml verify` builds and tests the fixture against an isolated registry snapshot. `(cd ../verification && just ecosystem-test sql)` also runs the library-specific smoke and compatibility checks.
