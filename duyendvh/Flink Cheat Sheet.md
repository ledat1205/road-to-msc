Here’s a **practical, concise Apache Flink cheat sheet** focused on the most commonly used parts in 2025–2026 (Flink 1.18–2.x era). It covers **DataStream API** (still widely used for custom streaming logic), **Table API / SQL** (now the recommended unified way for many pipelines), key concepts, and common patterns — especially useful for e-commerce recommendation systems like the ones we discussed earlier.

### 1. Core Concepts Quick Reference

| Concept              | Description                                                                 | Key Tip / When to Use                              |
|----------------------|-----------------------------------------------------------------------------|----------------------------------------------------|
| DataStream           | Unbounded / bounded stream of data                                          | Low-level control, custom state, CEP, timers       |
| KeyedStream          | Stream partitioned by key (`.keyBy(...)`)                                   | Required for stateful operations                   |
| State                | Managed (ValueState, ListState, MapState, etc.) or raw                     | Use managed for fault tolerance                    |
| Checkpoint / Savepoint | Exactly-once fault recovery                                                | Enable checkpoints: `env.enableCheckpointing(60000)` |
| Watermark            | Event-time progress indicator                                               | `forBoundedOutOfOrderness(Duration.ofSeconds(30))` |
| Window               | Tumbling, Sliding, Session, Global                                          | Event-time windows need watermarks                 |
| CEP                  | Complex Event Processing (pattern matching)                                 | Sequences, timeouts, cart abandonment              |
| Broadcast State      | Share read-only config/rules across all tasks                               | Dynamic rules, model parameters                    |

### 2. Quick Setup (Java – most common in production)

```java
// Minimal skeleton
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.enableCheckpointing(60000);           // 1 min checkpoints
env.setParallelism(4);                    // adjust per cluster

// For event time (almost always needed)
env.getConfig().setAutoWatermarkInterval(200);
```

### 3. Data Sources & Sinks (most used)

| Source / Sink          | Code Snippet (Java)                                                                                  | Notes                                      |
|------------------------|------------------------------------------------------------------------------------------------------|--------------------------------------------|
| Kafka (source)         | `KafkaSource.<String>builder().setBootstrapServers("...").setTopics("events").build()`              | Use `OffsetsInitializer.earliest()` or `latest()` |
| Kafka (sink)           | `KafkaSink.<String>builder().setBootstrapServers("...").setRecordSerializer(...).build()`           | Exactly-once with transactional producer  |
| Print (debug)          | `.print()`                                                                                           |                                            |
| File (bounded)         | `env.readTextFile("path")`                                                                           | For testing                                |
| From elements          | `env.fromElements(...).fromCollection(...)`                                                          | Quick prototyping                          |

### 4. Common Transformations (DataStream API)

| Operation              | Example                                                                                      | Notes                                      |
|------------------------|----------------------------------------------------------------------------------------------|--------------------------------------------|
| map / flatMap          | `.map(e -> e.toUpperCase())`                                                                 |                                            |
| filter                 | `.filter(e -> e.price > 100)`                                                                |                                            |
| keyBy                  | `.keyBy(e -> e.userId)`                                                                      | POJO / Tuple / KeySelector                 |
| reduce                 | `.reduce((a,b) -> a + b)`                                                                    | Per-key incremental                        |
| aggregate              | `.aggregate(new MyAggFunction())`                                                            | More flexible than reduce                  |
| window                 | `.window(TumblingEventTimeWindows.of(Time.minutes(5)))`                                      | + `.sum()`, `.max()`, custom AggregateFunc |
| process (rich)         | `.process(new MyProcessFunction())`                                                          | Timers, state, side outputs                |
| async I/O              | `AsyncDataStream.unorderedWait(stream, asyncFunc, timeout, capacity)`                        | Model inference, external lookups          |

### 5. Windows Quick Reference

```java
// Tumbling (non-overlapping)
.window(TumblingEventTimeWindows.of(Time.minutes(10)))

// Sliding (overlapping)
.window(SlidingEventTimeWindows.of(Time.minutes(10), Time.minutes(2)))

// Session (gap-based)
.window(SessionWindows.withGap(Time.minutes(15)))

// Global window (all data, custom trigger)
.window(GlobalWindows.create()).trigger(CountTrigger.of(1000))
```

### 6. State & Timers (very useful in rec systems)

```java
class MyProcess extends KeyedProcessFunction<String, Event, String> {
    private ValueState<UserProfile> profileState;
    private ListState<String> recentViews;

    @Override
    public void open(...) {
        profileState = getRuntimeContext().getState(new ValueStateDescriptor<>("profile", UserProfile.class));
        recentViews   = getRuntimeContext().getListState(new ListStateDescriptor<>("views", String.class));
    }

    @Override
    public void processElement(Event e, Context ctx, Collector<String> out) {
        // Update state
        recentViews.add(e.productId);

        // Register timer (event time)
        ctx.timerService().registerEventTimeTimer(ctx.timestamp() + 15 * 60 * 1000);
    }

    @Override
    public void onTimer(long ts, OnTimerContext ctx, Collector<String> out) {
        // Timeout logic → e.g. cart abandonment recs
    }
}
```

### 7. CEP (Complex Event Patterns) – Quick Examples

```java
Pattern<Event, ?> pattern = Pattern
    .<Event>begin("start")
        .where(SimpleCondition.of(e -> e.type.equals("ADD_TO_CART")))
    .followedBy("end")
        .where(SimpleCondition.of(e -> e.type.equals("PURCHASE")))
    .within(Time.minutes(30))
    .notFollowedBy("timeout");  // → timeout triggers match

CEP.pattern(stream.keyBy(e -> e.userId), pattern)
   .select(map -> "Abandoned cart for user " + map.get("start").get(0).userId);
```

### 8. Table API / SQL (recommended for most new pipelines)

```sql
-- Simple streaming SQL
CREATE TABLE events (
    user_id STRING,
    product_id STRING,
    ts TIMESTAMP(3),
    WATERMARK FOR ts AS ts - INTERVAL '5' SECOND
) WITH (...);

SELECT
    TUMBLE_END(ts, INTERVAL '5' MINUTE) AS window_end,
    product_id,
    COUNT(*) AS views
FROM events
GROUP BY TUMBLE(ts, INTERVAL '5' MINUTE), product_id;
```

### 9. Quick Tips for Production (2025–2026 style)

- Prefer **Table API / SQL** + **DataStream in BATCH mode** for new jobs (unified API).
- Use **RocksDB** state backend + incremental checkpoints for large state.
- Enable **Unaligned checkpoints** (since 1.11+) for very large state.
- Monitor backpressure via Flink UI / metrics.
- For ML inference: Async I/O or Flink ML / external gRPC.
- Common ports: JobManager 8081 (UI), TaskManager data 6122, RPC 6123.

### Best Quick-Start Resources (still relevant in 2026)

- Official docs: https://nightlies.apache.org/flink/flink-docs-stable
- Confluent Flink Java cheatsheet: https://github.com/confluentinc/learn-building-flink-applications-in-java-exercises/blob/main/cheatsheet.md
- Cheatography one-pager (PDF): https://cheatography.com/mliafol/cheat-sheets/flink/pdf

Bookmark these — they cover 80% of daily Flink work.

Which area do you use most (CEP, windows, SQL, state management, Kafka integration)? I can expand that section with more examples.