### Phase 1: Suspect There Is Blocking (First Signs)

You usually notice blocking indirectly — the system starts behaving strangely under load.

**Most common first symptoms** (in order of how often they appear):

1. **p99 / p95 latency jumps** suddenly (even though average is fine) → Users complain “sometimes it’s slow” → Grafana/Prometheus shows p99 from 200 ms → 2–5 seconds in spikes
2. **Vert.x logs show “Blocked thread” warnings** (gold — most common smoking gun) Example log you’ll see:
    
    text
    
    ```
    2026-02-28 17:45:12 WARN  io.vertx.core.impl.BlockedThreadChecker - Blocked thread: vertx-eventloop-thread-4
    java.lang.Thread.sleep(Thread.java:1234)
        at com.example.pricechecker.SlowPriceScraper.scrape(SlowPriceScraper.kt:89)
        ...
    Blocked time: 4123 ms
    ```
    
3. **Event-loop utilization spikes to 90–100%** on one or more event-loop threads (while overall CPU is not maxed) → Means one loop is stuck doing something slow
4. **Worker pool queue grows** (vertx.worker_pool.queue_size metric) → Blocking tasks are piling up faster than workers can handle them
5. **Throughput drops** even though CPU/memory look fine → Classic sign of event-loop starvation

**Quick triage checklist** (do this in 30 seconds):

- Are there “Blocked thread” logs in the last hour? → Yes → go to Phase 2
- Is p99 latency > 1–2 seconds while p50 is normal? → Likely event-loop blocking
- Is worker queue growing steadily? → Too many blocking tasks submitted

### Phase 2: Confirm & Locate the Blocking Code (Most Important Step)

**Method 1: Rely on Vert.x Blocked Thread Warnings (80% of cases solved here)**

- Vert.x automatically detects blocking on event loops
- Default threshold: 2 seconds (configurable)
- Tune it lower in dev/staging to catch smaller blocks:

Kotlin

```
Vertx.vertx(VertxOptions()
    .setBlockedThreadCheckInterval(1000)           // check every 1 second
    .setMaxEventLoopExecuteTime(500_000_000)       // alert after 500 ms
    .setMaxEventLoopExecuteTimeUnit(TimeUnit.NANOSECONDS)
)
```

- When it triggers, the log includes:
    - Thread name (e.g. vertx-eventloop-thread-7)
    - Blocked duration
    - Full stack trace — usually points **exactly** to the line

**Common stack traces you’ll see**:

- java.lang.Thread.sleep → leftover debug code
- java.sql.Statement.executeQuery → sync JDBC
- com.fasterxml.jackson.databind.ObjectMapper.readValue → huge JSON
- java.net.HttpURLConnection.getInputStream → sync HTTP call
- java.io.FileInputStream.read → large file read

**Action**:

- Open the file & line in IDE
- Ask: “Is this on the event loop?” → Yes → offload

**Method 2: Live Metrics + Grafana Alerts**

Set up alerts (in 5 minutes):

PromQL examples:

- Blocked events: increase(vertx_eventloop_blocked_total[5m]) > 0
- Blocked time per second: rate(vertx_eventloop_blocked_time_seconds_total[5m]) > 0.1
- Alert rule: “Event loop blocked more than 5 times in 5 minutes” → notify Slack/Teams

When alert fires → immediately check which event-loop thread is blocked (metrics are per-thread in Vert.x Micrometer).

**Method 3: Thread Dumps (When Logs/Metrics Are Not Enough)**

- Command (fastest): jstack -l <pid> > thread-dump-$(date +%s).txt Run 5–10 times, 5–10 seconds apart during the slow period
- Or use Arthas (zero-downtime, production-safe):
    1. java -jar arthas-boot.jar
    2. thread -b → shows blocked threads
    3. thread vertx-eventloop-thread-3 → dump specific thread

Look for:

- vertx-eventloop-thread-X in **RUNNABLE** state but stuck in native calls (socketRead0, readBytes, etc.)
- Many **WAITING** / **BLOCKED** threads in worker pool → backlog

### Phase 3: Reproduce & Isolate (Fast Feedback Loop)

1. **Lower detection threshold** to 100–300 ms in dev/staging → Catches even small blocks
2. **Add timing logs** around suspect code:
    
    Kotlin
    
    ```
    val start = System.nanoTime()
    val result = potentiallySlowCall()
    val tookMs = (System.nanoTime() - start) / 1_000_000
    if (tookMs > 100) {
        logger.warn("Slow call took {} ms: {}", tookMs, result)
    }
    ```
    
3. **Run load test** (k6, Gatling, Apache Bench) → Send 100–500 concurrent requests → watch logs for warnings
4. **Use async-profiler** (low overhead CPU profiler):
    
    Bash
    
    ```
    ./profiler.sh -d 30 -f profile.html <pid>
    ```
    
    Open in browser → flame graph → look for methods taking >5–10% CPU on event-loop threads

