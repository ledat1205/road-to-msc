Here are the errors every Spark developer hits, roughly in the order you encounter them as you go deeper.

---

## 1. AnalysisException: cannot resolve column name

**What you see**

```
AnalysisException: cannot resolve 'customer_id' given input columns: [id, name, amount]
```

**Why it happens** Column name doesn't exist in the DataFrame at that point in the plan. Most common causes: typo, wrong case (`customerId` vs `customer_id`), referencing a column before it was created, or referencing a column from a different DataFrame in a join condition using string syntax instead of DataFrame column objects.

**Fix**

```python
# Wrong — string reference after rename
df.withColumnRenamed("id", "customer_id").filter(col("id") == 1)  # "id" is gone

# Right
df.withColumnRenamed("id", "customer_id").filter(col("customer_id") == 1)

# Wrong — ambiguous column after join (both DataFrames have "id")
orders.join(customers, "id").select(col("id"))  # which "id"?

# Right — qualify with DataFrame reference
orders.join(customers, orders.id == customers.id).select(orders.id)
```

---

## 2. java.lang.OutOfMemoryError: Java heap space (Executor OOM)

**What you see**

```
ExecutorLostFailure: Executor 4 exited caused by: java.lang.OutOfMemoryError: Java heap space
```

**Why it happens** The partition being processed is too large for the executor's heap. Common triggers: `shuffle.partitions=200` on 500 GB of data (2.5 GB per task), a skewed key sending 90% of data to one partition, collecting a large broadcast variable, or a `collect()` pulling everything to driver.

**Fix**

```python
# Increase shuffle partitions — target 100–200 MB per partition
spark.conf.set("spark.sql.shuffle.partitions", "4000")

# Or enable AQE and set high — let it collapse down
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.shuffle.partitions", "10000")

# If it's the driver OOM (collect):
df.write.parquet("s3://output/")          # write instead of collect
df.limit(1000).toPandas()                 # sample for inspection only
```

---

## 3. java.lang.OutOfMemoryError: GC overhead limit exceeded

**What you see**

```
java.lang.OutOfMemoryError: GC overhead limit exceeded
```

**Why it happens** JVM is spending more than 98% of its time doing garbage collection and recovering less than 2% of heap. This means the heap is full of live objects that can't be freed. Almost always caused by too many Java objects — either large Python UDFs generating objects per row, or MEMORY_ONLY caching of a DataFrame that's too big.

**Fix**

```python
# Switch to G1GC — handles large heaps much better than default ParallelGC
--conf "spark.executor.extraJavaOptions=-XX:+UseG1GC -XX:InitiatingHeapOccupancyPercent=35"

# If caused by caching — use serialized storage (3× less memory, more CPU)
from pyspark import StorageLevel
df.persist(StorageLevel.MEMORY_AND_DISK_SER)

# Increase executor memory
--executor-memory 32g

# If caused by Python UDFs — replace with pandas_udf or native functions
```

---

## 4. FetchFailedException / org.apache.spark.shuffle.FetchFailedException

**What you see**

```
org.apache.spark.shuffle.FetchFailedException: Failed to connect to host/port
-- or --
org.apache.spark.shuffle.FetchFailedException: Missing an output location for shuffle 3
```

**Why it happens** A reduce task tried to fetch shuffle data from an executor that died or was lost. The shuffle files it needed are gone. Two root causes: executor was killed by OOM before the reduce stage started, or the executor node went away (spot instance preemption, hardware failure, timeout).

**Fix**

```python
# Most common fix: the executor died from OOM — fix the OOM first
# (see error #2 and #3 above)

# If executors are being killed by YARN/K8s for exceeding memory limits:
--conf spark.executor.memoryOverhead=4g     # increase off-heap budget

# If spot instances are being preempted:
spark.conf.set("spark.decommission.enabled", "true")
spark.conf.set("spark.storage.decommission.shuffleBlocks.enabled", "true")

# Increase max retries for fetch failures
spark.conf.set("spark.task.maxFailures", "8")       # default 4
spark.conf.set("spark.shuffle.io.maxRetries", "10") # default 3
spark.conf.set("spark.shuffle.io.retryWait", "60s") # default 5s
```

---

## 5. Task not serializable

**What you see**

```
org.apache.spark.SparkException: Task not serializable
Caused by: java.io.NotSerializableException: com.myco.SomeClass
```

