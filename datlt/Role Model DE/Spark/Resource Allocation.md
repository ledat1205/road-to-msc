
# Static allocation
# Dynamic allocation

Spark uses a set of heuristics to decide when to remove and request executors

## Request policy

The request is triggered when there have been pg endintasks for **spark.dynamicAllocation.schedulerBacklogTimeout** seconds, and then triggered again every **spark.dynamicAllocation.sustainedSchedulerBacklogTimeout** seconds thereafter if the queue of pending tasks persists.

![[Pasted image 20260410151016.png]]

