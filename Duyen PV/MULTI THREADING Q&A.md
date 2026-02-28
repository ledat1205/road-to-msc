
### Java/Kotlin Threading Basics (Q1–Q6)

**Q1. What is the difference between a process and a thread? Why do we prefer threads over processes in most backend applications?**

**Answer**: A process has its own memory space, address space, and resources (heap, stack, file descriptors). Threads share the same memory space and resources within a process, but each has its own stack and registers.

We prefer threads because:

- Lower creation overhead (no separate address space)
- Faster context switching (shared memory)
- Easier data sharing (no IPC needed)

In backend services (e.g., Vert.x apps), we almost always use threads within one JVM process — it allows high concurrency with low resource usage. Processes are used only for isolation (microservices) or fault tolerance.

**Q2. Explain what happens when two threads access the same non-volatile field without synchronization.**

**Answer**: Without synchronization, you get data races → undefined behavior in Java. Possible outcomes:

- Stale reads (one thread sees old value)
- Torn reads (partial updates)
- Reordered operations (compiler/JVM can reorder instructions)

Example: one thread writes flag = true, another reads flag — reader might never see true even after writer finishes.

Solution: volatile, synchronized, Atomic*, Lock, or higher-level constructs like coroutines with actors.

**Q3. What does the volatile keyword guarantee in Java/Kotlin? When is it enough? When is it not?**

**Answer**: volatile guarantees:

1. Visibility: writes are immediately visible to other threads
2. Happens-before relationship: write to volatile → subsequent reads see it
3. No reordering around volatile accesses

It is **enough** for simple flags (e.g., shutdown flag) or single-writer-many-readers patterns.

It is **not enough** when you need atomic compound actions (read-check-write), like counters — use AtomicInteger instead.

In Kotlin: same semantics as Java (@Volatile annotation for fields).

**Q4. Explain the difference between synchronized method/block and ReentrantLock. When would you choose one over the other?**

**Answer**: synchronized (monitor lock):

- Built-in, simple syntax
- Automatically releases on exception
- Can only lock on object reference

ReentrantLock:

- More features: tryLock(), lockInterruptibly(), fair/unfair, conditions
- Explicit lock/unlock (must use try-finally)
- Can have timeout, interruptible acquisition

Choose:

- synchronized for simple cases (short blocks, no need for advanced features)
- ReentrantLock when you need fairness, interruptibility, or conditions (e.g., producer-consumer with bounded queue)

In modern code I prefer higher-level constructs (Mutex in coroutines, ConcurrentHashMap) over raw locks.

**Q5. What is a deadlock? How can you prevent or detect it in production?**

**Answer**: Deadlock: two or more threads waiting for each other to release locks in a circular way.

Classic example: Thread A holds lock1, waits for lock2; Thread B holds lock2, waits for lock1.

Prevention:

1. Always acquire locks in the same order
2. Use timeout locks (tryLock)
3. Avoid nested locking when possible
4. Use higher-level concurrency primitives (queues, actors)

Detection in production:

- Thread dumps (jstack, VisualVM) — look for “deadlock detected” or circular wait chains
- Monitoring tools (Datadog, New Relic, Prometheus + jmx-exporter) expose blocked threads

In Vert.x we rarely see deadlocks because most code is non-blocking or uses Event Bus instead of shared locks.

**Q6. What is the Java Memory Model (JMM)? Why do we care about it in concurrent code?**

**Answer**: JMM defines how and when changes made by one thread are visible to others — it specifies visibility, atomicity, and ordering guarantees.

Key rules:

- volatile → write → read happens-before
- synchronized → exit → enter happens-before
- Thread start/join, final fields, etc.

We care because without proper synchronization, compilers/JVM can reorder instructions, cache values, leading to surprising bugs (stale reads, lost updates).

In Kotlin + Vert.x, we mostly avoid raw JMM issues by using coroutines (structured, dispatcher-bound) or atomic classes.

### Vert.x-Specific Threading & Concurrency (Q7–12)

