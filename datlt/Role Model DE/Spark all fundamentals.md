
# Spark RDD

Spark relies on in-memory processing. Resilient Distributed Dataset (RDD) abstraction to manage Spark's data in memory. All other abstraction from dataset and dataframe, they are complied into RDDs behind the scenes

RDD represents an **immutable, partitioned collection of records** that can be operated on in parallel. Data inside RDD is stored in memory for as long as possible.

Why immutable:
- concurrent processing: not allow update can avoid complex synchronization, race condions
- lineage and fault tolerance: each transformation creates a new RDD, preserving the lineage and allowing spark recompute lost data reliably
- functional programming

## Properties 

![[Pasted image 20260326160816.png]]

**List of partitions**: An RDD is divided into partitions, Spark's parallelism units. Each partition is a logical data subset and can be processed independently with different executors

**Computation Function:** A function determines how to compute the data for each partition.

**Dependencies:** The RDD tracks its dependencies on other RDDs, which describe how it was created.

**Partitioner (Optional):** For key-value RDDs, a partitioner specifies how the data is partitioned, such as using a hash partitioner.

**Preferred Locations (Optional):** This property lists the preferred locations for computing each partition, such as the data block locations in the HDFS.

## Lazy

Data is unavailable or transformed immediately until an action or triggers the execution. This approach allows Spark to determine the most efficient way to execute the transformations.

**Transformations**: define how data should be transformed (dont execute)
**Actions**: produce output and store data

## Fault Tolerance

Follow lineage to keep track RDDs dependencies and reconstruct on original RDD due to issues.

# Architecture
![[Pasted image 20260327114753.png]]

- **Driver**: JVM process manage spark app, handling user input to distribute to executor
- **Cluster manager**: manage cluster running spark app
- **Executors**: these processes execute tasks driver assigns and report their status and result 

# Mode

- **Cluster Mode**: the driver process is launched on a worker node alongside the executor. The cluster manager handles all processes related to the Spark application
- ![[Pasted image 20260327161229.png]]

- **Client Mode**: the driver remains on the client machine that submitted the application. The client machine have to maintain driver process throughout the application
![[Pasted image 20260327161241.png]]

- **Local Mode**: run entire spark on single machine, achieving parallelism through multiple threads.
![[Pasted image 20260327161248.png]]

# Detail workload

![[Pasted image 20260327161349.png]]

- **Job**: a series of transformation steps 
- **Stage**: a stage is a job segment executed without data shuffling. a job is split into different stages when a transformation requires shuffling data across partitions
- **DAG**: RDD dependencies are used to build a DAG of stages for a spark job
- **Task**: smallest unit of execution. Stage is divided into multiple tasks, which execute processing in parallel across different partitions 

![[Pasted image 20260327162957.png]]
Narrow transformation: Partitions depend on a single parent or specific subset of parent partitions known beforehand (eg: map, coalesce)
Wide transformation: A single partition of a parent RDD contributes to multiple partitions of child RDD which involve shuffle data across partitions. (eg: groupByKey, reduceByKey, join )


# Journey
![[Pasted image 20260327163758.png]]

- user defines spark application. It must include SparkSession object 
- submits a spark application to cluster manager and also requests driver resource
- cluster manager accepts to submission and places driver to a worker node
- the driver asks cluster manager to launch the executors. 
- the cluster manager launches the executor processes and sends the information about their locations to the driver process
- the driver formulates an execution plan to guide the physical execution. This process starts with the logical plan, which outlines intended transformations 
- it generates the physical plan through several refinement steps, specifying the detailed execution strategy for processing the data.
- The driver starts scheduling tasks on executors, and each executor responds to the driver with the status of those tasks.
- Once the application finishes, the driver exits with either success or failure. The cluster manager then shuts down the application’s executors.
- The client can check the status of the Spark application by asking the cluster manager.

# Plan
Catalyst Optimizer designed based on functional programming construct in scala. support rule-based and cost-based 
- rule-based optimization: relies on predefined rules and heuristics to choose the execution plan for query 
- cost-based optimization: uses statistical information about data - table size, index selectivity and data distribution to estimate various execution plans and chooses lowest estimated cost

logic go through an optimized process: analyze logical plan, optimize logical plan, physical plan and code generation.
![[Pasted image 20260329164659.png]]

- **Analysis:** The optimizer uses the rules and the catalog to answer questions like “Is the column/table name valid?” or “What is the column’s type?”.

> _The Catalog object enables interaction with metadata for databases, tables, and functions. It allows users to list, retrieve, and manage these entities and refresh table metadata to keep Spark's view in sync with underlying data sources._

- **Logical Optimization:** Spark applies standard rule-based optimizations, such as predicate pushdown, projection pruning, null propagation, etc.
- **Physical Planning:** Based on the logical plan, the optimizer generates one or more physical plans and selects the final one using a cost model.
- **Code Generation:** The final query optimization phase generates Java bytecode for execution.

cost model maybe access to outdated or unavailable statistics

Apache Spark 3 introduce Adaptive Query Execution to solve this problem. This allow query plans to be adjusted based on runtime statistics collected during execution. 

The next stage based on previous stage is complete. So data statistics can collect on previous stage and use for the following stage

# Scheduling process

process assign tasks to executors
![[Pasted image 20260329175804.png]]

- The DAGScheduler for stage-oriented scheduling
- The TaskScheduler for task-oriented scheduling
- The SchedulerBackend interacts with the cluster manager and provides resources to the TaskScheduler.

DAGScheduler creates a TaskSet for each stage and sends TaskSet to TaskScheduler. The DAGScheduler also determines the preferred locations for each task based on cache status and sends to TaskScheduler

