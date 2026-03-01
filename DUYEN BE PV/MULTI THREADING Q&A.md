Here are **refined, more detailed model answers** for the **20 questions** on multi-threading in Java/Kotlin + Vert.x.  
I’ve incorporated **realistic code examples** wherever it makes the explanation clearer and more interview-ready (especially for middle-level roles).  
The style is natural and confident — the kind you can say aloud in an interview, with practical ties to your experience (MoMo, Tiki, TMA, Vert.x usage).

### Java/Kotlin Threading Basics

**Q1. What is the difference between a process and a thread? Why do we prefer threads over processes in most backend applications?**

**Answer**:  
A **process** is an independent program running in its own isolated memory space (separate heap, stack, file descriptors). Creating or switching processes is expensive because the OS must copy memory tables, allocate new address space, etc.

A **thread** is a lightweight unit of execution inside a process. All threads in the same process **share the same memory** (heap, global variables), file handles, etc., but each has its own stack and registers. Creating/switching threads is much cheaper (no memory isolation).

We prefer threads in backend applications because:
- Lower overhead → can create thousands of threads (goroutines in Go, coroutines in Kotlin) without running out of memory
- Fast shared-memory communication → no need for slow IPC (pipes, sockets) between parts of the app
- Better resource utilization → one JVM process can use multiple CPU cores via multiple threads

Example: In Vert.x, one JVM process with 16 event-loop threads + 20 worker threads can handle 10,000+ concurrent HTTP requests.  
In contrast, running 10,000 separate processes would consume massive RAM and slow everything down.

At Tiki, we used one Vert.x JVM process with multiple verticle instances to scale across cores — much more efficient than spawning many small processes.

**Q2. Explain what happens when two threads access the same non-volatile field without synchronization.**

**Answer**:  
Without synchronization, you get a **data race** — undefined behavior according to the Java Memory Model (JMM).

Possible (and very real) outcomes:
- **Stale reads**: Thread A writes `flag = true`, but Thread B keeps seeing `false` forever (caching)
- **Torn reads**: Thread A writes a 64-bit `long` value, Thread B sees half old + half new (partial write)
- **Reordered operations**: Compiler/JVM can reorder instructions → Thread A does `flag = true; data = 42`, Thread B sees `data = 42` but `flag = false`

Classic example:
```java
class FlagExample {
    boolean flag = false;          // non-volatile
    int data = 0;

    void writer() {
        data = 42;
        flag = true;               // no guarantee B sees this order
    }

    void reader() {
        if (flag) {                // might be false even after writer finished
            System.out.println(data); // could print 0 or garbage
        }
    }
}
```

Fixes:
- `volatile boolean flag` → visibility + no reordering
- `synchronized` block
- `AtomicBoolean`
- In Kotlin coroutines: use structured scopes or `Mutex` instead of raw threads

In practice, I’ve seen this cause intermittent bugs in shared counters — fixed by switching to `AtomicInteger`.

**Q3. What does the volatile keyword guarantee in Java/Kotlin? When is it enough? When is it not?**

**Answer**:  
`volatile` guarantees three things (Java Memory Model):
1. **Visibility**: A write to a volatile field is immediately visible to all other threads  
2. **Happens-before**: Any write to volatile → subsequent read of same volatile sees it (and all prior writes)  
3. **No reordering**: Compiler/JVM cannot reorder code around volatile reads/writes

In Kotlin: same as Java (`@Volatile` on fields).

**When it is enough**:
- Simple flags (shutdown, configuration change)  
- Single-writer, many-readers patterns (e.g., `volatile boolean isRunning = true`)

Example:
```kotlin
@Volatile
private var isShutdown = false

fun shutdown() {
    isShutdown = true
}

fun loop() {
    while (!isShutdown) { ... }  // guaranteed to eventually see true
}
```

**When it is NOT enough**:
- Compound actions (read → check → write) — e.g. incrementing a counter  
  ```java
  volatile int count = 0;
  void increment() { count++; } // NOT atomic — race condition
  ```
  Fix: `AtomicInteger`, `synchronized`, or `Mutex` in coroutines

