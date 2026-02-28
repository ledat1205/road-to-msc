### 1. List<T> / MutableList<T> (by far the #1 most used)

- Ordered collection that allows duplicates.
- Access by index (like arrays but dynamic/resizable).
- Most common implementations:
    - ArrayList<T> (default backing for mutableListOf() — fast random access, O(1) get/set)
    - LinkedList<T> (rarely used directly — better for frequent inserts/deletes in middle)

**When to use**:

- Almost everything: user lists, search results, API responses, event logs, etc.
- Prefer **immutable**List<T> in function signatures/return types.

Kotlin

```
val names: List<String> = listOf("Alice", "Bob", "Charlie")          // immutable
val scores = mutableListOf(85, 92, 78)                               // mutable
scores.add(95)
scores[1] = 90
```

### 2. Map<K, V> / MutableMap<K, V> (very close #2)

- Key-value pairs (dictionary/hash table).
- Keys are **unique**, values can duplicate.
- Most common implementations:
    - HashMap<K, V> (unordered, fastest average O(1) lookup/insert)
    - LinkedHashMap<K, V> (preserves insertion order — very common for JSON serialization)
    - TreeMap<K, V> (sorted by keys — rare unless you need ordering)

**When to use**:

- Caching, configs, grouping data, API params, user sessions, etc.

Kotlin

```
val userScores: Map<String, Int> = mapOf("Alice" to 95, "Bob" to 82)
val cache = mutableMapOf<String, String>()
cache["token"] = "xyz123"
val score = userScores["Alice"] ?: 0   // safe access
```

### 3. Set<T> / MutableSet<T>

- Collection of **unique** elements (no duplicates).
- Most common implementations:
    - HashSet<T> (unordered, fastest membership check O(1))
    - LinkedHashSet<T> (preserves insertion order)
    - TreeSet<T> (sorted — via Java interop)

**When to use**:

- Removing duplicates, tracking seen items, tags, permissions, unique IDs.

Kotlin

```
val uniqueIds: Set<Long> = setOf(1001, 1002, 1001)          // → {1001, 1002}
val activeUsers = mutableSetOf<String>()
activeUsers.add("alice42")
activeUsers.contains("bob99")   // fast check
```

### 4. Array<T>

- Fixed-size, primitive-friendly collection (not a Collection subclass).
- Better performance for primitives (no boxing) and when size is known & fixed.

**When to use**:

- Performance-critical code, interop with Java arrays, primitive collections.

Kotlin

```
val numbers = arrayOf(1, 2, 3)              // Array<Int>
val primitives = IntArray(5) { it * 10 }    // [0,10,20,30,40] — no boxing
```

### 5. Queue / Deque (less common but important)

- **Queue<T>** → FIFO (first-in-first-out)
    - Most used: ArrayDeque<T> (double-ended queue — very efficient)
- **Stack** → LIFO (last-in-first-out) — usually just use ArrayDeque with push/pop

**When to use**:

- Task queues, BFS (queue), undo/redo (stack), sliding window, etc.

Kotlin

```
val tasks = ArrayDeque<String>()
tasks.addLast("fetch data")     // enqueue
tasks.addFirst("priority task") // can act as deque
val next = tasks.removeFirst()  // dequeue
```

### 6. Other somewhat common ones (but not daily for most devs)

|Data Structure|Kotlin Type / Impl|Best For|Approx. Frequency|
|---|---|---|---|
|PriorityQueue|java.util.PriorityQueue|Tasks by priority (Dijkstra, etc.)|Medium (algorithms)|
|Pair / Triple|Pair<A,B>, Triple<A,B,C>|Simple tuples (return 2–3 values)|Common|
|Data class (custom)|Your own data class|DTOs, domain models|Extremely common|
|Sequence<T>|sequence { ... }|Lazy / infinite / large data streams|Growing (performance)|
|ArrayDeque<T>|(as above)|Queue + Stack + Deque in one|Increasing|

### Quick decision table (what most Kotlin devs use 95% of the time)