### Phase 4: Fix & Verify

**Most common fixes** (in order of frequency):

|Blocking Code Pattern|Quick Fix|Long-Term Fix|
|---|---|---|
|Sync JDBC / JPA / JdbcTemplate|Wrap in vertx.executeBlocking { ... }|Switch to vertx-pg-client or R2DBC|
|Sync HTTP (HttpURLConnection, OkHttp sync)|Replace with WebClient + sendAwait()|Always use Vert.x WebClient|
|Large JSON (Gson/Jackson on big payload)|executeBlocking or streaming parser|Use Jackson JsonParser (streaming)|
|File I/O (Files.readAllBytes, large CSV)|vertx.fileSystem().readFileAwait()|Use Vert.x async file system|
|CPU-heavy (crypto, image processing)|Deploy as worker verticle|Offload to separate microservice|
|Thread.sleep / debug waits|Replace with vertx.setTimer|Remove completely|

**After fix**:

- Deploy → run load test again
- Confirm: no more blocked warnings, p99 stable, worker queue flat
- Add metric alert if blocked count > 0

In practice, **Vert.x blocked-thread logs** solve most cases — teams that monitor them rarely have mystery latency issues.

### Resolving Common Blocking Code Patterns in Vert.x

In Vert.x (or any event-driven framework), blocking code on the event-loop thread is a common issue that causes latency spikes. The general fix is to **offload blocking operations** to the worker pool using vertx.executeBlocking(), or better yet, switch to non-blocking alternatives (e.g., async clients). I'll explain how to resolve each of your examples, with code snippets in Kotlin + Vert.x. Then, I'll add a few more common examples.

For all fixes, assume you're inside a CoroutineVerticle (using suspend and await() for clean code). If not using coroutines, replace await() with onSuccess { ... } callbacks.

#### 1. java.sql.Statement.executeQuery → Sync JDBC (Classic Blocking DB Call)

**Why it's blocking**: Traditional JDBC is synchronous — the thread waits for the DB response, stalling the event loop.

**Resolution**:

- **Short-term fix**: Wrap in executeBlocking (offloads to worker pool).
- **Long-term fix**: Switch to Vert.x's async SQL client (vertx-pg-client for PostgreSQL, vertx-mysql-client, etc.) or R2DBC for reactive JDBC.

**Code Example (Short-term)**:

Kotlin

```
suspend fun syncJdbcQuery(id: Long): RowSet<Row> {
    return vertx.executeBlockingAwait { promise ->
        try {
            val conn = DriverManager.getConnection("jdbc:postgresql://localhost/mydb", "user", "pass")
            val stmt = conn.createStatement()
            val rs = stmt.executeQuery("SELECT * FROM users WHERE id = $id")
            // Convert ResultSet to Vert.x RowSet (or your model)
            val rowSet = convertToRowSet(rs)  // Custom helper
            promise.complete(rowSet)
        } catch (e: Exception) {
            promise.fail(e)
        } finally {
            conn.close()
        }
    }
}
```

**Code Example (Long-term — Async)**:

Kotlin

```
// In start(): init PgPool
val pgPool = PgBuilder.pool(vertx, connectOptions, poolOptions)

suspend fun asyncQuery(id: Long): RowSet<Row> {
    return pgPool.preparedQuery("SELECT * FROM users WHERE id = $1")
        .executeAwait(Tuple.of(id))
}
```

#### 2. com.fasterxml.jackson.databind.ObjectMapper.readValue → Huge JSON Parsing

**Why it's blocking**: Jackson's readValue is CPU-bound and memory-intensive for large payloads — it can take seconds on big JSON, stalling the event loop.

**Resolution**:

- **Short-term fix**: Offload to executeBlocking (worker pool).
- **Long-term fix**: Use Jackson's streaming API (JsonParser) to parse incrementally (non-blocking). Or switch to Kotlinx Serialization (faster, but still offload if huge).

**Code Example (Short-term)**:

Kotlin

```
suspend fun parseHugeJson(jsonString: String): JsonObject {
    return vertx.executeBlockingAwait { promise ->
        try {
            val mapper = ObjectMapper()
            val tree = mapper.readTree(jsonString)
            promise.complete(JsonObject(tree.toString()))  // Convert to Vert.x Json
        } catch (e: Exception) {
            promise.fail(e)
        }
    }
}
```

**Code Example (Long-term — Streaming)**:

Kotlin

```
suspend fun parseStreamingJson(input: Buffer): JsonObject {  // Vert.x Buffer
    val mapper = ObjectMapper()
    val parser = mapper.factory.createParser(input.bytes)
    
    return withContext(Dispatchers.IO) {  // Offload if parsing is heavy
        val tree = mapper.readTree(parser)
        JsonObject(tree.toString())
    }
}
```