I used `volatile` for shutdown flags in Vert.x services at MoMo — safe and cheap. For counters, we always used `Atomic*` or Event Bus + single verticle.

**Q4. Explain the difference between synchronized method/block and ReentrantLock. When would you choose one over the other?**

**Answer**:  
Both provide mutual exclusion (only one thread in critical section).

**`synchronized`** (monitor lock):
- Built-in keyword — simple syntax
- Automatically releases lock on exception (implicit finally)
- Locks on any object reference (or class for static)
- No advanced features (timeout, interrupt, fairness)

**`ReentrantLock`** (java.util.concurrent):
- Explicit `lock()` / `unlock()` (must use try-finally)
- More features:
  - `tryLock()` — non-blocking attempt
  - `lockInterruptibly()` — can be interrupted while waiting
  - `newCondition()` — multiple wait/notify conditions
  - Fair/unfair mode (fair = FIFO order)

Example:
```kotlin
val lock = ReentrantLock(true) // fair

lock.lockInterruptibly()
try {
    // critical section
} finally {
    lock.unlock()
}
```

**When to choose**:
- `synchronized` — simple, short critical sections, no need for timeouts or fairness
- `ReentrantLock` — need tryLock, interruptible wait, fairness, or multiple conditions (classic producer-consumer with bounded buffer)

In modern code, I prefer higher-level constructs:
- Kotlin coroutines → `Mutex`
- Vert.x → Event Bus or single verticle instance
- Java → `ConcurrentHashMap`, `CopyOnWriteArrayList`

Raw locks are rare now unless very specific performance needs.

**Q5. What is a deadlock? How can you prevent or detect it in production?**

**Answer**:  
A **deadlock** occurs when two or more threads are stuck forever waiting for each other to release locks in a circular chain.

Classic example:
```java
Thread A: lock1.lock(); lock2.lock(); ...
Thread B: lock2.lock(); lock1.lock(); ...
```
→ A holds lock1 waiting for lock2, B holds lock2 waiting for lock1 → both stuck.

**Prevention** (best defense):
1. **Global lock ordering** — always acquire locks in the same order (e.g., by ID or alphabetical name)
2. **Timeout locks** — use `tryLock(timeout)` instead of `lock()`
3. **Avoid nested locking** when possible — refactor code
4. **Use higher-level primitives** — `ConcurrentHashMap`, `BlockingQueue`, coroutines `Mutex`, Vert.x Event Bus

**Detection in production**:
- **Thread dumps** — `jstack <pid>` or VisualVM → look for “deadlock detected” or circular wait chains
  Example output:
  ```
  Found one Java-level deadlock:
  Thread-1: waiting to lock Monitor@0x1234 owned by Thread-2
  Thread-2: waiting to lock Monitor@0x5678 owned by Thread-1
  ```
- **Monitoring tools** — Datadog, New Relic, Prometheus + jmx-exporter expose `jvm.threads.deadlocked` metric
- **Java 9+** — `ThreadMXBean.findDeadlockedThreads()` in monitoring agent

In Vert.x + Kotlin services, deadlocks are extremely rare — we almost never use shared locks; we use Event Bus or single verticle instances.

**Q6. What is the Java Memory Model (JMM)? Why do we care about it in concurrent code?**

**Answer**:  
The **Java Memory Model (JMM)** is the official specification that defines **how and when** changes made by one thread become visible to other threads, and how operations can be reordered.

Key guarantees:
- **Happens-before relationships** — partial ordering that ensures visibility:
  - Unlock monitor → subsequent lock of same monitor
  - Write to `volatile` → subsequent read of same volatile
  - Thread.start() → actions in started thread
  - All actions in thread → Thread.join() return
- **No out-of-thin-air values** — you can’t see impossible values
- **Final fields** — safe publication after constructor finishes

Why we care:
- Without JMM rules, compilers/JVM can aggressively reorder instructions or cache values → threads see stale or impossible data
- Classic bug: Thread A writes `data = 42; flag = true` (volatile flag), Thread B sees `flag = true` but `data = 0` (reordering)

