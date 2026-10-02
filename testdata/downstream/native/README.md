# SQL native downstream fixture

An independent native downstream module using `ecosystem::sql` over its SQLite adapter to create
a table, bind a value in a transaction, commit, read a typed result row, and
reuse a connection through `SqlitePool`.
Its Go module maps the existing local SQLite FFI adapter and pins the same
native driver dependencies as `ecosystem::sqlite`.