|Need|Choose this|Creation example|
|---|---|---|
|Ordered list, allow duplicates|List<T> / MutableList<T>|listOf(), mutableListOf()|
|Key → value lookup|Map<K,V> / MutableMap<K,V>|mapOf(), mutableMapOf()|
|Unique items only|Set<T> / MutableSet<T>|setOf(), mutableSetOf()|
|Fixed size, fast access|Array<T> or primitive arrays|arrayOf(), intArrayOf()|
|FIFO / LIFO / double-ended|ArrayDeque<T>|ArrayDeque()|
|Preserve insertion order + map|LinkedHashMap<K,V>|linkedMapOf()|

In practice (from interviews, code reviews, production at places like Tiki/MoMo):

- **List** + **Map** cover ~80–90% of use cases.
- Prefer immutable types (List, Map, Set) in APIs and return types.
- Use mutableXxxOf() only when you actually need to mutate.
### Kotlin Language & Idioms

**1. When do you still use var instead of val?**

I strongly prefer val — immutability is safer and clearer. I only reach for var when mutation brings a clear, local benefit:

- Building collections before freezing them
- Performance-critical mutable accumulators in tight loops
- DSL / builder state

Example I’ve actually written:

Kotlin

```
suspend fun processLargeFileInBatches(
    reader: BufferedReader,
    batchSize: Int = 500
): Flow<List<Row>> = flow {
    var buffer = mutableListOf<Row>()
    var count = 0

    reader.forEachLine { line ->
        val row = parseLine(line) ?: return@forEachLine
        buffer.add(row)
        count++

        if (count >= batchSize) {
            emit(buffer.toList())           // freeze & send
            buffer = mutableListOf()        // reuse allocation
            count = 0
        }
    }

    if (buffer.isNotEmpty()) emit(buffer)
}
```

Reusing the same buffer object avoided thousands of small list allocations per file.

**2. Null-safety – 4 practical safe-handling patterns**

Kotlin

```
// 1. Safe call + Elvis (most common default)
val displayName = userJson?.profile?.name ?: "Anonymous"

// 2. Scoped non-null with let
user?.let { u ->
    analytics.trackLogin(u.id, u.email)
    renderProfile(u)
} ?: showGuestUI()

// 3. Fail-fast in critical paths
val token: String = requireNotNull(request.headers["X-Auth-Token"]) {
    "Missing auth token → 401"
}

// 4. Chaining many levels safely
val firstActiveOrderId = response
    .body?.data
    ?.orders
    ?.filter { it.status == "ACTIVE" }
    ?.firstOrNull()
    ?.id ?: -1L
```

In one ingestion pipeline we eliminated almost all !! usage — crash rate dropped noticeably.

**4. Extension function – real-world example**

Instead of utility classes we often do:

Kotlin

```
fun Instant.toUtcIso(): String =
    atZone(ZoneOffset.UTC).toString()

fun BigDecimal.roundToTwo(): BigDecimal =
    setScale(2, RoundingMode.HALF_UP)

fun String?.takeIfNotBlank(): String? =
    this?.takeUnless { it.isBlank() }

infix fun <T> T?.ifNull(default: T): T = this ?: default
```

Very common pattern: event.timestamp.toUtcIso()

**6. Scope functions – when to choose each**

Kotlin

```
// let   → transform / safe call result
json?.user?.let { sendWelcomeEmail(it.email) }

// run   → configure + return result (less common now)
val user = User().run {
    name = "Anna"
    role = Role.ADMIN
    this   // or just implicit return
}

// with  → non-extension grouping of calls
with(httpClient) {
    connectTimeout = 5.seconds
    readTimeout    = 30.seconds
    followRedirects = true
}

// apply → initialize & return same object (builders)
val client = OkHttpClient.Builder()
    .addInterceptor(AuthInterceptor())
    .connectTimeout(10, TimeUnit.SECONDS)
    .apply { if (isDebug) addNetworkInterceptor(LoggingInterceptor()) }
    .build()

// also  → side-effect, then continue chain
val ids = getActiveIds()
    .also { logger.info("Processing ${it.size} items") }
    .map { enrich(it) }
```