In practice:
- `volatile` + happens-before fixes visibility
- `synchronized` gives stronger guarantees
- In Kotlin + Vert.x, we avoid most JMM headaches by:
  - Using coroutines (structured, dispatcher-bound)
  - Atomic classes
  - Immutable messages on Event Bus

**Q7. Explain Vert.x's threading model in detail. What is the single-threaded execution guarantee?**

**Answer**:  
Vert.x uses two main thread pools:
- **Event-loop threads** — default size 2 × CPU cores (configurable)  
  Handle all non-blocking work: HTTP requests, timers, Event Bus handlers, async I/O completion
- **Worker threads** — default size 20 (configurable)  
  For blocking/long-running code (worker verticles, `executeBlocking`)

**Single-threaded execution guarantee**:
- Every **standard (non-worker) verticle instance** is pinned to **exactly one event-loop thread** for its entire life  
- Vert.x **never** calls two methods on the same verticle instance concurrently  
- All your handler code, timers, Event Bus consumers for that verticle run on the **same event-loop thread**

Result: you can write non-thread-safe code inside a verticle (no `synchronized`, no `volatile` for internal state) — as long as you don’t share mutable state with other verticles without protection.

Example:
```kotlin
class CounterVerticle : CoroutineVerticle() {
    private var count = 0  // safe — no concurrent access

    override suspend fun start() {
        vertx.eventBus().consumer<String>("increment") {
            count++  // safe — always same thread
            it.reply(count)
        }
    }
}
```

**Q8. What happens if you perform a blocking operation (e.g., Thread.sleep, sync JDBC) inside a normal Vert.x verticle?**

**Answer**:  
The **event-loop thread** that owns the verticle gets blocked.  
All other events assigned to that same event loop (other requests, timers, network reads) are delayed or starved — latency spikes, timeouts, degraded throughput for unrelated work.

Vert.x detects this automatically:
- After ~2 seconds of blocking → logs warning:
  ```
  Blocked thread: vertx-eventloop-thread-3
  java.lang.Thread.sleep(Thread.java:...)
      at com.example.SlowVerticle.slowHandler(SlowVerticle.kt:45)
  Blocked time: 3123 ms
  ```

**Effects**:
- One blocked loop → affects ~1/16th of traffic (on 8-core machine)  
- Multiple blocks → entire server slows down

**Fixes**:
- Wrap in `vertx.executeBlocking { ... }`
- Move to worker verticle
- Use async/reactive client (vertx-pg-client, WebClient)

Real case: At Tiki, a handler did sync ClickHouse batch insert → event loop blocked → p99 latency 8s. Moved to `executeBlocking` → p99 dropped to 400ms.

**Q9. When would you deploy a verticle as a worker verticle? Give a real use case.**

**Answer**:  
Deploy as worker verticle (`DeploymentOptions().setWorker(true)`) when the verticle must perform **blocking or long-running operations** that cannot be made async.

Use cases:
- Legacy synchronous JDBC (non-reactive drivers)
- Heavy file I/O (large CSV parsing, reading images)
- CPU-intensive work (image resizing, encryption, complex regex on big text)
- Calling third-party synchronous APIs
- Legacy Java libraries without async wrappers

**Real use case**:
In a Vert.x reporting service, we had a verticle that used Apache POI (sync library) to generate large Excel files from DB data.  
Deployed as worker verticle:
```kotlin
vertx.deployVerticle(ReportGeneratorVerticle(), DeploymentOptions().setWorker(true))
```
→ Event loops stayed free for HTTP requests, report generation ran safely in background.

**Q10. How do Kotlin coroutines integrate with Vert.x threading model? What is the recommended way?**

**Answer**:  
Kotlin coroutines integrate via the `vertx-kotlin-coroutines` module + `CoroutineVerticle`.

