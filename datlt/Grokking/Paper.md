MonetDB/X100: Hyper-Pipelining Query Execution

# Background

Prior to the vectorized engine era, the volcano model was widely adopted in industry. In the early years of database technologies, when data I/O was slow and memory and CPU resources were expensive, database experts developed the classic volcano model, which allows an SQL engine to compute one data row at a time to avoid memory exhaustion.

[[Grokking#Disadvantages|Some disadvantages of volcano model]]

To prove about opinion volcano doesn't use CPU resource efficiently. The paper “DBMSs On A Modern Processor: Where Does Time Go?”, the authors have minutely dissected the resource consumption of database systems in the framework of modern CPUs. 


![[Pasted image 20251017164057.png]]

They conclude the CPU time for computation is not greater than 50% in sequential scans, index-based scans, and JOIN queries. On the contrary, the CPU spends quite an amount of time (50% on average) waiting for resources, due to memory or resource stalls. Plus the cost of branch mispredictions, the percentage of CPU time for computation is often far less than 50%. For example, the minimum percentage of CPU time for computation in index-based scans is less than 20%.

Note: detail experiment setup can see on paper 