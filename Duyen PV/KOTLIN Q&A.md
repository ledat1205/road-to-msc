Here are **model answers** for the **19 Kotlin-focused middle-level interview questions** I prepared earlier. These answers are written in a natural, confident style that you can adapt to your own experience (e.g., mentioning MoMo, Tiki, or TMA). They are concise enough for verbal delivery but detailed enough to show depth.

Practice saying them aloud — aim for 1–2 minutes per answer. Feel free to personalize with your real stories.

### Kotlin Language & Idioms

**1. In production Kotlin code, when do you still use var instead of val? Give a concrete example.**

**Answer**: I default to val everywhere because immutability prevents accidental mutations, makes code thread-safe by default, and improves readability.

I only use var in these practical cases:

- Building mutable collections before converting to immutable (e.g., val temp = mutableListOf<T>() … return temp.toList())
- In performance-sensitive loops where re-assignment avoids extra allocations
- In builder patterns or DSLs (e.g., configuring OkHttpClient or Vert.x Router)

Real example from MoMo: When migrating minigame metadata from HBase to PostgreSQL, we had a batch builder class. We used var batchSize = 0 and var currentBatch = mutableListOf<Row>() inside a loop — it was clearer and avoided creating new lists every iteration. Once the batch was full, we passed currentBatch.toList() to the sink. Code review enforced: var only when mutation is intentional and scoped.

**2. How does Kotlin null-safety work? Show 3–4 different ways to safely handle a nullable value coming from a Java library or external JSON.**

**Answer**: Kotlin prevents NullPointerExceptions at compile time: non-nullable types (String) can't hold null; nullable types use ? (String?).

Safe handling patterns I use daily:

1. Safe call + Elvis:

Kotlin

```
val name = json?.user?.name ?: "Guest"
```

2. Safe call + let (scoped non-null):

Kotlin

```
json?.user?.let { user ->
    processUser(user.name, user.email)
}
```

3. requireNotNull / checkNotNull (fail fast):

Kotlin

```
val token = requireNotNull(headers["Authorization"]) { "Missing auth header" }
```

4. run + safe calls (for chaining):

Kotlin

```
val result = response.run { data?.items?.firstOrNull()?.id } ?: -1
```

From TMA ingestion pipelines: When parsing third-party JSON, we used let chains + Elvis defaults to avoid !! entirely — reduced runtime crashes by ~70% in one sprint.

**3. What is the difference between data class, class, and object? When would you choose one over the others?**

**Answer**:

- data class: For immutable data holders. Auto-generates equals(), hashCode(), toString(), copy(), componentN(). Use when you mainly care about data equality/value objects (DTOs, domain models).
- class: General-purpose. No auto-generated methods. Use for behavior-heavy classes, services, stateful objects.
- object: Singleton (one instance per JVM). Use for utilities, singletons, companion-like global state.

Choices:

- DTO from API → data class
- Service/repository → class (or object if stateless singleton)
- Factory/Logger → object

In MoMo Vert.x services, we used data class for event payloads (e.g., data class TransactionCreated(val id: String, val amount: Double)), class for repositories, and object for config loaders.

**4. Explain extension functions. Give a real-world example where you used (or would use) an extension function instead of a utility class.**

**Answer**: Extension functions add methods to existing classes without inheritance or modifying source code. Syntax: fun ReceiverType.extensionName(params): ReturnType { ... }.

Benefits: Cleaner API, better discoverability, no wrapper classes.

Real example: At Tiki, we had frequent PostgreSQL timestamp handling. Instead of DateUtils.toUtcString(timestamp), we added:

Kotlin

```
fun Instant.toUtcIsoString(): String = this.atZone(ZoneOffset.UTC).toString()
```

Usage: event.timestamp.toUtcIsoString() — more readable, discoverable via IDE, no static utils class clutter.

Another common one: fun String?.ifNotBlank(block: (String) -> Unit) for safe string processing.

**5. What are sealed classes / sealed interfaces? Why are they useful compared to regular enums or open classes?**