**Recommended way**:
```kotlin
class MyVerticle : CoroutineVerticle() {

    override suspend fun start() {
        val router = Router.router(vertx)

        router.get("/users/:id").coroutineHandler { ctx ->
            val id = ctx.pathParam("id").toLong()
            val user = fetchUserSuspend(id)           // suspend DB call
            ctx.response().end(Json.encode(user))
        }

        vertx.createHttpServer()
            .requestHandler(router)
            .listen(8080).await()
    }
}

// Extension helper (very common pattern)
fun Router.coroutineHandler(fn: suspend (RoutingContext) -> Unit) {
    route().handler { ctx ->
        vertx.launch(ctx.vertx().dispatcher()) {   // ← key: use Vert.x dispatcher
            try {
                fn(ctx)
            } catch (e: Exception) {
                ctx.fail(500, e)
            }
        }
    }
}
```

**Why this works**:
- `ctx.vertx().dispatcher()` ensures coroutine runs on the same event-loop thread → preserves Vert.x context (local data, tracing, MDC)
- `await()` on Futures (e.g. `listenAwait`, `executeAwait`) keeps code linear
- No blocking — coroutines suspend/resume on event loop

**Q11. What is vertx.executeBlocking? When should you use it instead of a worker verticle?**

**Answer**:  
`executeBlocking` runs a blocking lambda on the **worker thread pool** and returns a `Future` (or `await()` in coroutines).

Example:
```kotlin
suspend fun slowOperation(): String = vertx.executeBlockingAwait { promise ->
    // Blocking work
    Thread.sleep(2000)
    promise.complete("Done after slow work")
}
```

**When to use executeBlocking**:
- One-off or infrequent blocking call
- Don’t want to deploy/manage a separate verticle
- Simple fire-and-forget or request-reply

**When to use worker verticle instead**:
- Blocking operation is frequent/repeated (better resource isolation)
- You want to expose the blocking logic via Event Bus (decoupling)
- Need multiple instances or clustering

**Q12. In Vert.x, how do you achieve parallelism across multiple cores for CPU-bound work?**

**Answer**:  
Deploy **multiple instances** of the same verticle:

```kotlin
vertx.deployVerticle(MyCpuIntensiveVerticle(), DeploymentOptions().setInstances(8))
```

- Vert.x spreads instances across event loops (round-robin) → each on different thread/core
- Each instance has its own state → no shared mutable state needed

For pure CPU work inside handler:
- Use `vertx.executeBlocking` → worker pool
- Or deploy as worker verticle + multiple instances

In practice: set `instances = Runtime.getRuntime().availableProcessors()` for CPU-bound verticles, but test — too many instances can increase context switching.

### Advanced Concurrency & Pitfalls

**Q13. What is a happens-before relationship in Java Memory Model? Give 2–3 examples.**

**Answer**:  
Happens-before is a partial ordering that guarantees one action is visible and ordered before another in concurrent code.

Examples:
1. **Unlock → Lock** same monitor:  
   All actions before unlock are visible after subsequent lock
2. **Volatile write → read**:  
   Write to volatile field → subsequent read sees it + all prior writes
3. **Thread.start() → actions in thread**:  
   All actions before start are visible in the started thread

We care because without happens-before, compiler/JVM can reorder or cache → stale data or impossible results.

**Q14. What is the difference between fair and unfair locks? Which one does ReentrantLock use by default?**

**Answer**:  
- **Fair lock**: Grants lock to the thread that has been waiting longest (FIFO queue) → no starvation, but slower (queue management)
- **Unfair lock**: Grants lock to whichever thread tries first (usually the one that just released) → higher throughput, but long-waiting threads can starve

`ReentrantLock` is **unfair by default** (faster in high contention).

Fair version:
```java
ReentrantLock fairLock = new ReentrantLock(true);
```

In modern code, I rarely use raw locks — prefer coroutines `Mutex`, `ConcurrentHashMap`, or Vert.x Event Bus.

**Q15. Explain ThreadLocal. When is it useful? What are the pitfalls?**

**Answer**:  
`ThreadLocal<T>` gives each thread its own independent copy of a variable — like a per-thread global.

Useful for:
- Per-thread logging context (MDC in Logback/SLF4J)
- Per-thread caching (e.g., SimpleDateFormat instance — thread-unsafe)
- Per-thread transaction state in legacy code

