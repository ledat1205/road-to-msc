# Evolution of Database query engine

In the early stages, I/O limitations overshadowed computation time for queries. Query performance primarily hinged on the execution **plan selected by the query optimizer.**
**Nowadays, with advancements in computer hardware, the query scheduler and plan executor are gaining significant prominence.**

## Classical Volcano Engine

The **Volcano model** is the classic way many databases (like MySQL, Oracle, and SQL Server) process queries internally.

It’s **iterator-based**, meaning each operator in the query plan (e.g., `SCAN`, `FILTER`, `JOIN`, `AGGREGATE`) behaves like a little iterator with three key functions:
- `open()` → Initialize or prepare to start reading data.
- `next()` → Get the next row of data.
- `close()` → Clean up when done.

Operators are arranged in a **tree** — the **top operator** (like an `AGGREGATE` or `SELECT`) requests rows by calling `next()` on its child operator, which then calls `next()` on _its_ child, and so on, all the way down to the bottom where the actual table data lives.

This is why it’s called a **pull-based model** — the data is _pulled up_ one row at a time, starting from the top.