**Answer**: Sealed classes/interfaces restrict inheritance: all subclasses must be declared in the same file (or nested). Compiler knows all possible types at compile time → exhaustive when without else.

Compared to enums: sealed classes can hold state/data (not just constants). Compared to open classes: no unknown subclasses → safer pattern matching.

Use cases:

- Result/Error handling: sealed class ApiResult<out T> { data class Success<T>(val data: T) : ApiResult<T>(), data class Error(val message: String) : ApiResult<Nothing>() }
- State machines: sealed interface PaymentState { object Idle : PaymentState, data class Processing(val amount: Double) : PaymentState, ... }

In MoMo, we used sealed classes for event outcomes — when (result) { is Success -> … is Failure -> … } — compiler enforced all cases.

**6. Compare let, run, with, apply, and also. Give a situation for each where it is the most idiomatic choice.**

**Answer**: All are scope functions — they execute a block with an object as receiver (this) or argument (it).

- let (lambda with it, returns lambda result): null-safe transform / scoping → json?.let { process(it) } ?: default
- run (lambda with this, returns lambda result): combine null-check + configuration → user?.run { name = "New"; save() }
- with (non-extension, this, returns lambda result): group calls on non-null object → with(builder) { url = "..."; timeout = 30; build() }
- apply (lambda with this, returns receiver): configure and return same object → val client = OkHttpClient.Builder().apply { addInterceptor(...); timeout(...) }.build()
- also (lambda with it, returns receiver): side effects / logging → list.also { logger.info("Processing $it") }.map { ... }

Idiomatic choices:

- Null-safe mapping → let
- Builder pattern → apply
- Side effect/logging → also
- Grouping calls on external object → with

**7. How do you implement a singleton in Kotlin? What is the difference between object, companion object, and lazy initialization?**

**Answer**:

- object Singleton { ... } → JVM singleton (one instance per class loader). Thread-safe by default. Best for pure singletons (utils, configs).
- companion object → static-like members inside a class. Not a singleton instance unless you make it one.
- Lazy initialization: val instance by lazy { ExpensiveObject() } — thread-safe, lazy, inside class or top-level.

Preferred:

- Global singleton → object
- Class with singleton instance → companion object { val instance by lazy { ... } }

In Vert.x services, I use object Config { val env = System.getenv("ENV") } — simple, no DI overhead.

### Collections & Functional Style

**8. Explain the difference between List<T>, MutableList<T>, ArrayList<T>, listOf(), mutableListOf() and arrayListOf(). Which ones do you prefer in API signatures and why?**

**Answer**:

- List<T>: Read-only interface (immutable view)
- MutableList<T>: Read-write interface
- ArrayList<T>: Concrete mutable implementation (backed by array)

Creation functions:

- listOf() → immutable List (backed by ArrayAsList)
- mutableListOf() → MutableList (usually ArrayList)
- arrayListOf() → explicitly ArrayList

In API signatures (return types, parameters):

- Prefer List<T> (immutable) for inputs/outputs — protects callers, communicates intent
- Use MutableList<T> only when caller must mutate (rare)
- Never expose ArrayList — leaks implementation

Example: fun getUsers(): List<User> — caller can't mutate. At Tiki, this prevented bugs in recommendation pipelines.

**9. Write (or describe) a one-liner that groups a list of transactions by userId and calculates the total amount per user using functional style.**

**Answer**:

Kotlin

```
val totals = transactions
    .groupBy { it.userId }
    .mapValues { (_, txs) -> txs.sumOf { it.amount } }
```

Or more concise with associate:

Kotlin

```
val totals = transactions.associateBy { it.userId }
    .mapValues { (_, tx) -> transactions.filter { it.userId == tx.userId }.sumOf { it.amount } } // less efficient
```

Better: groupingBy + aggregate for single pass:

Kotlin

```
val totals = transactions.groupingBy { it.userId }
    .aggregate { _, acc: Double?, tx, first -> (acc ?: 0.0) + tx.amount }
```

I used similar in Tiki for per-user spend aggregation — groupingBy + aggregate was 2× faster than groupBy + map.