Pitfalls:
- **Memory leaks** — if not removed (`ThreadLocal.remove()`) in thread pools → threads keep old values
- **Wrong assumption** — developers think it’s global, but it’s per-thread
- **Hard to test** — requires mocking thread context

In Vert.x → avoid `ThreadLocal` — use `Context.localData()` or `Vertx.currentContext()` for per-request data.

**Q16. What is a race condition? Give an example and how to fix it.**

**Answer**:  
A **race condition** occurs when the outcome of code depends on the unpredictable timing or order of thread execution.

Example — non-atomic counter:
```java
int count = 0;

void increment() {
    count++;  // read → increment → write — three steps
}
```
- Thread A reads 5 → Thread B reads 5 → both write 6 → count = 6 instead of 7

Fixes:
- `synchronized` block
- `AtomicInteger.incrementAndGet()`
- In Kotlin coroutines: `Mutex` or single verticle instance
- In Vert.x: Event Bus + one verticle handling updates

**Q17. How do you safely share mutable state between different Vert.x verticles?**

**Answer**:  
Never share mutable objects directly — breaks the single-threaded guarantee.

Safe patterns:
1. **Event Bus** — publish immutable messages (data classes + Json.encode)
   ```kotlin
   vertx.eventBus().publish("cache.update", JsonObject().put("key", "user123").put("value", "active"))
   ```

2. **SharedData** (local or clustered maps/lists)
   ```kotlin
   val shared = vertx.sharedData().getLocalAsyncMap<String, String>("cache")
   shared.put("key", "value")
   ```

3. **Atomic classes** — `AtomicReference`, `AtomicInteger`
   ```kotlin
   val counter = AtomicInteger(0)
   counter.incrementAndGet()
   ```

4. **Immutable + copy-on-write** — use `toImmutableList()`, `toImmutableMap()`

Preferred: Event Bus + immutable data — simplest, safest, and scales with clustering.

**Q18. What is the difference between wait()/notify() and Condition from Lock?**

**Answer**:  
Both allow threads to wait and signal, but:

**wait()/notify()**:
- Tied to `synchronized` monitor
- Only one condition (the monitor itself)
- `notify()` wakes one thread, `notifyAll()` wakes all
- Not interruptible

**Condition** (from `ReentrantLock.newCondition()`):
- Multiple conditions per lock
- `await()` / `signal()` / `signalAll()`
- Interruptible (`awaitUninterruptibly`)
- Timed waits (`awaitNanos`)
- Better structured API

Modern preference: `Condition` or higher-level (queues, channels, coroutines Mutex).

**Q19. Explain the producer-consumer problem. How would you solve it in Kotlin coroutines?**

**Answer**:  
**Producer-consumer**: Producers add items to a shared buffer, consumers remove them. Need synchronization to avoid:
- Overflow (buffer full)
- Underflow (consume when empty)

Classic solution: bounded queue + locks/conditions.

In Kotlin coroutines → use `Channel` (safe, backpressure-aware):

```kotlin
val channel = Channel<Item>(capacity = 100)  // bounded

// Producer
launch {
    while (isActive) {
        val item = produceItem()
        channel.send(item)  // suspends if full
    }
}

// Consumer
launch {
    for (item in channel) {  // suspends if empty
        process(item)
    }
}
```

**Benefits**:
- Built-in backpressure — producer suspends when buffer full
- Structured concurrency — cancel scope → closes channel
- Safe exception propagation

Used this pattern in Tiki for event-driven ingestion — producers sent batches, consumers processed without locks.

**Q20. In a Vert.x + Kotlin service, how do you implement graceful shutdown so that in-flight requests finish?**

**Answer**:  
Use verticle `stop()` + track in-flight work + coroutines.

Pattern:
```kotlin
class MyVerticle : CoroutineVerticle() {

    private val inFlight = AtomicInteger(0)

    override suspend fun start() {
        router.get("/long").coroutineHandler { ctx ->
            inFlight.incrementAndGet()
            try {
                delay(5000)  // simulate long work
                ctx.response().end("Done")
            } finally {
                inFlight.decrementAndGet()
            }
        }
    }

    override suspend fun stop() {
        logger.info("Graceful shutdown - waiting for ${inFlight.get()} in-flight requests")

        withTimeoutOrNull(30_000) {  // max 30s
            while (inFlight.get() > 0) {
                delay(100)
            }
        }

        logger.info("All in-flight completed or timed out")
        // Close DB pools, Kafka consumers, etc.
        pgPool.close().await()
    }
}
```

