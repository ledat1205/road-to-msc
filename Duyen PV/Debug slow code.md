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