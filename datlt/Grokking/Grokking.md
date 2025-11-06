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

###  Advantages

1. **Modularity and extensibility**  
    Each operator only needs to know how to process one row at a time and can be reused or rearranged easily.  
    → It’s easy to build new query plans or add new operators.

2. **Low memory use**  
    Since rows are processed one by one, operators don’t need to store the whole dataset in memory.  
    → Efficient for streaming or pipeline-style execution (unless something like `SORT` requires buffering).


---

###  Disadvantages

1. **High function call overhead**  
    Each row passes through multiple `next()` calls — one per operator in the chain.  
    So if your query plan has 5 operators and 1 million rows, that’s 5 million nested function calls.
    These are **virtual function calls**, which:
    - Require looking up the function in a **virtual function table (vtable)**.
    - Prevent the compiler from **inlining** (i.e., optimizing away) the calls.
    - Introduce **branch instructions**, which can be **hard for the CPU to predict**.
    - When prediction fails, the CPU pipeline has to reset → **performance drops**.

2. **Deeply nested calls = poor CPU cache and branch prediction**  
    The CPU prefers simple, predictable instruction patterns.  
    The Volcano model’s repeated `next()` calls create many small, unpredictable jumps in control flow —  
    leading to **pipeline stalls** and **lower throughput**.

# Materialization Model

Same idea with Volcano Model, but instead of return each row operator. Each operator return all tuple (materialize result) for upper operator.

This approach is better for OLTP workloads because queries typically only access a small number of tuples at a time. Thus, there are fewer function calls to retrieve tuples. The materialization model is not suited for OLAP queries with large intermediate results because the DBMS may have to spill those results to disk between operators.

# Vectorization Model

This model was introduced 2005 in paper titled “MonetDB/X100: Hyper-Pipelining Query Execution”.
Detail in this page: [[Paper]]


# Techniques

Terminology:

Tight loop execution: Instructions without branching. CPU can execute sequentially.
CPU Cache:
- To avoid waiting hundreds of cycles every time CPU needs data (load from DRAM to register), CPU will prefe


## RowSet Iteration
