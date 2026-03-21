I wrote an article to discuss [everything you need to know about Spark](https://open.substack.com/pub/vutr/p/if-youre-learning-apache-spark-this?r=2rj6sg&utm_campaign=post&utm_medium=web&showWelcomeOnShare=false) for batch processing. Spark is more than that; it is also a reliable stream processing engine. For this week's article, I plan to compare Spark Structured Streaming and Apache Flink. However, I realized that both have numerous aspects that require discussion, so I decided to write a dedicated article about Spark’s stream processing engine first.

That way, I can focus more on the research process and keep the article from being too long. We will first explore its architecture. From there, we move on to explore how they support event time processing, stateful processing, and fault tolerance.

> _**Note**: This article won’t talk much about stream processing concepts, such as windowing. I highly recommend reading my previous article, [Batch and Stream Processing](https://vutr.substack.com/p/batch-and-stream-processing)._

## Architecture

Structured Streaming is a stream processing engine built on the Spark SQL engine. Its core design principle is to treat a continuous stream as a subset of bounded data. This approach enables Spark creators to leverage its robust batch engine for stream processing.

### Spark cluster

That said, revisiting the way a batch processing application is handled in Spark before we move on is very helpful.

A Spark application consists of:

[

![](https://substackcdn.com/image/fetch/$s_!u5Kg!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F581279d6-1900-41ee-924d-85cd229a6e0c_576x418.png)



](https://substackcdn.com/image/fetch/$s_!u5Kg!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F581279d6-1900-41ee-924d-85cd229a6e0c_576x418.png)

- **Driver:** This JVM process manages the entire Spark application, from handling user input to distributing tasks to the executors.
    
- **Executors:** These processes execute tasks that the driver assigns and report their status and results. Each Spark application has its own set of executors.
    

### Anatomy

A Spark application is a cluster of driver and executors. Each application can have multiple jobs. A **job** represents a series of transformations applied to data. It encompasses the entire workflow from start to finish.

[

![](https://substackcdn.com/image/fetch/$s_!3QsF!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F0aa8b3a1-ff25-470b-b13f-b87ae53d4275_1074x492.png)



](https://substackcdn.com/image/fetch/$s_!3QsF!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F0aa8b3a1-ff25-470b-b13f-b87ae53d4275_1074x492.png)

A job is defined by an action and is split into different **stages** when a transformation requires shuffling data across partitions. Each **stage** is divided into multiple tasks, which execute processing in parallel across various partitions.

### Typical life cycle of an application

Users will define the processing logic for the application. It must include the SparkSession object, serving as the central gateway for interacting with all Spark’s functionalities.

[

![](https://substackcdn.com/image/fetch/$s_!oi_M!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F6535e408-b004-4337-bf00-7c1d801a7d56_614x542.png)



](https://substackcdn.com/image/fetch/$s_!oi_M!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F6535e408-b004-4337-bf00-7c1d801a7d56_614x542.png)

The user then submits a Spark application to the cluster manager and requests the driver resource. When the cluster manager accepts this submission, it initiates the driver process. The driver then asks the cluster to launch executors.

Based on the user’s defined logic, the driver will form the execution plan and start scheduling tasks on the executors. Then, executors physically execute the tasks and send the status to the driver. The process continues until all tasks are processed.

### Back to Spark Structured Streaming

When we start a streaming application in Spark Structured Streaming, we create a long-running Spark application. The Driver process stays active continuously, managing the entire streaming query lifecycle. Each stream will have a trigger. This trigger defines _when_ Spark should check for new data. When the trigger fires, the Spark engine does the following:

- It queries the source (e.g., asks Kafka “what are the latest offsets?”).
    
- It identifies the new data that has arrived since the last batch (e.g., Kafka offsets 1001 to 5000).
    
- This chunk of new data is conceptualized as a **micro-batch**. Internally, Spark treats this micro-batch as a small, static DataFrame.
    

[

![](https://substackcdn.com/image/fetch/$s_!7oHC!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F05fabe90-4b15-4c75-bd62-74a83a8c0fa5_730x270.png)



](https://substackcdn.com/image/fetch/$s_!7oHC!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F05fabe90-4b15-4c75-bd62-74a83a8c0fa5_730x270.png)

Once Spark has defined the micro-batch as a DataFrame, it applies all the transformations you described in your code (select, withColumn, groupBy, filter, etc.). Spark will form job(s) for each batch. Please remember that transformations in Spark are lazy. Nothing actually executes until an action is called. In Structured Streaming, the action is writing to the sink (e.g., `.writeStream.format(”parquet”).start(”/path/to/sink”)`).

For each micro-batch:

- The full query plan (transformations + sink action) is applied to the micro-batch DataFrame.
    
- Logical and physical plans are created.
    
- The driver schedules the tasks for executors based on the physical plan.
    

[

![](https://substackcdn.com/image/fetch/$s_!AYZW!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fe9aca718-3945-42c4-8a65-c88f025f1fa5_704x396.png)



](https://substackcdn.com/image/fetch/$s_!AYZW!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fe9aca718-3945-42c4-8a65-c88f025f1fa5_704x396.png)

The process continues forever (in theory) as each micro-batch is retrieved incrementally. In a real-world application, each micro-batch can contain several files or a range of offsets from sources such as Kafka or Kinesis.

## The trigger

To define the frequency of the trigger, the user can choose from the following types:

[

![](https://substackcdn.com/image/fetch/$s_!PV36!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F0f6f49bc-ad74-490f-a71a-1ac7c4050550_802x318.png)



](https://substackcdn.com/image/fetch/$s_!PV36!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F0f6f49bc-ad74-490f-a71a-1ac7c4050550_802x318.png)

- **Default Trigger**: No explicit trigger is specified. Spark runs the query as fast as possible. A new micro-batch is started immediately after the previous one completes if new data is available. This type removes the time spent waiting for a scheduled interval.
    
- **Fixed-Interval:** The query attempts to execute, retrieve, and process the micro-batch at a fixed interval, regardless of when data arrives.
    
    - If a batch finishes _faster_ than the interval, Spark waits for the remainder of the time.
        
    - If a batch takes _longer_ than the interval, the next batch starts immediately upon the current batch’s completion and doesn’t wait for the next scheduler.
        
    
    [
    
    ![](https://substackcdn.com/image/fetch/$s_!NsHF!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F7eb11e73-ae78-4f99-bdcc-e88738294b9d_582x252.png)
    
    
    
    ](https://substackcdn.com/image/fetch/$s_!NsHF!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F7eb11e73-ae78-4f99-bdcc-e88738294b9d_582x252.png)
    
- **One-Time:** The trigger processes a finite amount of data and then stops the streaming query, making them ideal for running streaming logic as a single, batch job.
    
- **Available now, micro-batch:** It is similar to the one-time trigger; however, it can process the data in multiple batches, depending on the source.
    

## The batch size

In most cases, the primary factor to consider when working with Apache Spark Structured Streaming is the batch size. The user can specify some options to act as a throttle, limiting the _maximum amount_ of data consumed in a single micro-batch

[

![](https://substackcdn.com/image/fetch/$s_!iguB!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Faff6013d-f97d-4d71-ab9e-e5b4d8871fd4_646x230.png)



](https://substackcdn.com/image/fetch/$s_!iguB!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Faff6013d-f97d-4d71-ab9e-e5b4d8871fd4_646x230.png)

- **maxOffsetsPerTrigger**: It limits the maximum number of records (offsets) to be consumed in one micro-batch for sources like Apache Kafka, Kinesis.
    

> To celebrate Lunar New Year (the true New Year holiday in Vietnam), I’m offering _**50% off the annual subscription**_. The offer ends soon; grab it now to get full access to nearly 200 high-quality data engineering articles.
> 
> [50% off the annual subscription](https://vutr.substack.com/subscribe)

[

![](https://substackcdn.com/image/fetch/$s_!-Nl1!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F65177376-8e2a-4443-a6e9-5e394a06b94d_584x348.png)



](https://substackcdn.com/image/fetch/$s_!-Nl1!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F65177376-8e2a-4443-a6e9-5e394a06b94d_584x348.png)

- **maxFilesPerTrigger**: It limits the maximum number of files to be processed in one micro-batch for sources such as Delta Lake, JSON, or Parquet files.
    
    [
    
    ![](https://substackcdn.com/image/fetch/$s_!jDZd!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F2f4007ac-4982-40e7-887e-c9439d017079_588x334.png)
    
    
    
    ](https://substackcdn.com/image/fetch/$s_!jDZd!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F2f4007ac-4982-40e7-887e-c9439d017079_588x334.png)
    
- **maxBytesPerTrigger:** It (softly) limits the maximum total bytes of data to be processed in one micro-batch for sources such as Delta Lake, JSON, or Parquet files. Unlike the two options above, the total bytes can exceed the limit if the last file is larger than the remaining limit
    
    - For example, if you set the maxBytesPerTrigger to 5GB, three 2GB files are still processed. (6GB> maxBytesPerTrigger)
        

You might wonder if the trigger interval can control the batch size. In fact, it can, but not directly. This approach is only practical when you know the source throughput and the stream is stable. The throughput allows you to approximate the processed data for each trigger, and the stream’s stability will give every trigger roughly the same amount of data.

Now you can control the batch size. Great. However, choosing the size that best fits your needs is not easy, as it, like everything in the world, has trade-offs:

[

![](https://substackcdn.com/image/fetch/$s_!ql_R!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F296ce011-bbfc-494d-aa29-61d1093690da_1190x402.png)



](https://substackcdn.com/image/fetch/$s_!ql_R!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F296ce011-bbfc-494d-aa29-61d1093690da_1190x402.png)

- **Too Small Batch**: As discussed in the `Architecture` section, when Spark processes a micro batch, it goes from planning to scheduling tasks to the executors. Planning and scheduling have overhead. If the batch size is small, the engine will have more batches to process, resulting in increased overhead time with each Spark batch. Imagine Spark spends 7 seconds planning for each batch; a 100-batch will result in ~ 12 minutes. You spend 12 minutes just planning!
    
    - In some cases, **s**mall batches may not contain enough data to fully utilize the Spark cluster’s available resources, leaving resources idle
        
    
    [
    
    ![](https://substackcdn.com/image/fetch/$s_!XJf6!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F25ac6d4a-73f4-4e26-999a-803ee831818f_802x236.png)
    
    
    
    ](https://substackcdn.com/image/fetch/$s_!XJf6!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F25ac6d4a-73f4-4e26-999a-803ee831818f_802x236.png)
    
- **Too Huge Batch:** Latency is indeed affected. All records must wait until the entire batch is processed. If a single batch takes 10 minutes, the freshest record in that batch is 10 minutes old by the time it reaches the sink.
    
    - Large batches, especially for stateful operations (like complex aggregations or joins), can put pressure on the Spark executors’ memory.
        

## Watermark, Event Time Processing, and Late Data

### Event time, processing time, and the skew

When it comes to stream processing, time is a crucial factor. Some applications require processing data records in the order they occur in real life. For example, in a real-time leaderboard for a multiplayer game, if Player A completes a challenge at 08:45:14.100 and Player B completes it at 08:45:15.250, the processing engine must ensure that Player A has a higher rank than Player B in the final result.

This leads to the two notions of time we usually see in stream processing systems:

- **Event Time**: the time the data event itself happened.
    
- **Processing Time**: The time at which an event is observed at any given point during processing.
    

In an ideal world, the event time equals the processing time; we could always process all events as they occur. However, due to network latency, machine failures, and human errors, the processing time will always be larger than the event time.

[

![](https://substackcdn.com/image/fetch/$s_!3wDX!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F5bcfb8fe-af94-47b1-8ad4-75c20bdfb619_592x608.png)



](https://substackcdn.com/image/fetch/$s_!3wDX!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F5bcfb8fe-af94-47b1-8ad4-75c20bdfb619_592x608.png)

So, if a user wants to perform aggregation on an event time window, such as “SUM visited users from 12:00 to 12:10”, how does the system know that all the data occurring between 12:00 and 12:10 has arrived? That is a critical question in stream processing: how do we ensure data completeness?

There is a solution for that. The difference between the event and the processing time is referred to as time skew. Watermark is a typical method for measuring skew and is a tool used by streaming systems to verify data completeness. It is most used for:

- Determining when to trigger the event-time window.
    
- Specifying how late events can be
    

### Window

Speaking of a window, it is a mechanism that limits the search space for stateful aggregations, such as sum, count, or join. All of them involve searching records with the same key. Windowing divides the infinite stream into finite portions. Usually, the system uses time-based notions to organize data into windows (e.g., all data from the last hour will belong to one window). Now, the join, sum, or count can be executed in the scope of the window.

> _For stateless operations like filter, we don’t need windowing._

Spark supports three types of windows:

[

![](https://substackcdn.com/image/fetch/$s_!ARk1!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fffe615c0-899a-41cd-8a52-ce620c7e5c68_1424x612.png)



](https://substackcdn.com/image/fetch/$s_!ARk1!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fffe615c0-899a-41cd-8a52-ce620c7e5c68_1424x612.png)

- **Fixed (Tumbling)**: The windows are defined as static window size, e.g., hourly windows.
    
- **Sliding:** The windows are defined by a window size and slide period, e.g., 30-minute windows starting every five minutes.
    
- **Session:** The windows capture some period of activity over a subset of the data, in this case, per key. Typically, they are defined by a timeout gap.
    

Spark also allows users to define an event-time window. To work with it, a watermark must be specified. It is simply a threshold that dictates:

- When an event-time window can be **closed** and its result emitted
    
- Which in-memory state (for aggregation/joins) can be **cleared** to prevent memory issues?
    

### Spark’s watermark

Spark’s watermark is defined by the event time field and a threshold. This threshold specifies the latest time the data can arrive. The watermark will be calculated by the maximum observed event time minus the threshold, and we will refer to this as W.

Here is how the watermark is used:

> _Driver now includes the watermark information in the planning phase._

- **Closing the window**: If you need a window closed at 11:00 event time, the window will be closed when the W > 11:00
    
- **Handling the late event**: Any event with a timestamp **older than** the watermark value is considered **late** and may be excluded from the window.
    

Here is an example of a simple 10-minute window without a watermark:

[

![](https://substackcdn.com/image/fetch/$s_!bf66!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F82ea01c7-88d8-4a74-af3e-c1e7883446ad_1368x702.png)



](https://substackcdn.com/image/fetch/$s_!bf66!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F82ea01c7-88d8-4a74-af3e-c1e7883446ad_1368x702.png)

Then the user specifies a ten 10-minute threshold for the watermark:

[

![](https://substackcdn.com/image/fetch/$s_!s9rO!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F574fe172-1ff2-4b16-9c8a-caae275905c8_1266x826.png)



](https://substackcdn.com/image/fetch/$s_!s9rO!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F574fe172-1ff2-4b16-9c8a-caae275905c8_1266x826.png)

The advantage of this approach is its simplicity, requiring users to specify only the event time field and the threshold.

However, it has several disadvantages. First, because it relies on the micro-batch execution cycle, the watermark only progresses at the end of each batch. This introduces a minimum latency equal to the batch interval plus the watermark delay, making it less suitable for actual millisecond-latency requirements.

Second, the fix threshold is effective only if the difference between event time and processing time remains constant. For example, if the threshold is 5 minutes, we assume that the processing and event times will differ by no more than 5 minutes. Initially, most records are less than 5 minutes late. Due to network issues, most records are more than 6 minutes late, causing the application to discard too many records optimistically.

## Stateful processing

A **stateful** streaming query requires Spark to maintain an **intermediate state** (a memory of past events) across micro-batches to calculate the correct result. Simply put, a state is a variable that is updated with new data to yield the final result. For example, to count the website visitors in a window, Spark the counter somewhere to accumulate the number of visitors.

[

![](https://substackcdn.com/image/fetch/$s_!zgVg!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Ff13dc43a-9f7e-4fef-8308-a2126d1f6648_622x424.png)



](https://substackcdn.com/image/fetch/$s_!zgVg!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Ff13dc43a-9f7e-4fef-8308-a2126d1f6648_622x424.png)

The state can grow infinitely. There must be a way for Spark to know when to clean it up. And guess what, the watermark also plays a crucial role here. The window’s state can be cleaned up when the watermark passes the end of the window.

Because, as discussed above, when this happens, Spark can assume that all data (including late data within the threshold) has arrived; the window’s final aggregation, such as the number of visitors, can be finalized.

### Stateful operation

Spark’s stateful operations include streaming aggregation, dropDuplicates, stream-stream joins, and custom stateful applications:

- **Streaming Aggregation** (Grouped/Running Aggregations): This is the most common. It involves continuously calculating aggregate values (such as count, sum, and average) over the entire stream or over a window.
    
    [
    
    ![](https://substackcdn.com/image/fetch/$s_!UZOx!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F89d9eff3-e63d-408f-9768-48950afc8c68_964x442.png)
    
    
    
    ](https://substackcdn.com/image/fetch/$s_!UZOx!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F89d9eff3-e63d-408f-9768-48950afc8c68_964x442.png)
    
- **Stream-Stream Joins:** Joining two continuous, unbounded streams of data. Implementing a stream-stream join is more challenging than a batch-batch join because, at any given time, the engine lacks a complete view of the dataset. This raises questions: Do records with this key not actually appear? Or, will they arrive in the future?
    
    [
    
    ![](https://substackcdn.com/image/fetch/$s_!tWch!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F19d70b23-1de4-4f15-8e88-b15d670f042a_806x340.png)
    
    
    
    ](https://substackcdn.com/image/fetch/$s_!tWch!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F19d70b23-1de4-4f15-8e88-b15d670f042a_806x340.png)
    
    - That’s why both streams are buffered in states. The engine said: “We can wait for a bit to make sure we don’t miss the matched records“.
        
    - Of course, the buffer can’t grow forever; the watermark is also used to control the size of the buffer here.
        
- **dropDuplicates:** It ensures that a record, identified by a unique key, is only processed once within a specified period of time. Spark must store the unique key of every record it has seen for a duration to check if an incoming record is a duplicate.
    
    [
    
    ![](https://substackcdn.com/image/fetch/$s_!NO1H!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Ff88bff31-1c58-403a-a267-e84af9ca58e0_650x284.png)
    
    
    
    ](https://substackcdn.com/image/fetch/$s_!NO1H!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Ff88bff31-1c58-403a-a267-e84af9ca58e0_650x284.png)
    
- **Custom Stateful Applications (Arbitrary State):** Spark allows users to implement custom stateful operations based on their needs.
    

### State store

So, the state must be stored somewhere.

Simply put, the state store is a key-value store that allows the Spark Executors to read and write state. Spark Structured Streaming supports two stores:

- **HDFS-Backed State Store (Default)**: It stores the state data in an executor’s memory at first. If the state size grows large, it can easily lead to **Out-of-Memory (OOM)** errors. Storing state in the executor’s memory can also cause high garbage collection overhead. The state is then persisted transactionally to the files in an HDFS-compatible system.
    
    [
    
    ![](https://substackcdn.com/image/fetch/$s_!vawG!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F3c0ebb87-4c61-4f00-8fba-3c5b2ae72e3a_752x296.png)
    
    
    
    ](https://substackcdn.com/image/fetch/$s_!vawG!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F3c0ebb87-4c61-4f00-8fba-3c5b2ae72e3a_752x296.png)
    
- **RocksDB state store:** Since Spark 3.2, RocksDB, the famous embedded C++ key-value store, has been introduced as an option to store state. Rather than storing the state in the executor’s JVM memory, the state is stored in the RocksDB instance’s memory and disk.
    
    [
    
    ![](https://substackcdn.com/image/fetch/$s_!RdJr!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F8921506f-9e8c-4a18-bae3-cffbff8542e5_522x298.png)
    
    
    
    ](https://substackcdn.com/image/fetch/$s_!RdJr!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F8921506f-9e8c-4a18-bae3-cffbff8542e5_522x298.png)
    

## Fault tolerance

Machines can fail anytime. Humans can cause bugs. A system only operates forever in theory. It must have the ability to recover from failure, especially a stream processing system, which could back the most critical business operation in your company. Fault tolerance is one of the most essential features of Apache Spark Structured Streaming. Spark relies on checkpointing to offer that ability.

### Checkpointing

Users can configure the streaming application with a checkpoint location (HDFS-compatible system). Spark then periodically saves the entire progress (including the processed source offsets), the “identity” of a streaming query, and the state information, to that location. This helps the stream query recover to its previous status just before the failure.

[

![](https://substackcdn.com/image/fetch/$s_!-HE_!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F21a5c2ed-f3ae-48e4-a819-bd728f50f0a3_1272x332.png)



](https://substackcdn.com/image/fetch/$s_!-HE_!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F21a5c2ed-f3ae-48e4-a819-bd728f50f0a3_1272x332.png)

For the sources, the engine assumes that they are **replayable** and supports the concept of **offsets** to track the consumed position (e.g., Kafka). These sources allow Spark to request data starting from a specific, committed offset. Suppose Spark processes data from offset 100 to 150 but fails before committing. In that case, the source is guaranteed to re-deliver records starting from 100 on restart (using the offset stored in the checkpoint location).

Here is how checkpointing works behind the scenes:

- When you specify the `checkpointLocation`, Spark creates a directory with a specific structure. The most essential subdirectories are:
    
    - **offsets**: Acts as a write-ahead log (WAL). Before processing any batch, Spark writes the range of data it’s about to process here (e.g., “for batch 10, I will process Kafka offsets 501-600”).
        
    - **commits**: Contains commit files. With file 9, it means that batch #9 has been fully processed and committed. This is the source of truth for completed work.
        
- At the start of the micro batch, the engine must first write down the data range it gonna to process in the offsets folder. Given the engine is going to process batch #10, the offset write action informs that “for batch #10, I will process the Kafka offset 501-600“
    
    [
    
    ![](https://substackcdn.com/image/fetch/$s_!Mi4c!,w_2400,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Ff57bfb01-1f4c-4399-96fd-442f9cbc3aa6_918x232.png)
    
    
    
    ](https://substackcdn.com/image/fetch/$s_!Mi4c!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Ff57bfb01-1f4c-4399-96fd-442f9cbc3aa6_918x232.png)
    
- The engine processes data and writes to the sink
    
    > _For simplicity, we assume the data is output to the sink right at the end of the micro-batch. As discussed in the Watermark section, the engine can wait for late data, so the output materialization may not occur at the end of the micro-batch._
    
- The engine writes a file in the **commits** folder to inform that: “Batch #10 is successfully finished.“
    
- In case of failure, let's say the driver crashes in the middle of the processing of batch #10; the commit file of batch #10 is never written in the commits folder.
    
- When the application recovers, the engine looks into the checkpoint folder and sees that:
    
    [
    
    ![](https://substackcdn.com/image/fetch/$s_!YPnK!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fcbc91315-eb45-45d7-bc9b-abaa61948b5f_986x632.png)
    
    
    
    ](https://substackcdn.com/image/fetch/$s_!YPnK!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fcbc91315-eb45-45d7-bc9b-abaa61948b5f_986x632.png)
    
    - The engine tried to process the Kafka offset 501-600 for batch #10 (by looking at the offsets folder)
        
    - However, the process fails (by seeing that only file #9 is in the commits folder)
        
    - The engine understands that it needs to preprocess batch #10 with the 501-600 offset ranges.
        
    - The role of Kafka or any replayable source is essential here as it allows Spark to “rewind” to a specific offset.
        

## Exactly-once guarantee

Fault tolerance is great.

However, some business operations want more than that. They require the streaming query to be not only resilient but also semantically correct; data loss or duplication is not allowed. In other words, they need the exact-once guarantee.

Spark Structured Streaming can provide:

> _Delivering end-to-end exactly-once semantics was one of the key goals behind the design of Structured Streaming. To achieve that, we have designed the Structured Streaming sources, the sinks, and the execution engine to reliably track the exact progress of the processing so that it can handle any failure by restarting and/or reprocessing — [Source](https://spark.apache.org/docs/latest/streaming/getting-started.html) —_

The key idea is that for the end-to-end pipeline to be exactly-once guaranteed, the source, the data transform process, and the sink must ensure the property:

- **Source** like Kafka’s default guarantee is At-Least-Once because its primary design philosophy prioritizes no data loss and high throughput in an unreliable, distributed environment. This means the data flows from Kafka can be duplicated. To ensure exact-once delivery, a mechanism must be in place to deduplicate the message here.
    
    [
    
    ![](https://substackcdn.com/image/fetch/$s_!UU8G!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F810db997-55cf-4629-9ab0-e5664dfb9ee7_808x268.png)
    
    
    
    ](https://substackcdn.com/image/fetch/$s_!UU8G!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F810db997-55cf-4629-9ab0-e5664dfb9ee7_808x268.png)
    
- Structured Streaming can ensure no missing **data processing** here, thanks to the checkpoint mechanism discussed above. Plus, the same data might not be processed **successfully** **twice**, given that each micro-batch can only have one associated commit file in the commits folder.
    
    [
    
    ![](https://substackcdn.com/image/fetch/$s_!t9OH!,w_2400,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fdd5bf09f-16d8-48c2-9fca-8aa3b17cf6d6_1180x312.png)
    
    
    
    ](https://substackcdn.com/image/fetch/$s_!t9OH!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fdd5bf09f-16d8-48c2-9fca-8aa3b17cf6d6_1180x312.png)
    
- However, the result of a **Structured Streaming** might be materialized to the **sink** more than once. Back to the example of writing batch #10 above, the engine must write the output to the sink before committing the file. After materializing to the sink, the engine will write the commit file to the commits folder. However, the application fails, as there is no commit file for batch 10, despite the result of batch 10 already being written.
    
    [
    
    ![](https://substackcdn.com/image/fetch/$s_!OFCr!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fdb7ea493-9c85-434c-a034-127911398442_882x488.png)
    
    
    
    ](https://substackcdn.com/image/fetch/$s_!OFCr!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fdb7ea493-9c85-434c-a034-127911398442_882x488.png)
    
    - When recovered, the engine sees that there is a file #10 in the offsets folder, but no file #10 in the commits folder.
        
    - It restarts the processing of batch #10.
        
        [
        
        ![](https://substackcdn.com/image/fetch/$s_!00PN!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F787836c7-6b53-4057-8ab6-7af3a2271e03_932x708.png)
        
        
        
        ](https://substackcdn.com/image/fetch/$s_!00PN!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F787836c7-6b53-4057-8ab6-7af3a2271e03_932x708.png)
        
    - It processes and finally writes to the sink the result of batch #10, for the **second time**.
        
    - The key here is that the sink must be idempotent to ensure exactly once. Overwriting the whole table is a good example here.
        
        [
        
        ![](https://substackcdn.com/image/fetch/$s_!1nAj!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fb020040d-73dc-4867-91e5-91662e27a4f3_850x382.png)
        
        
        
        ](https://substackcdn.com/image/fetch/$s_!1nAj!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fb020040d-73dc-4867-91e5-91662e27a4f3_850x382.png)
        

What I want to say is that exactly-once delivery does not depend solely on your Spark Structured Streaming application; you must also consider whether your source and sink can provide this property. This requires careful reading of the document to understand the source or sink you have implemented.

## Outro

In this article, we first revisit the architecture of Spark, then move on to explore the architecture of Spark Structured Streaming, and understand that the batch size is an essential factor for the performance of its micro-batching approach. Next, we explore how it supports event-time processing (with watermark) and stateful processing. Finally, we come to how Spark Structured Streaming achieves fault tolerance and exactly-once processing.

Given that Spark is widely adopted and many engineers already have experience with it, Spark Structured Streaming is an attractive choice for stream processing with near-real-time demands. However, in cases where organizations need ultra-low latency, robust event-time, and stateful processing, the micro-batch approach and simple watermark might not meet the need. A true streaming engine like Flink is a better choice here; the trade-off is clear: more time and effort are needed to understand and operate the new system.

Thank you for reading this far. See you in my next articles.

## Reference

_[1] Sagar Lakshmipathy, [Apache Flink™ vs Apache Kafka™ Streams vs Apache Spark™ Structured Streaming — Comparing Stream Processing Engines](https://www.onehouse.ai/blog/apache-spark-structured-streaming-vs-apache-flink-vs-apache-kafka-streams-comparing-stream-processing-engines), 2025_

_[2] [Structured Streaming Programming Guide](https://spark.apache.org/docs/latest/streaming/index.html)_

_[3] Canadian Data Guy, [How Spark Structured Streaming Recovers After Failures](https://www.canadiandataguy.com/p/how-spark-structured-streaming-recovers), 2025_

_[4] StackOverflow answer from Jungtaek Lim, [Why so much criticism around Spark Streaming micro-batch (when using Kafka as source)?](https://stackoverflow.com/questions/65491431/why-so-much-criticism-around-spark-streaming-micro-batch-when-using-kafka-as-so), 2020_