**8. Collection types in API signatures**

Kotlin

```
// Good
interface OrderService {
    suspend fun getOrders(userId: Long): List<OrderDto>
    suspend fun findActiveOrders(): List<OrderDto>
}

// Risky / usually bad
suspend fun processOrders(orders: MutableList<Order>)   // caller can break invariants
suspend fun getOrders(): ArrayList<Order>              // leaks impl
```

**9. Group & sum – several styles**

Kotlin

```
// Classic
transactions.groupBy { it.userId }
    .mapValues { (_, txs) -> txs.sumOf { it.amount } }

// Single-pass with groupingBy + aggregate (often fastest)
val userTotals = transactions
    .groupingBy { it.userId }
    .aggregate { _, accumulator: Double?, element, first ->
        (accumulator ?: 0.0) + element.amount
    }

// Even shorter if you only want Map<Long, Double>
val totalsMap: Map<Long, Double> = transactions
    .groupingBy { it.userId }
    .fold(0.0) { acc, tx -> acc + tx.amount }
```

**10. flatMap vs map + flatten**

Kotlin

```
// Prefer flatMap
users.flatMap { it.recentOrders }

// vs (worse)
users.map { it.recentOrders }.flatten()

// Another typical flatMap
val allTags = products
    .flatMap { it.categories.flatMap { cat -> cat.tags } }
```

**11. Remove duplicates preserving order**

Kotlin

```
// Simple case – uses equals/hashCode
list.distinct()

// Keep first occurrence – custom key
val seen = mutableSetOf<String>()
val unique = items.filter { seen.add(it.transactionId) }

// Alternative – clean & preserves order
items.associateBy { it.transactionId }
    .values
    .toList()
```

**13. supervisorScope example**

Kotlin

```
suspend fun enrichMultipleEntities(ids: List<Long>): List<Enriched?> =
    supervisorScope {
        ids.map { id ->
            async {
                try {
                    enrichOne(id)
                } catch (e: Exception) {
                    logger.warn("Enrich failed for $id", e)
                    null
                }
            }
        }.awaitAll()
    }
```

One failure doesn’t kill the others.

**16. Limit concurrency to 10**

Two clean modern patterns:

Kotlin

```
// Pattern A – chunking (simple, very readable)
suspend fun fetchInParallelLimited(ids: List<String>): List<String> =
    coroutineScope {
        ids.chunked(10).flatMap { chunk ->
            chunk.map { id -> async { api.fetch(id) } }
                 .awaitAll()
        }
    }

// Pattern B – limitedParallelism dispatcher (cleaner when limit is fixed)
val limitedDispatcher = Dispatchers.IO.limitedParallelism(10)

suspend fun fetchInParallelLimited(ids: List<String>): List<String> =
    coroutineScope {
        ids.map { id ->
            async(limitedDispatcher) { api.fetch(id) }
        }.awaitAll()
    }
```

Both approaches were used in production — chunking is easier to understand, limitedParallelism is slightly more elegant when the limit is constant.

**18. Streaming large JSON safely**

Kotlin

```
// Kotlinx Serialization streaming style
suspend fun parseLargeResponse(stream: InputStream): Sequence<Item> =
    withContext(Dispatchers.IO) {
        Json.decodeToSequence<Item>(stream)
            .take(100_000)           // emergency guard
            // .filter { it.isValid() } if needed
    }

// Jackson streaming (more control)
suspend fun processLargeJson(stream: InputStream, consumer: (Item) -> Unit) =
    withContext(Dispatchers.IO) {
        val parser = JsonFactory().createParser(stream)
        if (parser.nextToken() != JsonToken.START_ARRAY) error("Not an array")

        while (parser.nextToken() != JsonToken.END_ARRAY) {
            val item = mapper.readValue<Item>(parser)
            consumer(item)
        }
    }
```