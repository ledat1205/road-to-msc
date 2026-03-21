 

### Here's everything you need to know


[Phi Vu Trinh](https://substack.com/@vutr)

Jun 10, 2025

∙ Paid

> To celebrate Lunar New Year (the true New Year holiday in Vietnam), I’m offering _**50% off the annual subscription**_. The offer ends soon; grab it now to get full access to nearly 200 high-quality data engineering articles.
> 
> [50% off the annual subscription](https://vutr.substack.com/subscribe)

[

![](https://substackcdn.com/image/fetch/$s_!ddQ_!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F0ab2a43e-775b-4d72-82b4-378b8666f219_2000x1429.png)



](https://substackcdn.com/image/fetch/$s_!ddQ_!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F0ab2a43e-775b-4d72-82b4-378b8666f219_2000x1429.png)

---

## Intro

In 2009, UC Berkeley’s AMPLab developed Spark.

At that time, MapReduce was the go-to choice for processing massive datasets across multiple machines. AMPLab observed that cluster computing had significant potential.

However, MapReduce made building large applications inefficient, especially for machine learning (ML) tasks requiring multiple data passes.

For example, the ML algorithm might need to make many passes over the data. With MapReduce, each pass must be written as a separate job and launched individually on the cluster.

They created Spark. Unlike MapReduce, which writes data to disks after every task, Spark relies on memory processing.

With a more friendly API, supporting wide use cases, and especially efficient in-memory processing, Spark has gained increasing attention and become the dominant solution in data processing.

But, do you know how Spark manages the memory?

This week, I will try to answer this question in the following text. We will revisit some Spark basics before diving into Spark’s memory management.

## **A Spark Application**

A typical Spark application consists of:

[

![](https://substackcdn.com/image/fetch/$s_!cE0u!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F9132b157-0299-4804-b04d-e560cc938586_624x526.png)



](https://substackcdn.com/image/fetch/$s_!cE0u!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F9132b157-0299-4804-b04d-e560cc938586_624x526.png)

- **Driver:** This JVM process manages the Spark application, handling user input and distributing work to the executors.
    
- **Executors:** These JVM processes execute tasks the driver assigns and report their status and results. Each Spark application has a set of isolated executors. A single physical worker node can have multiple executors.
    
- **Cluster Manager:** This component manages the cluster of machines running the Spark application. Spark can work with different managers, including YARN or Spark’s standalone manager.
    

> _**Note**: You might find some confusion here. The cluster manager will have its own “master” and “worker” abstractions. The main difference is that these are tied to physical machines rather than Spark processes._

When the cluster manager receives the Spark application, the manager places the driver process in one of the worker nodes. Next, the SparkSession from the application code asks the cluster manager for resources. If things go well, the manager launches the executor processes and sends the relevant information about their locations to the driver. The driver then forms the plan and starts scheduling tasks for executors.

## **Jobs, Stages, Tasks**

[

![](https://substackcdn.com/image/fetch/$s_!ZUba!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fa8a593a9-7308-4d0c-b262-bfe476c28509_854x430.png)



](https://substackcdn.com/image/fetch/$s_!ZUba!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fa8a593a9-7308-4d0c-b262-bfe476c28509_854x430.png)

From the previous section, we know that an executor will handle tasks from the driver. To understand the task concept, we must learn more about other things:

- **Job:** A job represents a series of transformations applied to data. It encompasses the entire workflow from start to finish.
    
- **Stage:** A stage is a job segment executed without data shuffling. A job is split into different stages when a transformation requires shuffling data across partitions.
    
- **DAG:** In Spark, RDD dependencies are used to build a Directed Acyclic Graph (DAG) of stages for a Spark job. The DAG ensures that stages are scheduled in topological order.
    
- **Task:** A task is the smallest unit of execution within Spark. Each stage is divided into multiple tasks, which execute processing in parallel across different data partitions.
    

So, the driver will do [something](https://vutr.substack.com/p/i-spent-8-hours-learning-apache-spark) to break down the Spark job into executable tasks for the executor. Spark was designed to run tasks in parallel; an executor can handle multiple tasks at the same time. Here’s where the “in-memory processing“ happens.

## The memory management

> _This article only discusses the executor’s memory management._

As mentioned above, when submitting the Spark application, the cluster manager initiates the driver and executors, which are JVM processes. The user can specify the resource for these processes.

- The driver resource: `spark.driver.cores`, `spark.driver.memory`,…
    
- The executor resource: `spark.executor.cores`, `spark.executor.memory`, …
    

The executor has three main regions for memory: on-heap, off-heap, and overhead.

[

![](https://substackcdn.com/image/fetch/$s_!ctVP!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F14950950-df69-4b22-a409-2236aab3dbe5_498x414.png)



](https://substackcdn.com/image/fetch/$s_!ctVP!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F14950950-df69-4b22-a409-2236aab3dbe5_498x414.png)

Let’s start with the on-heap region.

### **On Heap**

> To celebrate Lunar New Year (the true New Year holiday in Vietnam), I’m offering _**50% off the annual subscription**_. The offer ends soon; grab it now to get full access to nearly 200 high-quality data engineering articles.
> 
> [50% off the annual subscription](https://vutr.substack.com/subscribe)

This is the region we would take care of most of the time when running Spark.

[

![](https://substackcdn.com/image/fetch/$s_!y-Dq!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F68edac27-7b64-4fbf-9756-46c78f7e659b_478x254.png)



](https://substackcdn.com/image/fetch/$s_!y-Dq!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F68edac27-7b64-4fbf-9756-46c78f7e659b_478x254.png)

The `spark.executor.memory` specifies the amount of JVM heap memory each executor can have. Most things happen here.

This region has several parts.

#### **The reserved memory**

[

![](https://substackcdn.com/image/fetch/$s_!r4ld!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F5959fea9-3be0-4119-9dd4-6073ec622aad_1142x370.png)



](https://substackcdn.com/image/fetch/$s_!r4ld!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F5959fea9-3be0-4119-9dd4-6073ec622aad_1142x370.png)

Screenshot from the [Apache Spark GitHub repo, file UnifiedMemoryManager.scala](https://github.com/apache/spark/blob/a9bfacb084e696265a9d1473efe5001d03700ee3/core/src/main/scala/org/apache/spark/memory/UnifiedMemoryManager.scala#L200).

Spark uses this region to store internal objects. Users can’t specify the amount of memory in this region. It is [hardcoded and equal to 300 MB](https://github.com/apache/spark/blob/a9bfacb084e696265a9d1473efe5001d03700ee3/core/src/main/scala/org/apache/spark/memory/UnifiedMemoryManager.scala#L200).

#### **The user memory**

This region provides some memory for user data structures (e.g., your hash tables or arrays) and Spark’s internal metadata and safeguards [against OOM errors in special cases](https://spark.apache.org/docs/latest/tuning.html#memory-management-overview).

The formula specifies it:

[

![](https://substackcdn.com/image/fetch/$s_!vHGU!,w_2400,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Faddedab0-1e0e-45c6-b4b7-91a9075072db_1234x208.png)



](https://substackcdn.com/image/fetch/$s_!vHGU!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Faddedab0-1e0e-45c6-b4b7-91a9075072db_1234x208.png)

#### **The unified memory**

You might wonder what the `spark.memory.fraction` used for. Spark uses this setting to specify the amount of the unified memory.

[

![](https://substackcdn.com/image/fetch/$s_!P8Oy!,w_2400,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F9c9da8a9-32b8-4466-ab47-99f103df5c59_1124x198.png)



](https://substackcdn.com/image/fetch/$s_!P8Oy!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F9c9da8a9-32b8-4466-ab47-99f103df5c59_1124x198.png)

The unified memory has two regions: execution and storage.

[

![](https://substackcdn.com/image/fetch/$s_!EtXi!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F98073367-28a6-46ea-9d87-f9da942b6d46_416x258.png)



](https://substackcdn.com/image/fetch/$s_!EtXi!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F98073367-28a6-46ea-9d87-f9da942b6d46_416x258.png)

The **execution** region is used for shuffling, joins, aggregations, and sorting. The memory is released as soon as the task completes.

[

![](https://substackcdn.com/image/fetch/$s_!s2pe!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fa6c9b9a2-906e-4bae-b986-dc5c3bcbb89d_842x454.png)



](https://substackcdn.com/image/fetch/$s_!s2pe!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fa6c9b9a2-906e-4bae-b986-dc5c3bcbb89d_842x454.png)

For the **storage,** it is used for [data caching](https://luminousmen.com/post/explaining-the-mechanics-of-spark-caching) and data broadcasting.

[

![](https://substackcdn.com/image/fetch/$s_!uYfR!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fc60c5b42-cd70-4378-9e45-26b637baf996_524x390.png)



](https://substackcdn.com/image/fetch/$s_!uYfR!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fc60c5b42-cd70-4378-9e45-26b637baf996_524x390.png)

> _There is a join technique called the broadcast join, which involves a large dataset and a much smaller one. Instead of shuffling the small dataset across the cluster, it is broadcast to all the workers. This reduces network overhead and speeds up the join operation. Broadcasting is especially effective when the small dataset fits entirely in memory._

The amount of **storage** region is specified by the `spark.memory.storageFraction`.

Giving the executor 4 GB of memory, the fraction is 0.6 (by default), and the storage fraction is 0.5 (by default), we can calculate the amount of Heap memory used for storage:

[

![](https://substackcdn.com/image/fetch/$s_!TyHo!,w_2400,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F2b5cf3dc-0ce0-47ba-a2c5-ca4305e9ae61_1744x324.png)



](https://substackcdn.com/image/fetch/$s_!TyHo!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F2b5cf3dc-0ce0-47ba-a2c5-ca4305e9ae61_1744x324.png)

The rest of the unified memory is for execution:

[

![](https://substackcdn.com/image/fetch/$s_!69zj!,w_2400,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F1dc52197-9b5c-4e1b-bcdd-269f324068ef_1882x316.png)



](https://substackcdn.com/image/fetch/$s_!69zj!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F1dc52197-9b5c-4e1b-bcdd-269f324068ef_1882x316.png)

The `spark.memory.storageFraction` defines the boundary between storage and execution. Initially, this boundary is fixed; storage can’t use space from execution, and vice versa.

[

![](https://substackcdn.com/image/fetch/$s_!e9UY!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F87d9fc8f-be5a-40c5-92f6-75f6cbc4b70f_632x310.png)



](https://substackcdn.com/image/fetch/$s_!e9UY!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F87d9fc8f-be5a-40c5-92f6-75f6cbc4b70f_632x310.png)

Since Spark 1.6, the boundary between the execution and storage regions is crossable.

[

![](https://substackcdn.com/image/fetch/$s_!A3XD!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fe52a7382-c759-46ff-9f97-a2e9076b11f6_610x246.png)



](https://substackcdn.com/image/fetch/$s_!A3XD!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fe52a7382-c759-46ff-9f97-a2e9076b11f6_610x246.png)

The executor can use a part of the execution region for storage and vice versa. [The motivations behind this design are](https://issues.apache.org/jira/secure/attachment/12765646/unified-memory-management-spark-10000.pdf):

- Tuning the fractions requires expertise in Spark internals.
    
- The fixed fraction setting is not suitable for all workloads.
    
- With applications that do not cache much data, the storage regions are wasted.
    

Essentially, the goal of the unified approach is to help the executor leverage resources more efficiently. However, this does not mean each one can take up the memory of the other for as long/as much as they want. There are rules.

The storage region can borrow free execution space. If the execution needs to get the space back, it reclaims it, which causes **cached data from the storage to be evicted**. The data eviction happens only when the total storage memory usage falls under the R threshold, which is expressed by the `storageFraction`

In another direction, the execution can borrow free storage memory. However, the extra space the **execution borrows is never evicted by the storage. Even though the execution initially borrowed the space that crossed the R threshold, it won’t be evicted.**

This is because the data used in intermediate computations is very important; losing it causes the whole process to fail.

Let’s take some examples to understand the unified design:

- When the storage borrows memory from the execution: If there is free space in the execution, the storage can borrow it. When execution needs the memory back, the storage is forced to evict data using the Least Recently Used (LRU) policy. The eviction happens until the storage space falls under the R threshold.
    
    [
    
    ![](https://substackcdn.com/image/fetch/$s_!zLvK!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F22030392-464c-410e-9b89-1cd1c0b5b31a_762x624.png)
    
    
    
    ](https://substackcdn.com/image/fetch/$s_!zLvK!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F22030392-464c-410e-9b89-1cd1c0b5b31a_762x624.png)
    
- The execution borrows memory from the storage: If there is free space in the storage, the execution can borrow it. When storage needs to take the space back, it simply can’t because the design prioritizes the execution. When new data needs to be cached, the storage is forced to evict data using the Least Recently Used (LRU) policy to make room for the new data in the remaining storage region. The R threshold only protected the data in the storage from being evicted when the execution did not initially cross the threshold.
    
    [
    
    ![](https://substackcdn.com/image/fetch/$s_!Cm04!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F00370e08-8131-489b-98c7-a9a75df1f262_712x596.png)
    
    
    
    ](https://substackcdn.com/image/fetch/$s_!Cm04!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F00370e08-8131-489b-98c7-a9a75df1f262_712x596.png)
    

This design ensures several things.

First, applications not using caching can use the entire space for execution, limiting data from being spilled to disk.

Second, if an application leverages caching, it still has a minimum storage space where the data won’t be evicted.

Third, this approach helps Spark adapt reasonably to various use cases without requiring users to care much about the memory internals.

### **Off heap**

The on-heap data is subject to the JVM garbage collection (GC) process. The defined objects are automatically cleaned up when they are no longer used. This is convenient but can sometimes cause overhead, as the GC process requires the current process to be put on hold until the GC finishes.

In addition, JVM’s object has a significant memory overhead. [A 4-byte string would have over 48 bytes in the JVM object.](https://www.databricks.com/blog/2015/04/28/project-tungsten-bringing-spark-closer-to-bare-metal.html)

To address the GC inefficiency and JVM object overhead, [the project Tungsten](https://www.databricks.com/blog/2015/04/28/project-tungsten-bringing-spark-closer-to-bare-metal.html) introduces a memory manager that operates directly against binary data rather than Java objects.

[Instead of relying on the JVM](https://www.youtube.com/watch?v=5ajs8EIPWGI), Tungsten manages objects directly by representing them as specialized Spark SQL Types objects. This can be highly efficient and friendly to the GC process. Even operating on heap memory, these advantages make memory management more performant.

[

![](https://substackcdn.com/image/fetch/$s_!y_36!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fd342a63b-dc3c-4f56-af3e-1061978df5bd_538x214.png)



](https://substackcdn.com/image/fetch/$s_!y_36!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fd342a63b-dc3c-4f56-af3e-1061978df5bd_538x214.png)

Tungsen can work with the off-heap mode, which directly manages data outside the JVM. The off-heap memory is turned off by default, but can be enabled by setting `spark.memory.offHeap.enabled` to True and specifying the `spark.memory.offHeap.size` to have a positive value.

Compared to the heap memory, the off-heap one has only two regions: the execution and the storage, and it is also subjected to the `spark.memory.storageFraction.`

Enabling the off-heap will not impact the existing on-heap memory. The total execution region is the sum of the on-heap and off-heap execution regions; the same is true for the storage region.

[

![](https://substackcdn.com/image/fetch/$s_!nOjd!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fc29cbf9b-abcb-42fa-bca1-bcabf6a5604a_792x278.png)



](https://substackcdn.com/image/fetch/$s_!nOjd!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fc29cbf9b-abcb-42fa-bca1-bcabf6a5604a_792x278.png)

### The overhead memory

> `spark.executor.memoryOverhead`

[This](https://spark.apache.org/docs/latest/configuration.html#:~:text=2.4.0-,spark.executor.memoryOverhead,-executorMemory%20*%20spark) is the amount of additional memory to be allocated per executor process. It stores things like interned strings, VM overheads, other native overheads, etc. The overhead region is not included in the `spark.executor.memory`. Its default minimum is 384 MB, and can grow with the executor’s memory (with the default 10% of the `spark.executor.memoryOverheadFactor`)

This overhead memory is also used for the PySpark executor’s memory if the `spark.executor.pyspark.memory` is not specified.

---

## Outro

We go through Spark basics with the typical architecture and the anatomy of the Spark jobs, stages, and tasks. Then, we explore execution memory management, mainly focusing on on-heap memory and unified memory management.

Thank you for reading this far. See you in my next articles.

---

## Reference

_[1] [Tuning Spark Document](https://spark.apache.org/docs/latest/tuning.html#memory-management-overview)_

_[2] [Spark Configuration](https://spark.apache.org/docs/latest/configuration.html)_

_[3] [Spark Github Repo](https://github.com/apache/spark/tree/master)_

_[4] Luminousmen, [Deep Dive into Spark Memory Management](https://luminousmen.com/post/dive-into-spark-memory) (2024)_

_[5] Holden Karau; Rachel Warren, [High Performance Spark](https://www.oreilly.com/library/view/high-performance-spark/9781491943199/) (2017)_

_[6] Alexey Grishchenko, [Spark Memory Management](https://0x0fff.com/spark-memory-management/)_
