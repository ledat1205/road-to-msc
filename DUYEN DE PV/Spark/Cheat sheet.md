![[spark_cheatsheet (2).html]]

|Color in your chart|Name in legend / Spark UI|What it actually measures|Typical meaning / when it's large / how to optimize|
|---|---|---|---|
|**Blue**|Scheduler Delay|Time between when the task was **submitted** by the driver (to the cluster scheduler) and when the executor actually **started running** it.|Queuing in the driver → high values = too many tasks at once, small executor cores, slow driver, dynamic allocation lag, or resource contention on K8s. Reduce partitions or increase driver resources.|
|**Red**|Task Deserialization Time|Time spent **deserializing** the task object (code + closure + variables captured from driver) on the executor before it can run.|Large broadcast variables, big user-defined functions (UDFs), or complex closures. Optimize: avoid capturing large objects, use broadcast wisely, prefer column expressions over UDFs.|
|**Orange**|Shuffle Read Time|Time spent **reading shuffle data** from previous stage (fetching map outputs over network + local disk if spilled). Includes decompress/decode time.|Large shuffles, network congestion, slow disk, skew. Mitigate: increase shuffle partitions, enable AQE, use compression, fix data skew (salting).|
|**Green**|Executor Computing Time|The **actual useful work** — running your transformation logic (filter, map, aggregate, etc.) on the data partition. This is the part you want to dominate the bar.|High = heavy computation (complex UDFs, large data per partition). Optimize: better algorithms, caching, push down filters/projections, use native Spark functions.|
|**Yellow**|Shuffle Write Time|Time spent **writing shuffle data** to disk (serializing, partitioning, spilling if needed) so the next stage can read it.|Large output data, poor partitioning, high cardinality. Reduce: repartition smarter, use bucketing, enable AQE coalesce.|
|**Purple**|Result Serialization Time|Time to **serialize** the final task result (after computation) before sending it back to the driver (for actions like collect/count).|Very large result sent back to driver (e.g., collect() on huge data). Avoid: never collect large datasets; use reduce/aggregate instead.|
|**Cyan**|Getting Result Time|Time for the **driver** to **receive and process** the task result from the executor (network transfer + driver-side deserialization/aggregation).|Slow network, large results being sent back, driver overload. Avoid large driver-side collections; prefer distributed writes (saveAsTable, write.parquet).|
### Quick Summary – Ideal vs Problematic Bar Patterns

- **Healthy task** → mostly **green** (computing time dominates), very thin other colors.
- **Scheduler bottleneck** → long **blue** at the beginning → too many concurrent tasks or slow driver.
- **Serialization/closure issues** → noticeable **red** → optimize closures, reduce captured data.
- **Shuffle-heavy job** → large **orange** + **yellow** → classic wide transformation (join/groupBy); tune partitions, use AQE.
- **Driver overload** → big **purple** or **cyan** → stop collecting large data to driver.
- **Skew** → some bars much longer than others (often green or shuffle parts huge on few tasks).

### How to Read Your Specific Chart

- Tasks are grouped by **executor ID** or index (0/, 1/ etc. on the left).
- Many tasks have short **blue** (low scheduler delay) and mostly **green** → computation is the main work.
- Some tasks show visible **orange**/**yellow** → shuffle is happening (likely a reduce-side stage).
- Very little **purple**/**cyan** → good, results aren't huge or driver isn't bottlenecked.
- If many bars have noticeable **red** at start → look at broadcast variables or large UDF closures
-
![[Pasted image 20260320012459.png]]
![[Pasted image 20260320010007.png]]
![[Pasted image 20260320010121.png]]
![[Pasted image 20260320010342.png]]
![[Pasted image 20260320010546.png]]
![[Pasted image 20260320010811.png]]
![[Pasted image 20260320010953.png]]
![[Pasted image 20260320011054.png]]

## Optimization

![[Pasted image 20260320013326.png]]![[Pasted image 20260320013418.png]]
![[Pasted image 20260320013447.png]]![[Pasted image 20260320013517.png]]

![[Pasted image 20260320013549.png]]
![[Pasted image 20260320013605.png]]
![[Pasted image 20260320013616.png]]

![[Pasted image 20260320013632.png]]