**10. What is the difference between map, flatMap, mapNotNull, filterNotNull, and associate? When would you choose flatMap over map + flatten?**

**Answer**:

- map: 1:1 transform (List → List**)**
**- flatMap: 1:many transform + flatten (List → List**)**
**- mapNotNull: map + skip null results
- filterNotNull: remove nulls
- associate: create Map<K,V> from list (key selector + value selector)****

****

flatMap vs map + flatten:

- flatMap is more efficient (single pass) and idiomatic
- map { ... }.flatten() creates intermediate list → more allocations

Choose flatMap when each element produces a collection (e.g., splitting strings, API calls returning lists).

Example:

Kotlin

```
users.flatMap { user -> user.orders }  // better than users.map { it.orders }.flatten()
```

**11. How do you efficiently remove duplicates from a list while preserving insertion order? What about when you need to keep only the first occurrence of each item?**

**Answer**: Preserve order + remove duplicates:

Kotlin

```
val unique = list.distinct()                    // uses equals/hashCode, preserves order
```

Or if custom equality:

Kotlin

```
val seen = mutableSetOf<Key>()
val unique = list.filter { seen.add(it.key) }   // keeps first occurrence
```

For large lists:

Kotlin

```
val unique = list.associateBy { it.id }.values.toList()  // preserves order, keeps first
```

distinct() is O(n), uses hash set internally — very efficient.

### Coroutines & Concurrency

**12. What is a suspend function? How is it different from a normal function under the hood?**

**Answer**: suspend marks a function that can be paused/resumed without blocking the thread. It can call other suspend functions and use suspension points (e.g., delay, await).

Under the hood: Kotlin compiler turns suspend functions into state machines (continuations). Each suspension point saves state and returns a Continuation. No OS thread is blocked — coroutine is suspended and can resume on any thread.

Difference from normal function:

- Can't call suspend from non-suspend
- Transformed to CPS (continuation-passing style)
- Enables structured concurrency

**13. Explain the difference between launch, async, withContext, coroutineScope, and supervisorScope. When do you use supervisorScope instead of coroutineScope?**

**Answer**:

- launch: Fire-and-forget coroutine (returns Job)
- async: Deferred result (returns Deferred<T>) — use await()
- withContext: Change dispatcher for block, wait for completion
- coroutineScope: Create child scope, wait for all children, propagate exceptions
- supervisorScope: Like coroutineScope, but exceptions in one child don't cancel siblings

Use supervisorScope when partial failures are acceptable (e.g., fetch multiple independent resources — one fails, others continue).

Example:

Kotlin

```
suspend fun loadDashboard() = supervisorScope {
    val user = async { fetchUser() }
    val orders = async { fetchOrders() } // if this fails, user still completes
    combine(user.await(), orders.awaitOrNull())
}
```

**14. How do you handle exceptions in coroutines so that one failing task does not cancel all siblings?**

**Answer**: Use supervisorScope + async (or launch) + await with try-catch or awaitOrNull.

Pattern:

Kotlin

```
suspend fun fetchAll() = supervisorScope {
    val results = listOf("A", "B", "C").map { key ->
        async {
            try {
                fetch(key)
            } catch (e: Exception) {
                logger.error("Failed $key", e)
                null
            }
        }
    }

    results.awaitAll() // or map { it.awaitOrNull() }
}
```

CoroutineExceptionHandler can be installed on scope/job for global handling, but supervisorScope is preferred for structured partial failure.

**15. What are the main Dispatchers (Default, IO, Main, Unconfined)? Which one should you almost never use in production code and why?**

**Answer**:

- Dispatchers.Default: CPU-bound work (parsing, calculations). Pool size = CPU cores.
- Dispatchers.IO: I/O-bound (network, disk, DB). Starts with 64 threads, grows if needed.
- Dispatchers.Main: UI thread (Android only).
- Dispatchers.Unconfined: Runs on caller thread, no switch. No thread confinement.

Unconfined should almost never be used in production:

- Breaks structured concurrency guarantees
- Can leak coroutines across threads
- Makes debugging hard (no predictable thread)

