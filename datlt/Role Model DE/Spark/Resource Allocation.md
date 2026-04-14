
# Static allocation
# Dynamic allocation

Spark uses a set of heuristics to decide when to remove and request executors

## Request policy

The request is triggered when there have been pending tasks for **spark.dynamicAllocation.schedulerBacklogTimeout** seconds, and then triggered again every **spark.dynamicAllocation.sustainedSchedulerBacklogTimeout** seconds thereafter if the queue of pending tasks persists.

![[Pasted image 20260410151016.png]]

Start with 1 executor then increase to 2, 4, 8, etc.
Reason:
- Request small number of executors at first to ensure that only a few additional executors are used if they are sufficient.
- Accelerate its resource usage if many executors are actually needed.

## Remove policy

Spark application removes an executor when idle for a predefined interval  (_**spark.dynamicAllocation.executorIdleTime**_).

![[Pasted image 20260414151414.png]]

## Graceful Decommission of executor

In static resource allocation, executor is safely discarded (exist until application complete)

In dynamic resource allocation, executor can be removed in app running process. So we need a mechanism to store data in that executor to avoid recompute

During shuffle, the executors write its map outputs locally to disk then serves as the entry point for other executors can fetch data.

Solution is use external shuffle service. This service runs as a long-running process on each cluster node, independently of Spark applications and their executors.
![[Pasted image 20260414160226.png]]

Executors also can cache data in executor memory, user can configure executors containing cache data.

# Schedule Mode
## First in first out (FIFO)

![[Pasted image 20260414160441.png]]

## Fair 
![[Pasted image 20260414160515.png]]

