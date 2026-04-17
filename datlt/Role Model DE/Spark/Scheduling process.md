
# The scheduling process

## SparkContext

When we submit a Spark application, the SparkContext is first initialized

The SparkContext then initializes the TaskScheduler (for task-oriented scheduling) and the SchedulerBackend (which interacts with the cluster manager and provides resources to the TaskScheduler). After that, the DAGScheduler (for stage-oriented scheduling) is created.

![[Pasted image 20260416172404.png]]

## DagScheduler