**Why it happens** Spark serializes the task closure — everything your lambda/function references — and ships it to executors. If your function references an object that isn't Java-serializable (a database connection, a file handle, a non-serializable class, or the SparkSession itself), serialization fails. Most commonly happens when you accidentally capture `self` in a method used as a UDF.

**Fix**

```python
# Wrong — captures 'self' which holds a DB connection (not serializable)
class Processor:
    def __init__(self):
        self.db = DatabaseConnection()  # not serializable

    def transform(self, df):
        return df.rdd.map(lambda row: self.db.lookup(row.id))  # captures self

# Right — create the connection inside the function (one per partition)
def transform(df):
    def process_partition(rows):
        db = DatabaseConnection()   # created inside executor, not serialized
        for row in rows:
            yield db.lookup(row.id)
    return df.rdd.mapPartitions(process_partition)

# Right for UDFs — use module-level functions, not methods
def my_udf_func(value):
    return value.upper()

my_udf = udf(my_udf_func, StringType())  # module-level, no self captured
```

---

## 6. Py4JError / Py4JJavaError

**What you see**

```
py4j.protocol.Py4JJavaError: An error occurred while calling o42.collectToPython
-- or --
py4j.Py4JException: Target Object ID does not exist for this gateway
```

**Why it happens** The Python process lost its connection to the JVM (the SparkContext died or timed out). Common in notebooks when the session expired, or when you try to use a SparkContext after calling `spark.stop()`. The "Target Object ID does not exist" variant means you're holding a stale Python reference to a JVM object that no longer exists.

**Fix**

```python
# Never call spark.stop() in the middle of a notebook session
# If SparkContext died, restart the kernel — you need a fresh session

# Check if session is alive before using it
if spark.sparkContext._jsc is None:
    spark = SparkSession.builder.getOrCreate()

# Avoid long-running operations that exceed py4j gateway timeout
spark.conf.set("spark.network.timeout", "800s")
spark.conf.set("spark.executor.heartbeatInterval", "60s")
```

---

## 7. WARN HeartbeatReceiver: Removing executor X with no recent heartbeats

**What you see**

```
WARN HeartbeatReceiver: Removing executor 5 with no recent heartbeats: 130039 ms exceeds timeout 120000 ms
```

**Why it happens** An executor stopped sending heartbeats to the driver for longer than `spark.network.timeout` (default 120s). Usually means the executor was completely frozen — stuck in a long GC pause, doing intensive CPU work that blocked the heartbeat thread, or the network between executor and driver was interrupted.

**Fix**

```python
# Increase timeout — gives long GC pauses more room
spark.conf.set("spark.network.timeout", "800s")
spark.conf.set("spark.executor.heartbeatInterval", "60s")  # must be < network.timeout

# If it's GC pauses causing the freeze:
--conf "spark.executor.extraJavaOptions=-XX:+UseG1GC -XX:InitiatingHeapOccupancyPercent=35"

# If the executor is genuinely stuck on a heavy task — check for skew
# One task running for 130+ seconds on one executor = likely data skew
```

---

## 8. Job aborted due to stage failure: Total size of serialized results exceeds spark.driver.maxResultSize

**What you see**

```
Job aborted due to stage failure: Total size of serialized results of 450 tasks (1026.0 MB) is bigger than spark.driver.maxResultSize (1024.0 MB)
```

**Why it happens** Each task sends its result back to the driver. When you call `collect()`, `show()`, or run a job where tasks return large results (like a large `groupBy` result), the total size of all results exceeds the driver's result size limit (default 1 GB).

**Fix**

```python
# Increase the limit
spark.conf.set("spark.driver.maxResultSize", "4g")

# Better fix: don't collect large results to driver
df.write.parquet("s3://output/")         # write to storage instead
df.show(100)                              # show only a sample
df.limit(10000).toPandas()               # safe small sample

# If this is a legitimate groupBy result — write then read back
df.groupBy("country").count().write.csv("s3://counts/")
```

---

## 9. IllegalArgumentException: Unsupported class file major version

**What you see**

```
java.lang.IllegalArgumentException: Unsupported class file major version 61
```

**Why it happens** Your JAR was compiled with a newer Java version than what the executor JVM is running. Major version 61 = Java 17, 55 = Java 11, 52 = Java 8. If the executor runs Java 8 but the JAR was compiled targeting Java 17, it can't load the class.

**Fix**