Only use in tests or very specific low-level cases. In Vert.x, I use vertx.dispatcher() instead to stay on event loop.

**16. You need to run 100 network requests in parallel but limit concurrency to 10 at a time. How would you implement this with coroutines?**

**Answer**: Use coroutineScope + channelFlow / semaphore / limitedParallelism.

Cleanest modern way (Kotlin 1.7+):

Kotlin

```
suspend fun fetchAll(ids: List<String>): List<Result<String>> = coroutineScope {
    ids.chunked(10).flatMap { chunk ->
        chunk.map { id ->
            async { fetchWithRetry(id) }
        }.awaitAll()
    }
}
```

Or with explicit concurrency limit using limitedParallelism:

Kotlin

```
val limited = Dispatchers.IO.limitedParallelism(10)

suspend fun fetchAll(ids: List<String>) = coroutineScope {
    ids.map { id ->
        async(limited) { fetch(id) }
    }.awaitAll()
}
```

Both approaches keep max 10 concurrent requests. I used the chunked + async pattern in Tiki for parallel ClickHouse queries — controlled load without overwhelming the cluster.

### Kotlin in Backend / Production

**17. How do you usually handle serialization / deserialization in Kotlin backend services? (Kotlinx Serialization vs Gson vs Jackson vs Moshi) What are the trade-offs?**

**Answer**: I prefer **Kotlinx Serialization** in new Kotlin-first services because:

- Type-safe (compile-time checks)
- Null safety integrated
- Kotlin idioms (data classes, sealed) work out-of-box
- No reflection by default (faster, smaller binary)
- Good Vert.x/Ktor integration

Trade-offs:

- Kotlinx: Best for pure Kotlin, but less mature ecosystem than Jackson, fewer custom adapters
- Jackson: Most mature, huge ecosystem, good Java interop, but reflection-heavy, null-safety issues
- Gson: Simple, lightweight, but reflection-based, weak Kotlin support (needs annotations)
- Moshi: Good balance, Kotlin-friendly, but smaller community

In MoMo Vert.x services, we switched to Kotlinx Serialization for internal events — reduced runtime errors and improved performance on high-throughput endpoints.

**18. You receive a large JSON payload from a third-party API. How do you parse it safely without OOM or blocking the thread for too long?**

**Answer**: Steps I follow:

1. Use streaming parser (Jackson JsonParser or Kotlinx JsonReader) — parse incrementally, not whole string
2. Offload to worker / Dispatchers.IO / executeBlocking
3. Validate size first (e.g., Content-Length header < 10MB)
4. Use mapNotNull / takeWhile to process only needed parts
5. Set timeouts on HTTP client

Example with Kotlinx (streaming):

Kotlin

```
suspend fun parseLargeJson(input: InputStream): List<Item> = withContext(Dispatchers.IO) {
    Json.decodeFromStream<List<Item>>(input) // streaming under the hood
}
```

Or Jackson:

Kotlin

```
val parser = JsonFactory().createParser(input)
while (parser.nextToken() != JsonToken.END_ARRAY) { ... }
```

In Tiki, we used Jackson streaming for large ClickHouse exports — avoided OOM and kept event loops free.

**19. In a high-throughput service, how do you avoid common coroutine pitfalls (leaked coroutines, context loss, wrong dispatcher usage)? Give 3–4 concrete rules or patterns you follow.**

**Answer**: Rules I enforce in code reviews:

1. **Never use GlobalScope**: always coroutineScope, viewModelScope, lifecycleScope, or custom scope — prevents leaks
2. **Always use correct dispatcher**: Dispatchers.IO for I/O, withContext(vertx.dispatcher()) in Vert.x for context propagation
3. **Structured concurrency everywhere**: wrap parallel work in coroutineScope or supervisorScope — automatic cleanup
4. **Avoid async without await**: if fire-and-forget needed, use launch + Job tracking

Patterns:

- supervisorScope for independent tasks
- CoroutineExceptionHandler on root scope for logging
- Job.join() in tests to ensure completion

In MoMo high-traffic services, we added linter rule against GlobalScope — reduced leaked coroutines from ~5% to near zero.

****