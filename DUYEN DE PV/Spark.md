![[spark_cheatsheet (1).html]]
## Debugging Spark
## Concepts 
https://vutr.substack.com/p/if-youre-learning-apache-spark-this?utm_source=publication-search - Everything you need to know
https://vutr.substack.com/p/i-spent-8-hours-learning-apache-spark?utm_source=publication-search - On Scheduling
https://vutr.substack.com/p/i-spent-6-hours-to-learn-how-apache?utm_source=publication-search - On Execution Plan

## K8S
https://vutr.substack.com/p/lets-learn-kubernetes-by-running?utm_source=publication-search

## PySpark
https://vutr.substack.com/p/i-spent-6-hours-learning-pyspark?utm_source=publication-search

## Spark Structured Streaming
https://vutr.substack.com/p/everything-you-need-to-know-about-46d?utm_source=publication-search

**Cluster** — the whole company. All machines combined.

**Node** — one physical machine. Its specs (16 cores, 64 GB) are fixed hardware limits you cannot change in software.

**Executor** — one JVM process renting space on a node. A node runs 3 executors the same way one office building houses 3 teams. Each executor is completely isolated — separate heap, separate GC, separate crash boundary. This isolation is why you want multiple executors per node instead of one giant one.

**Core** — one thread slot inside an executor. It is not a physical CPU core exclusively — it is a concurrency slot. One core = one task at a time. 5 cores = 5 tasks running simultaneously inside that executor.

**Task** — one unit of work. It lives on one core, processes one partition, runs one stage's logic (filter, join, aggregate), and dies when done. The next task on that core immediately starts.

## Spark Shuffle
https://vutr.substack.com/p/a-9-minute-simple-explanation-of?utm_source=publication-search

## Cluster Tuning
https://vutr.substack.com/p/i-spent-4-hours-learning-apache-spark?utm_source=publication-search - Resource allocation
https://vutr.substack.com/p/i-spent-8-hours-understanding-apache?utm_source=publication-search - Memory management

![[Pasted image 20260320012459.png]]
![[Pasted image 20260320010007.png]]
![[Pasted image 20260320010121.png]]
![[Pasted image 20260320010342.png]]
![[Pasted image 20260320010546.png]]
![[Pasted image 20260320010811.png]]
![[Pasted image 20260320010953.png]]
![[Pasted image 20260320011054.png]]

## Optimization

![[Pasted image 20260320013326.png]]![[Pasted image 20260320013418.png]]
![[Pasted image 20260320013447.png]]![[Pasted image 20260320013517.png]]

![[Pasted image 20260320013549.png]]
![[Pasted image 20260320013605.png]]
![[Pasted image 20260320013616.png]]

![[Pasted image 20260320013632.png]]