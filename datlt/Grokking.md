# Evolution of Database query engine

In the early stages, I/O limitations overshadowed computation time for queries. Query performance primarily hinged on the execution **plan selected by the query optimizer.**
**Nowadays, with advancements in computer hardware, the query scheduler and plan executor are gaining significant prominence.**

## Classical Volcano Engine

row-based streaming iterator model
