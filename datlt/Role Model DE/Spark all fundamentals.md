
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