```bash
# Compile with target version matching cluster JVM
javac --release 8 ...
# or in Maven:
# <maven.compiler.source>1.8</maven.compiler.source>
# <maven.compiler.target>1.8</maven.compiler.target>

# Or upgrade the cluster JVM — Spark 3.3+ recommends Java 11 or 17
# Check cluster Java version:
java -version
```

---

## 10. org.apache.spark.sql.AnalysisException: Resolved attribute(s) missing from child

**What you see**

```
AnalysisException: Resolved attribute(s) country#45L missing from child
```

**Why it happens** You created a column reference from one DataFrame and tried to use it in a different DataFrame's plan. This happens when you cache a reference to a `col()` object and use it after the DataFrame has been transformed, or when you mix column references from different DataFrames.

**Fix**

```python
# Wrong — col_ref was created from df1 but used on df2
col_ref = df1.groupBy("country")  # holds reference to df1's plan
result  = df2.join(col_ref, ...)  # df2 doesn't know about df1's attributes

# Right — always derive columns fresh from the current DataFrame
result = df2.join(df1.groupBy("country").agg(F.count("*")), "country")

# Wrong — reusing a column object after a DataFrame is transformed
col = df["country"]
df2 = df.withColumnRenamed("country", "nation")
df2.filter(col == "US")   # col still refers to df's "country", not df2's "nation"

# Right — re-derive from the new DataFrame
df2.filter(df2["nation"] == "US")
```

---

## 11. WARN TaskSetManager: Lost task X.0 in stage Y.0 (TID Z): ExecutorLostFailure

**What you see**

```
WARN TaskSetManager: Lost task 3.0 in stage 2.0 (TID 15, node-1, executor 2):
ExecutorLostFailure (executor 2 exited caused by one of the running tasks)
Reason: Container killed by YARN for exceeding memory limits. 12.5 GB of 12 GB physical memory used.
```

**Why it happens** YARN (or K8s) killed the executor container because it exceeded the physical memory limit. This is different from JVM heap OOM — it's the OS-level container memory limit being exceeded. Common cause: Python worker memory (PySpark), Arrow buffers, off-heap Tungsten memory, or JVM native memory all adding up beyond the container limit.

**Fix**

```python
# Increase memoryOverhead — this is the off-heap budget
--conf spark.executor.memoryOverhead=4g     # default is max(384MB, 10% of heap)

# PySpark-specific: Python worker needs extra memory
--conf spark.executor.memoryOverhead=6g     # higher for pandas-heavy workloads

# Check: total container memory = executor.memory + memoryOverhead
# YARN/K8s limit must be >= this total
# If YARN: yarn.scheduler.maximum-allocation-mb must be >= total
```

---

## 12. Broadcast exchange exceeds spark.driver.maxResultSize

**What you see**

```
SparkException: Failed to get broadcast_5_piece0 of RDD (broadcast 5 as broadcast)
-- or --
java.lang.OutOfMemoryError during broadcast
```

**Why it happens** You broadcast a table that's too large. The driver serializes the entire broadcast variable and ships it to every executor. If the table is larger than driver memory or exceeds the result size limit, it fails.

**Fix**

```python
# Check table size before broadcasting
print(df.count(), len(df.columns))   # quick sanity check

# Disable auto-broadcast if Catalyst is over-eager
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "-1")   # disable
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "50m")  # or raise limit carefully

# Increase driver memory if you genuinely need a large broadcast
--driver-memory 16g

# Alternative: use bucket joins instead of broadcast for medium tables
df.write.bucketBy(256, "key").saveAsTable("bucketed_table")
```

---

## 13. Schema mismatch / Cannot up cast

**What you see**

```
AnalysisException: Cannot up cast amount from string to double
-- or --
AnalysisException: schema mismatch in writing to table
```

**Why it happens** You're trying to write a string column where a double is expected, or union two DataFrames with incompatible types, or write to an existing Parquet/Delta table with a different schema.

**Fix**

```python
# Cast explicitly before writing
df.withColumn("amount", col("amount").cast("double"))

# For union — ensure same schema order and types
df1.select("id", "amount", "country") \
   .union(df2.select("id", "amount", "country"))  # same order!

# Use unionByName to match by column name instead of position
df1.unionByName(df2, allowMissingColumns=True)

# For Delta schema mismatch — enable schema evolution
df.write.format("delta") \
    .option("mergeSchema", "true") \   # adds new columns automatically
    .mode("append").save("s3://table/")
```

