
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