**Why it works**:
- Vert.x calls `stop()` during undeploy/shutdown
- `inFlight` tracks active requests
- `withTimeoutOrNull` gives time to finish but prevents hang
- Works with clustering (each node shuts down independently)

In production, we added health-check endpoint that returns 503 when shutting down — load balancer stops sending new traffic.

These answers are now more detailed with code where it adds value.





### 1. Global Lock Ordering — Always acquire locks in the same order

**Why it prevents deadlock**: If all threads always lock resources in a fixed global order (e.g., by resource ID or name), it's impossible to create a circular wait.

**Example**: Transferring money between two accounts (classic deadlock risk if order is not enforced).

**Bad code (can deadlock)**:

Kotlin

```
suspend fun transferBad(from: Account, to: Account, amount: Long) {
    from.lock.withLock {
        to.lock.withLock {  // different order depending on call
            from.balance -= amount
            to.balance += amount
        }
    }
}
```

**Good code (global ordering by account ID)**:

Kotlin

```
suspend fun transfer(from: Account, to: Account, amount: Long) {
    // Always lock lower ID first → global order
    val (first, second) = if (from.id < to.id) from to else to to from

    first.lock.withLock {
        second.lock.withLock {
            first.balance -= amount
            second.balance += amount
        }
    }
}
```

**Vert.x version** (using Event Bus instead — even better):

Kotlin

```
// No locks needed — single verticle owns all accounts
class AccountVerticle : CoroutineVerticle() {
    private val balances = mutableMapOf<Long, Long>()

    override suspend fun start() {
        vertx.eventBus().consumer<JsonObject>("account.transfer") { msg ->
            val fromId = msg.body().getLong("fromId")!!
            val toId = msg.body().getLong("toId")!!
            val amount = msg.body().getLong("amount")!!

            // Single-threaded → safe without locks
            val fromBalance = balances.getOrDefault(fromId, 0L)
            if (fromBalance >= amount) {
                balances[fromId] = fromBalance - amount
                balances[toId] = balances.getOrDefault(toId, 0L) + amount
                msg.reply("success")
            } else {
                msg.reply("insufficient")
            }
        }
    }
}
```

### 2. Timeout Locks — use tryLock(timeout) instead of lock()

**Why it prevents deadlock**: If a thread can't acquire a lock within a timeout, it gives up → breaks potential cycles.

**Example**: Acquiring locks for order processing with timeout.

Kotlin

```
import java.util.concurrent.TimeUnit
import java.util.concurrent.locks.ReentrantLock

class OrderProcessor {
    private val inventoryLock = ReentrantLock()
    private val paymentLock = ReentrantLock()

    suspend fun processOrder(orderId: String): Boolean {
        // Try to get inventory lock with 5-second timeout
        if (!inventoryLock.tryLock(5, TimeUnit.SECONDS)) {
            logger.warn("Failed to acquire inventory lock for order $orderId")
            return false
        }

        try {
            // Now try payment lock with timeout
            if (!paymentLock.tryLock(3, TimeUnit.SECONDS)) {
                logger.warn("Failed to acquire payment lock for order $orderId")
                return false
            }

            try {
                // Critical section
                reserveInventory(orderId)
                processPayment(orderId)
                return true
            } finally {
                paymentLock.unlock()
            }
        } finally {
            inventoryLock.unlock()
        }
    }
}
```

**With coroutines Mutex** (preferred in Kotlin):

Kotlin