---

## 14. java.io.IOException: No space left on device

**What you see**

```
java.io.IOException: No space left on device
```

**Why it happens** Spark spilled shuffle data or cached data to the executor's local disk, and the disk filled up. Common on K8s when the pod's ephemeral storage limit is set too low, or on nodes with small local disks running large jobs.

**Fix**

```python
# Configure multiple local directories across different disks
spark.conf.set("spark.local.dir", "/data1/spark-temp,/data2/spark-temp,/data3/spark-temp")

# Reduce spill by right-sizing shuffle partitions
spark.conf.set("spark.sql.shuffle.partitions", "4000")  # smaller partitions = less spill

# On K8s — mount a large emptyDir volume for spark local dir
# In PodTemplate:
# volumes:
#   - name: spark-local
#     emptyDir:
#       sizeLimit: 200Gi
# volumeMounts:
#   - name: spark-local
#     mountPath: /tmp/spark-local

spark.conf.set("spark.local.dir", "/tmp/spark-local")
```

---

## 15. StreamingQueryException: Stream stopped due to checkpoint failure / query terminated

**What you see**

```
StreamingQueryException: Query [id] terminated with exception
-- or --
Caused by: java.io.FileNotFoundException: No such file or directory: s3://checkpoints/job/_commits/5
```

**Why it happens** The streaming checkpoint was deleted, moved, or corrupted. Or the checkpoint is on a location that became unavailable (S3 permission change, path deleted by a cleanup job). Also happens when you change the schema of a stateful aggregation and try to resume from an old checkpoint.

**Fix**

```python
# Prevention: checkpoint on durable, dedicated storage
query = df.writeStream \
    .option("checkpointLocation", "s3://spark-checkpoints/prod/orders-etl/v1/") \
    .start()
# Note the version in the path — when you change schema, bump to v2/

# If checkpoint is corrupt/gone and you must restart fresh:
# Delete checkpoint dir and restart with startingOffsets="earliest" or specific offsets
# Accept that state is lost and aggregations restart from zero

# If schema changed — you MUST use a new checkpoint location
# Old checkpoint: s3://checkpoints/orders-etl/v1/
# New checkpoint: s3://checkpoints/orders-etl/v2/

# Never let automated cleanup jobs touch checkpoint directories
```

---

## 16. WARN TaskSchedulerImpl: Initial job has not accepted any resources

**What you see**

```
WARN TaskSchedulerImpl: Initial job has not accepted any resources; check your cluster UI to ensure that workers are registered and have sufficient resources
```

**Why it happens** The driver submitted tasks but no executor accepted them. The cluster has no available resources — all executors are full, the YARN queue is at capacity, or on K8s the executor pods can't be scheduled (insufficient CPU/memory on nodes, node selector mismatch, taints without tolerations).

**Fix**

```bash
# Check YARN queue capacity
yarn queue -status root.data-engineering

# Check K8s pod scheduling issues
kubectl describe pod <executor-pod-name> -n spark-jobs
# Look for: "Insufficient cpu", "Insufficient memory", "Taint not tolerated"

# Common K8s fix: node selector doesn't match any available node
# Check that nodes with the required label actually exist and have capacity
kubectl get nodes -l node-role=spark-executor

# Reduce resource request to fit available nodes
--executor-memory 8g   # instead of 32g
--executor-cores 4     # instead of 16
```

---

## 17. ERROR TransportRequestHandler: Error sending result to / Connection reset by peer

**What you see**

```
ERROR TransportRequestHandler: Error sending result to /10.0.1.5:52341
java.io.IOException: Connection reset by peer
```

**Why it happens** A network connection between a task and the driver (or between executors during shuffle) was dropped mid-transfer. Can be caused by a task running for so long that the connection timed out, a network partition, or the receiving side (driver or executor) being killed while the transfer was in progress.

**Fix**

```python
# Increase network timeouts
spark.conf.set("spark.network.timeout", "800s")
spark.conf.set("spark.shuffle.io.connectionTimeout", "300s")
spark.conf.set("spark.shuffle.io.maxRetries", "10")
spark.conf.set("spark.shuffle.io.retryWait", "30s")

# If caused by very long-running tasks — investigate and fix the slow task (likely skew)
# spark.speculation=true helps: launches a duplicate task when one is unusually slow
spark.conf.set("spark.speculation", "true")
spark.conf.set("spark.speculation.multiplier", "1.5")
```