TaskScheduler is responsible for scheduling tasks from TaskSet on available executors. It requests resources from TheschedulerBackend 

The SchedulerBackend requests executors from the cluster manager, which then launches executors based on the application's requirements. Once started, the executors attempt to register with the SchedulerBackend through an RPC endpoint. If successful, the SchedulerBackend receives a list of the application's desired executors


![[Pasted image 20260329180102.png]]


![[Pasted image 20260329180231.png]]

When the TaskScheduler requests resources, the SchedulerBackend informs the TaskScheduler about the available resources on the executors.

The TaskScheduler assigns tasks to these resources, resulting in a list of task descriptions. For each entry in this list, the SchedulerBackend serializes the task description and sends it to the executor.

The executor deserializes the task description and begins launching the task.

# Scheduling Mode

How a job will be scheduled at the task level. Cluster has two or three jobs to run
2 modes:
- **First In First Out (FIFO)**: default mode, very simple idea
![[Pasted image 20260330164018.png]]

- **Fair**: Since spark 0.8, the user can configure fair scheduling between jobs. Spark assigns tasks between jobs in round-robin.
![[Pasted image 20260330164927.png]]

The fair scheduler supports grouping jobs into pools and setting various scheduling options for each pool, such as the weight. This can help isolate workload so critical jobs can be executed on a larger resource pool. The user can configure which jobs can be run on which pools.
![[Pasted image 20260330165106.png]]

# Resource allocation
a Spark application gets an isolated set of executors (JVM processes)
2 ways of allocating resources: static allocation and dynamic allocation
- Static allocation: resources for application is fixed and cannot change during runtime. User can define resource configuration
![[Pasted image 20260330173301.png]]

- Dynamic allocation (enable by setting `spark.dynamicAllocation.enabled` to `True`): Since version 1.2, Spark offers dynamic resource allocation. The application may return resources to the cluster if they are no longer used and can request them later when there is demand.
![[Pasted image 20260330173410.png]]

# Memory management

The executor has three central regions for memory: on-heap, off-heap and overhead
![[Pasted image 20260330173511.png]]

## On heap
use the `spark.executor.memory` setting to specify each executor's on-heap memory

### The reserved memory
Spark uses this region to store internal objects. It is [hardcoded and equal to 300 MB](https://github.com/apache/spark/blob/a9bfacb084e696265a9d1473efe5001d03700ee3/core/src/main/scala/org/apache/spark/memory/UnifiedMemoryManager.scala#L200).

### The user memory
![[Pasted image 20260330174122.png]]

memory for user data structures and spark's internal metadata and safeguards 

### The unified memory
![[Pasted image 20260331171605.png]]

This region is specified by the setting `spark.memory.fraction`. The unified memory includes two parts: the execution and the storage region.

![[Pasted image 20260331171619.png]]

- The **execution** region is used for shuffling, joins, aggregations, and sorting. The memory is released as soon as the task completes.
- For **storage,** it is used for [data caching](https://luminousmen.com/post/explaining-the-mechanics-of-spark-caching)

The boundary between execution and storage is defined by the `spark.memory.storageFraction`

Example: 
With 4 GBs mem, the `spark.memory.fraction` is 0.6 (default),  `spark.memory.storageFraction` is 0.5 (default)
![[Pasted image 20260331171831.png]]

![[Pasted image 20260331171836.png]]

from spark >= 1.6, execution can borrow memory if it need
![[Pasted image 20260331171920.png]]

Motivation: 
- Tuning the fractions requires expertise in Spark internals.
- The fixed fraction setting is not suitable for all workloads.
- With applications that do not cache much data, the storage regions are wasted.

There are rules for borrow memory:
![[Pasted image 20260331172110.png]]

If there is free space in the execution, storage can borrow it. When execution needs the memory back, the storage is forced to evict data using the Least Recently Used (LRU) until storage space under R threshold 

![[Pasted image 20260331172307.png]]

If there is free space in the storage, the execution can borrow it. However, when storage needs to take the space back, it can’t because the design prioritizes the execution. When new data needs to be cached, the storage is forced to evict data using the Least Recently Used (LRU) policy to make room for the new data in the remaining storage region.

## Off heap
The on-heap data is subject to the JVM garbage collection (GC) process. Sometimes GC is the problem, as GC process pauses the current process until the GC finishes 

In addition, JVM's object has a significant memory overhead (wasted resource)

 [project Tungsten](https://www.databricks.com/blog/2015/04/28/project-tungsten-bringing-spark-closer-to-bare-metal.html) introduces a memory manager that operates directly against binary data rather than Java objects to address above problems
 ![[Pasted image 20260402114729.png]]

Tungsen can work with the off-heap memory, which directly manages data outside the JVM. The off-heap memory is turned off by default, but can be enabled by setting `spark.memory.offHeap.enabled` to True and specifying the `spark.memory.offHeap.size` to have a positive value.
![[Pasted image 20260402114837.png]]

off-heap only has 2 memory regions: execution and storage
 The total execution region is the sum of the on-heap and off-heap execution regions; the same is true for the storage region.

# Cache
cache like transformation, cache is lazy

![[Pasted image 20260402142802.png]]

- MEMORY_ONLY: Spark stores unserialized cache data in memory 
- MEOORY_AND_DISK: Spark stores unserialized cache data in memory. If memory is full, Spark writes the data to disk 
- DISK_ONLY: Data is cached on disk only in a serialized format
- OFF_HEAP: Spark stores cached data in the off-heap region

Use suffix `_SER` for each option to cache serialized data. Save storage but adds m