```
import kotlinx.coroutines.sync.Mutex
import kotlinx.coroutines.withTimeoutOrNull

class OrderProcessor {
    private val inventoryMutex = Mutex()
    private val paymentMutex = Mutex()

    suspend fun processOrder(orderId: String): Boolean {
        if (!withTimeoutOrNull(5_000) { inventoryMutex.lock() }) {
            logger.warn("Inventory lock timeout")
            return false
        }

        try {
            if (!withTimeoutOrNull(3_000) { paymentMutex.lock() }) {
                logger.warn("Payment lock timeout")
                return false
            }

            try {
                reserveInventory(orderId)
                processPayment(orderId)
                return true
            } finally {
                paymentMutex.unlock()
            }
        } finally {
            inventoryMutex.unlock()
        }
    }
}
```

### 3. Avoid Nested Locking When Possible — Refactor Code

**Why it helps**: Nested locks increase deadlock risk and contention.

**Bad (nested locking)**:

Kotlin

```
suspend fun updateUserAndOrder(userId: Long, orderId: Long) {
    userLock.withLock {
        orderLock.withLock {  // nested → deadlock risk
            updateUserBalance(userId)
            updateOrderStatus(orderId)
        }
    }
}
```

**Good (refactored — no nesting)**:

Kotlin

```
suspend fun updateUserAndOrder(userId: Long, orderId: Long) {
    // Step 1: Update user first (no lock overlap)
    userLock.withLock {
        updateUserBalance(userId)
    }

    // Step 2: Update order separately
    orderLock.withLock {
        updateOrderStatus(orderId)
    }
}
```

**Even better (Vert.x style — no locks)**:

Kotlin

```
// Single verticle owns both user and order state
class UserOrderVerticle : CoroutineVerticle() {
    private val userBalances = mutableMapOf<Long, Long>()
    private val orderStatuses = mutableMapOf<Long, String>()

    override suspend fun start() {
        vertx.eventBus().consumer<JsonObject>("update.userAndOrder") { msg ->
            val userId = msg.body().getLong("userId")!!
            val orderId = msg.body().getLong("orderId")!!

            // No locks — single-threaded
            val balance = userBalances.getOrDefault(userId, 0L)
            if (balance >= 100) {
                userBalances[userId] = balance - 100
                orderStatuses[orderId] = "PAID"
                msg.reply("success")
            } else {
                msg.reply("insufficient")
            }
        }
    }
}
```

### 4. Use Higher-Level Primitives — ConcurrentHashMap, BlockingQueue, coroutines Mutex, Vert.x Event Bus

**Examples**:

**ConcurrentHashMap** (thread-safe map, no explicit locks):

Kotlin

```
val cache = ConcurrentHashMap<String, String>()

// Safe concurrent access
cache.compute("key") { _, oldValue ->
    oldValue?.plus(" updated") ?: "new value"
}
```

**BlockingQueue** (producer-consumer without manual locks):

Kotlin

```
val queue = LinkedBlockingQueue<String>(100)

// Producer
launch {
    queue.put("task-1")  // blocks if full
}

// Consumer
launch {
    while (isActive) {
        val task = queue.take()  // blocks if empty
        process(task)
    }
}
```

**Kotlin Mutex** (coroutine-friendly lock):

Kotlin

```
val mutex = Mutex()
val sharedList = mutableListOf<String>()

suspend fun addItem(item: String) {
    mutex.withLock {
        sharedList.add(item)
    }
}
```

**Vert.x Event Bus** (no locks, single verticle):

Kotlin

```
// Single verticle owns shared state
class SharedStateVerticle : CoroutineVerticle() {
    private val state = mutableMapOf<String, Int>()

    override suspend fun start() {
        vertx.eventBus().consumer<JsonObject>("state.update") { msg ->
            val key = msg.body().getString("key")!!
            val delta = msg.body().getInteger("delta")!!
            state.compute(key) { _, old -> (old ?: 0) + delta }
            msg.reply(state[key])
        }
    }
}
```

**When to choose which**:

- ConcurrentHashMap → concurrent key-value store
- BlockingQueue → producer-consumer queue
- Mutex → coroutine-specific locking
- **Event Bus + single verticle** → preferred in Vert.x (no locks, decoupling, clustering-ready)

These are the exact patterns used in production to eliminate deadlock risk while keeping code safe and performant.