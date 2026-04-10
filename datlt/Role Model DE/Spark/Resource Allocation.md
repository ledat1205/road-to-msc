
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