**Q7. Explain Vert.x's threading model in detail. What is the single-threaded execution guarantee?**

**Answer**: Vert.x uses:

- Event-loop threads (default 2 × CPU cores) — handle non-blocking I/O, timers, handlers
- Worker threads (default 20) — for blocking work

Key guarantee: **Each standard verticle instance is executed by exactly one event-loop thread at a time**. No two threads call methods on the same verticle instance concurrently. This is Vert.x’s “single-threaded execution model” — similar to Node.js event loop, but multi-threaded across cores.

Result: you can write non-thread-safe code inside a verticle (no synchronized needed for internal state), as long as you don’t share mutable state between verticles without Event Bus or locks.

**Q8. What happens if you perform a blocking operation (e.g., Thread.sleep, sync JDBC) inside a normal Vert.x verticle?**

**Answer**: The event-loop thread that owns the verticle gets blocked. All other handlers/timers/network events assigned to that same event loop are delayed or starved → latency spikes, timeouts, degraded throughput across unrelated requests.

Vert.x detects and logs warnings: Blocked thread warning after 2 seconds (configurable).

Fix:

- vertx.executeBlocking { ... }
- Move to worker verticle
- Use async/reactive client (e.g., vertx-pg-client)

**Q9. When would you deploy a verticle as a worker verticle? Give a real use case.**

**Answer**: Deploy as worker (DeploymentOptions().setWorker(true)) when the verticle performs **blocking or long-running operations** that cannot be made async.

Use cases:

- Legacy synchronous JDBC drivers
- File system heavy I/O (reading large files, CSV parsing)
- CPU-intensive tasks (image resizing, encryption)
- Calling third-party sync APIs

Real example: In a Vert.x service I worked on, we had a legacy report generator that used sync POI library to create Excel files. We deployed that verticle as worker — kept event loops free for HTTP requests.

**Q10. How do Kotlin coroutines integrate with Vert.x threading model? What is the recommended way?**

**Answer**: Use CoroutineVerticle + vertx-kotlin-coroutines module.

Best practice:

Kotlin

```
class MyVerticle : CoroutineVerticle() {
    override suspend fun start() {
        val router = Router.router(vertx)
        router.get("/").coroutineHandler { ctx ->
            val data = fetchDataSuspend()
            ctx.response().end(data)
        }
    }
}
```

Key points:

- Use ctx.vertx().dispatcher() to launch coroutines → stays on correct event loop
- All await() calls are non-blocking
- Preserves Vert.x context (local data, tracing)

This combines Vert.x’s event-loop guarantees with Kotlin’s linear async code.

**Q11. What is vertx.executeBlocking? When should you use it instead of a worker verticle?**

**Answer**: executeBlocking runs a blocking task on the worker thread pool and returns a Future — callback or await when done.

Kotlin

```
vertx.executeBlocking<String>({ promise ->
    val result = slowSyncCall()
    promise.complete(result)
}, { ar ->
    if (ar.succeeded()) ctx.response().end(ar.result())
})
```

Use executeBlocking when:

- Blocking work is one-off or infrequent
- You don’t want a separate verticle

Use worker verticle when:

- Blocking operation is frequent/repeated
- You want Event Bus interface to it
- Better resource isolation

**Q12. In Vert.x, how do you achieve parallelism across multiple cores for CPU-bound work?**

**Answer**: Deploy multiple instances of the same verticle:

Kotlin

```
vertx.deployVerticle(MyCpuVerticle(), DeploymentOptions().setInstances(8))
```

Vert.x distributes instances across event loops (round-robin). Each instance runs on its own event loop thread → uses different cores.

For pure CPU work, combine with executeBlocking or worker verticles + multiple instances.

In practice: set instances ≈ number of cores, but test under load — too many can increase context switching.

### Advanced Concurrency & Pitfalls (Q13–20)

**Q13. What is a happens-before relationship in Java Memory Model? Give 2–3 examples.**

**Answer**: Happens-before is the partial ordering that guarantees visibility and ordering of actions between threads.

