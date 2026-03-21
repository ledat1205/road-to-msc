
  
Since its first release, Apache Spark has established itself as a leader in data processing.

It’s fast

It has a whole ecosystem (batch, stream, ML, …)

It has many configuration knobs you can tune

And it provides a wide range of interfaces: SQL, Python, Scala, Java, R…

Most people believe Spark efficiency comes from in-memory processing. That’s true, but not enough, as Spark's performance is influenced by many factors, including query planning, lazy and adaptive execution, memory management, cache management, and, especially, shuffle-based (MapReduce-style) processing.

In this week's article, I tried my best to dive into this aspect of Spark: what shuffle is, when it happens, the details of the shuffle process behind the scenes, and how we can optimize it.

---

## Narrow vs Wide transformation

Spark introduced the RDD abstraction to manage data in memory. All other abstractions, such as DataFrame to Dataset, are compiled into RDDs behind the scenes. Each RDD has a list of partitions, each of which is a subset of your data.

When you define an RDD, its data is unavailable or is transformed immediately until an **action** triggers execution. This approach allows Spark to determine the most efficient way to execute the transformations:

[

![](https://substackcdn.com/image/fetch/$s_!z37r!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F39ca1705-4f3d-46ed-8c77-2c3ff6962d11_998x554.png)



](https://substackcdn.com/image/fetch/$s_!z37r!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F39ca1705-4f3d-46ed-8c77-2c3ff6962d11_998x554.png)

- **Transformations**, such as `map` or `filter`, define how the data should be transformed, but they don’t execute until an action forces the computation. Because RDDs are immutable, Spark creates a new RDD after a transformation.
    
- **Actions** are the commands that Spark runs to produce output or store data (e.g., `collect`), thereby driving the actual execution of the transformations.
    

A Spark job associated with actions is split into stages during planning. Each stage has multiple tasks, which process data partitions in parallel.

Speaking of stages, the stage boundary only appears when data shuffling appears (which is also today's main topic), and data is shuffled when there are wide-dependency transformations:

[

![](https://substackcdn.com/image/fetch/$s_!u7Uj!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F0d1bf6d8-2bcd-4fd2-b403-6fe159f3133b_972x524.png)



](https://substackcdn.com/image/fetch/$s_!u7Uj!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F0d1bf6d8-2bcd-4fd2-b403-6fe159f3133b_972x524.png)

- Transformations with **narrow dependencies** are those where each partition in the downstream RDD has a limited number of dependencies on partitions in the parent RDD. These partitions may depend on a single parent (e.g., the map operator) or on a specific subset of parent partitions known beforehand (e.g., with coalesce).
    
- Transformations with **wide dependencies** require data to be partitioned in a specific way, where a single partition of an upstream RDD contributes to multiple partitions of the child RDD. This typically occurs with operations such as groupByKey or join, which involve data shuffling.
    
    These operators require data with similar attributes (e.g., join keys or aggregation columns) to be processed on the same worker. This process is called data shuffling.
    
    [
    
    ![](https://substackcdn.com/image/fetch/$s_!5rxf!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fca3dc359-0f14-478a-bcd9-f07c2f2572a3_622x700.png)
    
    
    
    ](https://substackcdn.com/image/fetch/$s_!5rxf!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fca3dc359-0f14-478a-bcd9-f07c2f2572a3_622x700.png)
    
    Consequently, wide dependencies result in stage boundaries; tasks in the child stage create new data partitions with data with similar attributes.
    

We can understand that narrow dependency transformations can be “pipelined” and handled in a stage, reducing network communication overhead; on the other hand, wide dependency transformations are more expensive.

---

## Shuffle behind the scenes.

But why are wide dependency transformations, especially the shuffling process, expensive?

### Naming Convention

For the naming convention, I use “mappers“ to refer to the tasks that output the data in the shuffling process, and “reducers” to refer to tasks that read data from the mappers.

[

![](https://substackcdn.com/image/fetch/$s_!C280!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Ffa4ec4cd-891a-4998-9ac6-aedd6395a21a_1618x674.png)



](https://substackcdn.com/image/fetch/$s_!C280!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Ffa4ec4cd-891a-4998-9ac6-aedd6395a21a_1618x674.png)

This convention is widely used in MapReduce; the paradigm has brought the data shuffling concept closer to the community. However, as I understand it, MapReduce doesn’t invent data shuffling, since this process has long existed in DBMSs.

### Setup

Imagine we have a Spark job with two transformations: first, the data is filtered, and then it is grouped by column X to calculate the sum.

### Mappers

After planning and scheduling, we will have two Spark stages. The first filters the data, and the second groups the data by the value of column X.

In the first stage, a set of tasks runs in parallel, reading data from the source and filtering the data.

The data is first read into memory as RDDs with a given number of partitions.

[

![](https://substackcdn.com/image/fetch/$s_!J5dy!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F891fdab7-dcfe-476c-9e2a-4eb8e842481d_1022x708.png)



](https://substackcdn.com/image/fetch/$s_!J5dy!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F891fdab7-dcfe-476c-9e2a-4eb8e842481d_1022x708.png)

The number of partitions may be determined by your source data (e.g., a Parquet file), or you can determine it yourself. This number is important because it defines resource utilization and the level of parallelism. [Spark suggests 2-3 tasks per CPU core in the Spark cluster](https://spark.apache.org/docs/latest/tuning.html#level-of-parallelism).

For example, if you have 4 executors with 4 CPU cores each, the total cores will be 16, and the number of partitions (as one task is in charge of one partition) should be 32-48 (given the “spark.task.cpus” is 1, which means 1 task has 1 CPU core).

After the data is loaded into memory, the input filter logic is applied to it.

To prepare for the next stage, the shuffle process must happen to bring records with the same column x value to the same partition. Mappers (the filter tasks) will write filtered records to shuffled partitions on disks.

[

![](https://substackcdn.com/image/fetch/$s_!PDZY!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F6672bb53-4a7c-4b9d-aca0-2dde16e28459_690x596.png)



](https://substackcdn.com/image/fetch/$s_!PDZY!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F6672bb53-4a7c-4b9d-aca0-2dde16e28459_690x596.png)

Yeah, you heard it right: to disk, not memory, as people often misunderstand because Spark is famous for in-memory processing.

> _I invite you to join my paid membership list to read this writing and 150+ high-quality data engineering articles:_
> 
> [Upgrade subscription](https://vutr.substack.com/subscribe)
> 
> - _If that price isn’t affordable for you, check this [DISCOUNT](https://vutr.substack.com/subscribe?coupon=c08a9839)_
>     
> - _If you’re a student with an education email, use this [DISCOUNT](https://vutr.substack.com/subscribe?coupon=0b37c676)_
>     
> - _You can also claim this post for free (one post only)._
>     
> - _Or take the [7-day trial](https://vutr.substack.com/7d8f19f0) to get a feel for what you’ll be reading._
>     

#### Hash

Before Spark 1.2, each map task calculates the hash of each key (e.g., if you group by “user_id“, the value of column “user_id“ will be hashed) and determines which downstream reduce task (partition) it belongs to. It then writes the data to a separate, dedicated file for that specific reduce task.

[

![](https://substackcdn.com/image/fetch/$s_!tUbC!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F6e443036-2479-40af-9f3b-b4d1f6349eef_1824x842.png)



](https://substackcdn.com/image/fetch/$s_!tUbC!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F6e443036-2479-40af-9f3b-b4d1f6349eef_1824x842.png)

If there are M map tasks and R reduce tasks (R tasks are controlled by the “spark.sql.shuffle.partitions“ which is 200 in default. That means, no matter size of your data, you will always have 200 partitions after map operators), the hash shuffle manager would create MxR files.

[

![](https://substackcdn.com/image/fetch/$s_!6H4R!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F894067e1-ac24-4874-be2f-546644a1b5a2_1056x846.png)



](https://substackcdn.com/image/fetch/$s_!6H4R!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F894067e1-ac24-4874-be2f-546644a1b5a2_1056x846.png)

If you have 1,000 map tasks and 1,000 reduce tasks, you will end up with 1,000,000 intermediate files, which could degrade overall performance because many I/O operations are required.

#### Sort

Since Spark 1.2.0, Spark has used “sort” as its default shuffle manager. Instead of creating separate files for each reduce task/partition, each mapper now creates a single file.

[

![](https://substackcdn.com/image/fetch/$s_!z3iZ!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F8f9fca15-9334-4fc2-aa02-3b6d7fb45d23_1274x706.png)



](https://substackcdn.com/image/fetch/$s_!z3iZ!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F8f9fca15-9334-4fc2-aa02-3b6d7fb45d23_1274x706.png)

Data in this file is sorted by partition/reducer ID; data from the same partition is written continuously. Besides the data file, there is an index file that contains the start and stop offsets of partitions in the data file. This is optimal because most disk I/O operations are sequential (which is faster than random access on disk).

However, if the number of reducer tasks is small, the hash method would be more efficient as hashing is faster than sorting, and the number of files can be kept sustainable.

[

![](https://substackcdn.com/image/fetch/$s_!lm8s!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F341148b0-09f8-4df7-ba17-c275e96135f9_864x532.png)



](https://substackcdn.com/image/fetch/$s_!lm8s!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F341148b0-09f8-4df7-ba17-c275e96135f9_864x532.png)

Thus, when the number of partitions/reducers is smaller than “spark.shuffle.sort.bypassMergeThreshold” (200 by default), Spark [will use hashing instead](https://github.com/apache/spark/blob/master/core/src/main/java/org/apache/spark/shuffle/sort/BypassMergeSortShuffleWriter.java) to write the data to separate files for each reducer and combine them to a single file later. Of course, the index file will also be created.

> _Later, project Tungsten was introduced to improve Spark’s memory and CPU efficiency. This project also introduced a new sort-based strategy. At the end of the day, it also creates a data file and an index file. However, Spark will operate directly on serialized binary data without deserializing it, which improves performance. Plus, this strategy is only enabled when [these conditions are satisfied](https://www.waitingforcode.com/apache-spark/shuffle-writers-unsafeshufflewriter/read#UnsafeShuffleWriter-when)._

The mapper will register the location of the shuffle file to the Driver.

### Reducers

When Map tasks finish, they report the location of their output files to the Driver.

When the Driver schedules Reduce tasks, it assigns them a partition. For example, Reduce Task #7 is responsible for **Partition #7**. This task must retrieve **all data blocks** labeled as "Partition 7" from **every single mapper**.

[

![](https://substackcdn.com/image/fetch/$s_!vvmP!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F657076ad-dcad-4849-aa77-effb4346c23e_1540x976.png)



](https://substackcdn.com/image/fetch/$s_!vvmP!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F657076ad-dcad-4849-aa77-effb4346c23e_1540x976.png)

Thanks to the information that the mappers register the Driver, a reducer knows the locations of the mappers’ output files. If the file lives on a remote executor, the reducer will issue TCP requests to all mappers’ shuffle services that have the desired partitions.

[

![](https://substackcdn.com/image/fetch/$s_!Wu-0!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F24f24b04-db3c-455a-8726-f93a064218b8_1434x524.png)



](https://substackcdn.com/image/fetch/$s_!Wu-0!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F24f24b04-db3c-455a-8726-f93a064218b8_1434x524.png)

When a shuffle service receives a request, it reads the offset from the index file, locates the desired reducer partition, and returns it.

[

![](https://substackcdn.com/image/fetch/$s_!3_jh!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F68b0866b-65b0-4066-951f-1fdd8b43f3ee_766x738.png)



](https://substackcdn.com/image/fetch/$s_!3_jh!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F68b0866b-65b0-4066-951f-1fdd8b43f3ee_766x738.png)

After that, the reducer pulls the data into the memory buffer and then generates an iterator to process it.

[

![](https://substackcdn.com/image/fetch/$s_!nnNA!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fba287201-a873-4921-9b23-524cb3aca6e4_1290x530.png)



](https://substackcdn.com/image/fetch/$s_!nnNA!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fba287201-a873-4921-9b23-524cb3aca6e4_1290x530.png)

The fact that the reducer has to ask many mappers for a partition’s data can incur network overhead and cause the mapper hosts to perform many disk operations.

---

_So, after learning that the principle of a shuffle process is that the mappers produce shuffle outputs, and the reducers pull them over the network. How do we optimize them? In the following sections, I present several options for addressing the shuffle._

_**Note**: In this article, I won’t dive into the external shuffle service or leveraging the push approach for exchanging shuffle data (Spark’s canonical approach is reducers pulling from mappers)_

---

## Eliminate the shuffle in join

In Spark join, data is partitioned and shuffled by the join keys to ensure that data with the same key ends up in the same partition.

After the shuffling, the task can then look up matched keys locally.

### Broadcast join

Spark also leverages broadcast hash join to optimize the join performance. The cool thing is that it can remove the shuffle process, but under a specific condition.

[

![](https://substackcdn.com/image/fetch/$s_!0rz6!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F08b88425-9d11-45d0-bc0b-3edcb51b4079_1100x642.png)



](https://substackcdn.com/image/fetch/$s_!0rz6!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F08b88425-9d11-45d0-bc0b-3edcb51b4079_1100x642.png)

Spark tries to detect whether one of the tables is small enough [based on the configurable threshold: “spark.sql.autoBroadcastJoinThreshold”](https://spark.apache.org/docs/latest/sql-performance-tuning.html#automatically-broadcasting-joins); the default is 10MB. If yes, it will execute the broadcast join.

In this scenario, the small table is broadcast to all workers executing the join. Each worker builds the hash table from this broadcast table and performs the join locally, without requiring the shuffle process. This optimization significantly reduces the overhead of network communication in the shuffle process.

### Bucket join

Spark also offers the [bucket join](https://spark.apache.org/docs/latest/sql-performance-tuning.html#storage-partition-join). As mentioned, in Spark, data is partitioned and shuffled by the join keys. If the data from both tables is physically organized into “buckets” defined by the join keys, Spark can avoid the shuffle phase when performing the joins.

[

![](https://substackcdn.com/image/fetch/$s_!Za6m!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fbb00904d-c153-43b4-8fe4-0964b14f8c6b_1498x752.png)



](https://substackcdn.com/image/fetch/$s_!Za6m!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fbb00904d-c153-43b4-8fe4-0964b14f8c6b_1498x752.png)

Bucketing is a technique that distributes data across multiple buckets based on the hash of a column value. Imagine the user_id buckets a table: all rows with the same user_id belong to the same bucket. This helps avoid a shuffle when joining.

In other words, a bucket join is when you shuffle the data during write time rather than during join time. This approach is helpful when you know how the tables are joined together beforehand. However, this might increase write time because the engine needs to organize the data to associate it with buckets.

## Reduce the shuffle data

The next strategy is to reduce the data before shuffling.

The idea is simple: if the shuffle process is expensive, we make it cheaper by giving it less data.

[

![](https://substackcdn.com/image/fetch/$s_!XXqp!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F64cb194f-05aa-431a-aeef-4f133571e4c7_1092x642.png)



](https://substackcdn.com/image/fetch/$s_!XXqp!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F64cb194f-05aa-431a-aeef-4f133571e4c7_1092x642.png)

If you can, filter the data as much as possible before shuffling.

Or, shrink the data by aggregating it.

For example, both reduceByKey() and groupByKey() are wide transformations and require shuffling, but they differ significantly in their execution and performance characteristics due to how they handle shuffling:

[

![](https://substackcdn.com/image/fetch/$s_!bl1P!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F1f61a917-83b9-4d70-a057-bb46bd1ac5a6_1354x562.png)



](https://substackcdn.com/image/fetch/$s_!bl1P!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F1f61a917-83b9-4d70-a057-bb46bd1ac5a6_1354x562.png)

- **“reduceByKey()”** performs aggregation within each partition before shuffling the data across the network. This significantly reduces the volume of data shuffled, improving performance and reducing memory usage, especially for large datasets. It requires an associative and commutative function for the reduction.
    
- **“groupByKey()”** simply groups all values by their key and transfers the entire dataset across the network without any pre-aggregation. That said, the amount of network data transfer is larger.
    

## Controlling the reducer partitions

Fundamentally, Spark is a distributed engine, so its goal is to maximize both resource utilization and parallelism. Thus, controlling the reducer partitions/tasks is one way to optimize shuffle performance. Too many reduced tasks increase scheduling/communication overhead, while a few tasks can increase memory pressure, since each task now has to handle a larger partition.

### spark.sql.shuffle.partitions

By default, you will have 200 output partitions (the default value of “spark.sql.shuffle.partitions“ configuration). That means, no matter how large the input data is, a 10GB CSV file or a 1TB of multi-Parquet files, you will end up with 200 reducers.

[

![](https://substackcdn.com/image/fetch/$s_!6HNp!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F5831284b-723d-4ed1-a1c9-2646e85ed525_1314x906.png)



](https://substackcdn.com/image/fetch/$s_!6HNp!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F5831284b-723d-4ed1-a1c9-2646e85ed525_1314x906.png)

Don’t trust the configuration in all cases. Tune this based on your data.

To calculate the shuffle partitions, we need two parameters: the size of your input data and the target partition size. For the second parameter, I can’t find it on the Spark official document (or I missed it, please comment if you see it), so I refer to the suggestion from [Louminoumen (he’s a Spark expert): 100-200MB per partition.](https://luminousmen.substack.com/p/spark-partitions?utm_source=publication-search) We can start with that and tune the partition size to meet your cost and performance requirements later.

Now you can calculate the number of partitions by:

```
number of partitions = the size of your data / target partition size.
```

Another hint is that [Spark suggests 2-3 tasks per CPU core in the Spark cluster](https://spark.apache.org/docs/latest/tuning.html#level-of-parallelism). Because the executor is kept alive during the application, launching a new task on it incurs low overhead.

For example, if you need 150 partitions, you might only need 50-75 cores

### Adaptive Query Execution (AQE)

In Apache Spark 3, released in 2020, Adaptive Query Execution (AQE) was introduced to adjust query plans, including the number of partitions, based on runtime statistics collected during execution.

- **Dynamically coalescing shuffle partitions**: The user can start with a relatively large number of shuffle partitions, and then AQE can help combine smaller adjacent partitions at runtime by analyzing shuffle file statistics. This helps balance partition sizes and improve query performance.
    
    [
    
    ![](https://substackcdn.com/image/fetch/$s_!4ex8!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F66de87a1-76a5-4ec1-8499-a61b586835d8_1456x922.png)
    
    
    
    ](https://substackcdn.com/image/fetch/$s_!4ex8!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F66de87a1-76a5-4ec1-8499-a61b586835d8_1456x922.png)
    
- **Dynamically optimizing skew joins**: If your data is skewed, for example, “Apple“ data is 1000x more than other data, the “Apple“ partitions will be much larger than other partitions, which causes the tasks that handle “Apple“ partitions to run much slower. With AQE in place, the “Apple“ data is split into smaller partitions behind the scenes and assigned to more executors, improving overall performance.
    
    [
    
    ![](https://substackcdn.com/image/fetch/$s_!6Y8O!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fb9e6532c-ff19-45ee-bb6d-1ba4a3eb4788_1372x1258.png)
    
    
    
    ](https://substackcdn.com/image/fetch/$s_!6Y8O!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fb9e6532c-ff19-45ee-bb6d-1ba4a3eb4788_1372x1258.png)
    
- **Dynamically switching join strategies**: Also, with the AQE, the optimizer can dynamically change the join strategies at runtime. For example, it can convert a broadcast hash mentioned above and join when the runtime statistics for any table fall below a threshold.
    
    [
    
    ![](https://substackcdn.com/image/fetch/$s_!uUfM!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F8de272e5-7dac-4173-8458-bb3daa32091b_1310x840.png)
    
    
    
    ](https://substackcdn.com/image/fetch/$s_!uUfM!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F8de272e5-7dac-4173-8458-bb3daa32091b_1310x840.png)
    

## Advanced configurations

There are other configurations you can consider when dealing with shuffled data; however, _**I recommend you touch these configurations only when you know what you’re doing, as those strictly relate to low-level in-memory and network operations**_:

- **spark.shuffle.file.buffer**: This is the size of the Mapper’s in-memory buffer for each shuffle file output stream. The default is 32KB. If you increase it, you will have less I/O overhead because the mapper writes to disk less frequently. However, a higher buffer means the mapper will have more memory pressure.
    
    [
    
    ![](https://substackcdn.com/image/fetch/$s_!HjIZ!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F9665cf44-342b-4e12-99b4-e1c25760c6d2_942x606.png)
    
    
    
    ](https://substackcdn.com/image/fetch/$s_!HjIZ!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F9665cf44-342b-4e12-99b4-e1c25760c6d2_942x606.png)
    
- **spark. reducer.maxSizeInFlight:** This configuration implies a fixed memory buffer per reduce task to receive mapper output. Higher means the reducer will fetch more data from mappers in a single request (if your network bandwidth allows); on the other hand, the reducer will now be under more memory pressure as it reserves more memory to receive data.
    
    [
    
    ![](https://substackcdn.com/image/fetch/$s_!aagY!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F726a11de-12ca-4b91-8cf0-b76265c0c54c_1890x734.png)
    
    
    
    ](https://substackcdn.com/image/fetch/$s_!aagY!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F726a11de-12ca-4b91-8cf0-b76265c0c54c_1890x734.png)
    

In addition, you can consider using faster hardware for your Spark cluster. For example, use SSDs for your executor to speed up I/O operations on shuffle output.

---

## Outro

In this article, I first introduce Spark’s narrow and wide dependencies transformation and note that shuffling occurs when data needs to be moved around so records with similar attributes (e.g., join keys or aggregation columns) can be processed in the same partition.

Then we explored the detailed process of data shuffling, from mappers writing shuffle outputs to disk to the reducer pulling them over the network. From that, we discovered strategies to optimize the shuffle process: eliminate shuffling in joins, reduce shuffle data, manually control reducer partitions or use AQE, and implement other advanced configurations related to the in-memory buffer and network.

Now, it’s time to say goodbye. See you in my next articles.

---

## Reference

_[1] [Spark Github Repo](https://github.com/apache/spark/tree/master)_

_[2] [Tuning Spark](https://spark.apache.org/docs/latest/tuning.html#memory-usage-of-reduce-tasks)_

_[3] luminousmen, [Spark Partitions](https://luminousmen.substack.com/p/spark-partitions?utm_source=publication-search)_

_[4] Alexey Grishchenko, [Spark Architecture: Shuffle](https://0x0fff.com/spark-architecture-shuffle/)_

_[5] Bartosz Konieczny, [Shuffling in Spark](https://www.waitingforcode.com/apache-spark/shuffling-in-spark/read)_

_[6] Bartosz Konieczny, [Shuffle writers: UnsafeShuffleWriter](https://www.waitingforcode.com/apache-spark/shuffle-writers-unsafeshufflewriter/read)_

_[7] [Spark Configuration](https://spark.apache.org/docs/latest/configuration.html)_