---

## 18. Hive metastore / Unable to instantiate org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient

**What you see**

```
ERROR DataNucleus.Persistence: Unable to instantiate org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient
```

**Why it happens** Spark can't connect to the Hive metastore. Either the metastore service is down, the JDBC URL in `hive-site.xml` is wrong, the metastore database (MySQL/Postgres) is unreachable, or there's a version mismatch between the Hive client jar and the metastore server.

**Fix**

```python
# Check hive-site.xml has correct metastore URI
# <property>
#   <name>hive.metastore.uris</name>
#   <value>thrift://metastore-host:9083</value>
# </property>

# Test metastore connectivity
spark.sql("SHOW DATABASES").show()

# Use Derby local metastore for dev/test (no external service needed)
spark = SparkSession.builder \
    .config("spark.sql.warehouse.dir", "/tmp/spark-warehouse") \
    .enableHiveSupport() \
    .getOrCreate()

# Skip Hive entirely if you don't need it
spark = SparkSession.builder.getOrCreate()  # no enableHiveSupport()
# Use Delta or Parquet directly — no metastore needed
```

---

## 19. Watermark not advancing / state growing unboundedly (Streaming)

**What you see** — not an exception but a silent killer. You check `query.lastProgress` and see:

```json
"eventTime": {"watermark": "2024-01-15T10:00:00.000Z"}
```

The watermark hasn't moved for 30 minutes even though new data is arriving.

**Why it happens** The `event_time` column is null, has a wrong timezone, is being parsed incorrectly, or the column referenced in `withWatermark()` doesn't actually contain event timestamps. State grows unboundedly because the watermark never advances so old state is never cleared.

**Fix**

```python
# Debug: inspect actual event_time values coming through
df.writeStream.foreachBatch(lambda batch, _: batch.select(
    "event_time",
    F.current_timestamp().alias("processing_time")
).show(5)).start()

# Common cause: timestamps parsed as string, not TimestampType
df.withColumn("event_time",
    F.to_timestamp(col("event_time_str"), "yyyy-MM-dd HH:mm:ss"))  # explicit parse

# Common cause: UTC vs local timezone mismatch
spark.conf.set("spark.sql.session.timeZone", "UTC")

# Verify watermark is actually advancing in query progress
import json
print(json.dumps(query.lastProgress["eventTime"], indent=2))
# eventTime.watermark should be close to current time minus delay
```

---

## 20. ClassNotFoundException: Failed to load class for data source

**What you see**

```
ClassNotFoundException: Failed to load class for data source: delta
-- or --
ClassNotFoundException: org.apache.spark.sql.delta.catalog.DeltaCatalog
```

**Why it happens** You're trying to use a format (Delta, Kafka, Iceberg) that isn't on the classpath. The JAR for that connector isn't included in the Spark session.

**Fix**

```bash
# Option 1: include at submit time with --packages (downloads from Maven)
spark-submit --packages io.delta:delta-spark_2.12:3.0.0 my_job.py

# Option 2: bake into Docker image (preferred for production)
RUN wget https://repo1.maven.org/maven2/io/delta/delta-spark_2.12/3.0.0/delta-spark_2.12-3.0.0.jar \
    -P /opt/spark/jars/

# Option 3: configure in SparkSession (Delta specific)
spark = SparkSession.builder \
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog") \
    .getOrCreate()

# For Kafka streaming connector:
spark-submit --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.0 my_job.py
```

---

## Quick diagnostic checklist

When a Spark job fails and you don't know where to start:

1. **Spark UI → Stages tab** — find the failed stage. Click it.
2. **Failed Tasks** — click the failed task. Read the full exception, not just the first line. The root cause is usually at the bottom of the stack trace after "Caused by:".
3. **Executor logs** — in YARN: `yarn logs -applicationId app_xxx`. In K8s: `kubectl logs <executor-pod> -n spark-jobs`. The JVM stack trace there is more complete than what the driver shows.
4. **Check memory first** — GC overhead, heap space, container killed, FetchFailed all trace back to memory. Check executor memory configuration before anything else.
5. **Check data skew second** — if one task is 10× slower than the median in the Tasks tab, it's skew. No config change fixes skew — you need salting, AQE, or a hot/cold split.
6. **Check the root cause, not the symptom** — `FetchFailedException` is almost never a network problem. It's an executor OOM that you need to trace back one step.