Examples:

1. Unlock of monitor → subsequent lock of same monitor
2. Write to volatile → subsequent read of same volatile
3. Thread.start() → actions in started thread
4. All actions in thread → Thread.join() return

We care because it tells us when changes are safely visible — without it, threads can see stale data even after writes.

**Q14. What is the difference between fair and unfair locks? Which one does ReentrantLock use by default?**

**Answer**: Fair lock: grants access to the longest-waiting thread first (FIFO). Unfair lock: grants to any waiting thread (usually the one that just tried to acquire) — better throughput but can starve threads.

ReentrantLock is **unfair by default** (faster in high-contention).

Pass true to constructor for fair: ReentrantLock(true).

In Vert.x we rarely use raw locks — prefer Event Bus or coroutines.

**Q15. Explain ThreadLocal. When is it useful? What are the pitfalls?**

**Answer**: ThreadLocal<T> gives each thread its own independent copy of a variable.

Useful for:

- Per-thread configuration (MDC logging context)
- Per-thread caching
- Transaction context in legacy code

Pitfalls:

- Memory leaks if not removed (remove()) in thread pools
- Wrong assumption that it’s global (it’s per-thread)
- Hard to test

In Vert.x: avoid ThreadLocal — use Context.localData() or MDC with Vert.x context.

**Q16. What is a race condition? Give an example and how to fix it.**

**Answer**: Race condition: outcome depends on unpredictable timing/order of threads.

Classic example: incrementing counter without synchronization

Java

```
int count = 0;
void increment() { count++; }  // not atomic
```

Fixes:

- synchronized block
- AtomicInteger.incrementAndGet()
- Mutex in coroutines
- Vert.x: Event Bus + single verticle instance

**Q17. How do you safely share mutable state between different Vert.x verticles?**

**Answer**: Never share mutable objects directly — breaks single-thread guarantee.

Safe ways:

1. Event Bus (send immutable messages)
2. SharedData (local/async maps/lists) – distributed with clustering
3. Atomic classes (AtomicReference, AtomicInteger)
4. Immutable objects + copy-on-write

Preferred: Event Bus + immutable data classes.

**Q18. What is the difference between wait()/notify() and Condition from Lock?**

**Answer**: wait()/notify(): tied to synchronized monitor, can only notify one/all, no interruptible wait.

Condition (from ReentrantLock.newCondition()):

- Multiple conditions per lock
- Interruptible (awaitUninterruptibly)
- Timed waits
- Better structured API

Modern code prefers Condition or higher-level primitives (queues, semaphores).

**Q19. Explain the producer-consumer problem. How would you solve it in Kotlin coroutines?**

**Answer**: Producer-consumer: producers add items to a buffer, consumers take them. Need synchronization to avoid overflow/underflow.

In coroutines: use Channel (like blocking queue)

Kotlin

```
val channel = Channel<Item>(capacity = 100)

launch { // producer
    while (isActive) {
        channel.send(produceItem())
    }
}

launch { // consumer
    for (item in channel) {
        process(item)
    }
}
```

Channel handles backpressure, cancellation, exceptions automatically.

**Q20. In a Vert.x + Kotlin service, how do you implement graceful shutdown so that in-flight requests finish?**

**Answer**: Use verticle stop() + coroutines + Vert.x lifecycle.

Pattern:

Kotlin

```
class MyVerticle : CoroutineVerticle() {

    private val inFlight = AtomicInteger(0)

    override suspend fun start() {
        router.get("/long").coroutineHandler { ctx ->
            inFlight.incrementAndGet()
            try {
                delay(5000)
                ctx.response().end("Done")
            } finally {
                inFlight.decrementAndGet()
            }
        }
    }

    override suspend fun stop() {
        // Wait for in-flight to finish (timeout 30s)
        withTimeoutOrNull(30_000) {
            while (inFlight.get() > 0) {
                delay(100)
            }
        }
        // Close resources
    }
}
```

Vert.x undeploys verticles in reverse order — use stop() for cleanup.