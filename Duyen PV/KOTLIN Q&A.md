
### Kotlin Language & Idioms

**1. When do you still use `var` instead of `val`?**

I strongly prefer `val` — immutability is safer and clearer. I only reach for `var` when mutation brings a clear, local benefit:

- Building collections before freezing them
- Performance-critical mutable accumulators in tight loops
- DSL / builder state

Example I’ve actually written:

```kotlin
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

Reusing the same `buffer` object avoided thousands of small list allocations per file.

**2. Null-safety – 4 practical safe-handling patterns**

```kotlin
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

In one ingestion pipeline we eliminated almost all `!!` usage — crash rate dropped noticeably.

**4. Extension function – real-world example**

Instead of utility classes we often do:

```kotlin
fun Instant.toUtcIso(): String =
    atZone(ZoneOffset.UTC).toString()

fun BigDecimal.roundToTwo(): BigDecimal =
    setScale(2, RoundingMode.HALF_UP)

fun String?.takeIfNotBlank(): String? =
    this?.takeUnless { it.isBlank() }

infix fun <T> T?.ifNull(default: T): T = this ?: default
```

Very common pattern: `event.timestamp.toUtcIso()`

**6. Scope functions – when to choose each**

```kotlin
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

```kotlin
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

```kotlin
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

```kotlin
// Prefer flatMap
users.flatMap { it.recentOrders }

// vs (worse)
users.map { it.recentOrders }.flatten()

// Another typical flatMap
val allTags = products
    .flatMap { it.categories.flatMap { cat -> cat.tags } }
```

**11. Remove duplicates preserving order**

```kotlin
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

```kotlin
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

```kotlin
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

```kotlin
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



