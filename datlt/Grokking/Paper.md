MonetDB/X100: Hyper-Pipelining Query Execution

# Background

Prior to the vectorized engine era, the volcano model was widely adopted in industry. In the early years of database technologies, when data I/O was slow and memory and CPU resources were expensive, database experts developed the classic volcano model, which allows an SQL engine to compute one data row at a time to avoid memory exhaustion.

[[Grokking#Disadvantages|Some disadvantages of volcano model]]


