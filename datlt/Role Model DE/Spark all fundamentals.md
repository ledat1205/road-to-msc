
# Spark RDD

Spark relies on in-memory processing. Resilient Distributed Dataset (RDD) abstraction to manage Spark's data in memory. All other abstraction from dataset and dataframe, they are complied into RDDs behind the scenes

RDD represents an **immutable, partitioned collection of records** that can be operated on in parallel. Data inside RDD is stored in memory for as long as possible.

Why immutable:
- concurrent processing: not allow update can avoid complex synchronization, race condions
- lineage and fault tolerance: each transformation creates a new RDD, preserving the lineage and allowing spark recompute lost data reliably
- functional programming

## Properties 

**List of partitions**: An RDD is divided into partitions, Spark's parallelism units. Each partition is a logical data subset and can be processed independently with different executors