#### 3. java.net.HttpURLConnection.getInputStream → Sync HTTP Call

**Why it's blocking**: getInputStream() waits for the full response, blocking the thread on network I/O.

**Resolution**:

- **Short-term fix**: Wrap in executeBlocking.
- **Long-term fix**: Use Vert.x's non-blocking WebClient (built on Netty) for all HTTP calls.

**Code Example (Short-term)**:

Kotlin

```
suspend fun syncHttpGet(url: String): Buffer {
    return vertx.executeBlockingAwait { promise ->
        try {
            val conn = URL(url).openConnection() as HttpURLConnection
            conn.requestMethod = "GET"
            val input = conn.inputStream
            val body = input.readBytes()
            promise.complete(Buffer.buffer(body))
        } catch (e: Exception) {
            promise.fail(e)
        } finally {
            conn.disconnect()
        }
    }
}
```

**Code Example (Long-term — Async)**:

Kotlin

```
val webClient = WebClient.create(vertx)

suspend fun asyncHttpGet(url: String): Buffer {
    return webClient.getAbs(url)
        .sendAwait()
        .body()
}
```

#### 4. java.io.FileInputStream.read → Large File Read

**Why it's blocking**: read() waits for disk I/O, which can be slow for large files (hundreds of MB).

**Resolution**:

- **Short-term fix**: Offload to executeBlocking.
- **Long-term fix**: Use Vert.x's async file system API (vertx.fileSystem()) for non-blocking reads.

**Code Example (Short-term)**:

Kotlin

```
suspend fun syncFileRead(path: String): Buffer {
    return vertx.executeBlockingAwait { promise ->
        try {
            val input = FileInputStream(path)
            val bytes = input.readBytes()
            promise.complete(Buffer.buffer(bytes))
        } catch (e: Exception) {
            promise.fail(e)
        } finally {
            input.close()
        }
    }
}
```

**Code Example (Long-term — Async)**:

Kotlin

```
suspend fun asyncFileRead(path: String): Buffer {
    return vertx.fileSystem().readFileAwait(path)
}
```

### More Examples of Blocking Code and Resolutions

**Example 5: Thread.sleep or TimeUnit.sleep (Debug / Wait Logic)**

**Why blocking**: Obviously blocks the thread for the duration.

**Resolution**:

- Replace with Vert.x's non-blocking timer.
- **Code**:
    
    Kotlin
    
    ```
    suspend fun asyncSleep(ms: Long) {
        vertx.timer(ms).await()
    }
    ```
    

**Example 6: Large Collection Processing (e.g., list.sortBy on 1M items)**

**Why blocking**: CPU-bound — takes seconds on event loop.

**Resolution**:

- Offload to executeBlocking or use worker verticle for repeated heavy tasks.
- **Code**:
    
    Kotlin
    
    ```
    suspend fun sortLargeList(list: MutableList<Item>): List<Item> {
        return vertx.executeBlockingAwait { promise ->
            list.sortBy { it.price }
            promise.complete(list)
        }
    }
    ```
    

**Example 7: Sync Encryption / Decryption (e.g., Cipher.doFinal on large data)**

**Why blocking**: CPU-intensive for big payloads.

**Resolution**:

- Offload to worker, or use non-blocking crypto libs if available.
- **Code** (offload):
    
    Kotlin
    
    ```
    suspend fun encryptLargeData(data: Buffer): Buffer {
        return vertx.executeBlockingAwait { promise ->
            val cipher = Cipher.getInstance("AES/GCM/NoPadding")
            cipher.init(Cipher.ENCRYPT_MODE, key)
            val encrypted = cipher.doFinal(data.bytes)
            promise.complete(Buffer.buffer(encrypted))
        }
    }
    ```
    

**Example 8: External Command Execution (e.g., ProcessBuilder.start().waitFor)**

**Why blocking**: waitFor blocks until subprocess finishes.

**Resolution**:

- Use Vert.x's Process API for async execution.
- **Code**:
    
    Kotlin
    
    ```
    suspend fun runExternalCommand(cmd: String): String {
        return vertx.executeBlockingAwait { promise ->
            val process = Runtime.getRuntime().exec(cmd)
            val output = process.inputStream.readBytes().toString(Charsets.UTF_8)
            process.waitFor()
            promise.complete(output)
        }
    }
    ```
    

### General Tips for Resolutions

- **Always measure before/after**: Use Micrometer metrics to track handler time (vertx_http_server_request_duration_seconds).
- **Pattern for offloading**: Use executeBlockingAwait in coroutines — clean and safe.
- **Prevention**: In code reviews, flag any import of java.sql.*, java.net.HttpURLConnection, java.io.FileInputStream inside normal verticles.
- **Edge case**: If blocking is from GC pauses (not code), tune JVM heaps (-Xmx, -XX:+UseG1GC).