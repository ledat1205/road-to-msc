Architecture


## Cluster tuning
The hierarchy in one sentence per layer:

**Cluster** — the whole company. All machines combined.

**Node** — one physical machine. Its specs (16 cores, 64 GB) are fixed hardware limits you cannot change in software.

**Executor** — one JVM process renting space on a node. A node runs 3 executors the same way one office building houses 3 teams. Each executor is completely isolated — separate heap, separate GC, separate crash boundary. This isolation is why you want multiple executors per node instead of one giant one.

**Core** — one thread slot inside an executor. It is not a physical CPU core exclusively — it is a concurrency slot. One core = one task at a time. 5 cores = 5 tasks running simultaneously inside that executor.

**Task** — one unit of work. It lives on one core, processes one partition, runs one stage's logic (filter, join, aggregate), and dies when done. The next task on that core immediately starts.
![[Pasted image 20260320012459.png]]
![[Pasted image 20260320010007.png]]
![[Pasted image 20260320010121.png]]
![[Pasted image 20260320010342.png]]
![[Pasted image 20260320010546.png]]
![[Pasted image 20260320010811.png]]
![[Pasted image 20260320010953.png]]
![[Pasted image 20260320011054.png]]

Optimization

![[Pasted image 20260320013326.png]]