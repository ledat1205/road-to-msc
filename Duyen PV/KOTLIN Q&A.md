### 1. Kotlin Language & Best Practices

**Q1: Explain the difference between val and var, and when you prefer one over the other in production code.**

**Answer**: val is immutable (read-only) — once assigned, it cannot change. var is mutable.

In production Kotlin code, I almost always prefer val unless I have a very good reason to mutate state. Immutability reduces bugs (no accidental changes), makes code thread-safe by default, and improves readability/intent.

For example, in a data class or function return value:

Kotlin

```
data class User(val id: Long, val name: String) // never var here
```

I only use var when state must change, like in a builder pattern, mutable collections during construction, or performance-critical mutable buffers (very rare). At MoMo, we enforced val-first style in code reviews — it caught many concurrency bugs early.

**Q2: How do you handle null safety in Kotlin when working with legacy Java libraries or JSON from external APIs?**

**Answer**: I use safe calls (?.), Elvis operator (?:), let, run, or requireNotNull/checkNotNull depending on context.

For JSON (e.g., from Retrofit/Gson/Jackson), I prefer:

- Custom deserializers or @JsonAdapter to map nulls to defaults
- Sealed class wrappers: sealed class ApiResult<out T> { data class Success<T>(val data: T) : ApiResult<T>(), data class Error(val message: String) : ApiResult<Nothing>() }
- Or NonNullJsonAdapter if using Moshi

Example from real code:

Kotlin

```
val userName = response.data?.user?.name ?: "Guest"
```

In legacy Java interop, I wrap dangerous calls in requireNotNull or use !! only in very controlled places with immediate checks. We had a rule: max 1 !! per 1000 lines — it forced us to handle nulls properly.

### 2. Coroutines & Concurrency

**Q3: What is structured concurrency and why is it important in Kotlin? Give an example.**

**Answer**: Structured concurrency means that coroutines have a clear parent-child hierarchy and lifecycle. When a scope is cancelled or fails, all children are cancelled automatically. No leaked coroutines.

Without it (e.g., globalScope.launch), coroutines can outlive their intended lifetime and cause memory leaks or zombie work.

Example (good):

Kotlin

```
suspend fun loadData() = coroutineScope {
    val user = async { fetchUser() }
    val orders = async { fetchOrders() }
    combine(user.await(), orders.await())
}
```

If the outer scope is cancelled, both asyncs are cancelled.

At Tiki, we used coroutineScope { } in streaming pipelines — it made cancellation reliable when jobs were interrupted.

**Q4: When would you use Dispatchers.IO vs Dispatchers.Default vs Dispatchers.Unconfined?**

**Answer**:

- Dispatchers.IO: For I/O-bound work (network, disk, DB calls). Limited parallelism (64 threads max by default, grows if needed). Prevents starving the CPU pool.
- Dispatchers.Default: For CPU-bound work (parsing, calculations). Uses thread pool sized to CPU cores.
- Dispatchers.Unconfined: Almost never in production — runs on caller thread, no thread switch. Useful only in tests or very specific cases (can break structured concurrency).

Rule of thumb: 90% of suspend functions use Dispatchers.IO or withContext(IO) { ... }. In Vert.x I usually use vertx.dispatcher() to stay on event loop when possible.

### 3. Vert.x Specific Questions (Middle Level)

**Q5: Explain Vert.x event-loop model. What happens if you block inside a normal verticle?**

**Answer**: Vert.x uses a small number of event-loop threads (default 2 × CPU cores). Each normal verticle instance is pinned to **one event-loop thread** — Vert.x guarantees no concurrent execution on the same instance.

All handlers, timers, Event Bus consumers for that verticle run on the same thread.

If you block (e.g., Thread.sleep, synchronous JDBC, heavy CPU loop), **that entire event loop stalls** — no other requests/timers on that loop are processed → high latency or timeouts across unrelated requests.

Fix:

- Use vertx.executeBlocking { ... }
- Deploy as worker verticle (setWorker(true))
- Or use virtual threads (Vert.x 4.5+)

In MoMo I once debugged a slow startup that blocked event loops — moved DB init to worker verticle and latency dropped 10×.

**Q6: How do you deploy multiple instances of the same verticle? When do you do it?**

**Answer**: Use DeploymentOptions().setInstances(n) when deploying:

Kotlin

```
vertx.deployVerticle(MyHttpVerticle(), DeploymentOptions().setInstances(4))
```

Vert.x distributes instances across event loops (round-robin).

You do this when:

- The verticle is CPU-bound (more instances = better core utilization)
- You want higher throughput for I/O-bound work
- Horizontal scaling inside one JVM before clustering

In production I usually set instances = Runtime.getRuntime().availableProcessors() for HTTP verticles — gives good balance without over-threading.

**Q7: How do you integrate Kotlin coroutines with Vert.x? Give a code example.**

**Answer**: Use CoroutineVerticle + suspend fun start() + await() extensions.

Example (common pattern):

Kotlin

```
class UserVerticle : CoroutineVerticle() {

    private lateinit var pgPool: PgPool

    override suspend fun start() {
        pgPool = PgBuilder.pool(...).build()

        val router = Router.router(vertx)
        router.get("/users/:id").coroutineHandler(this::getUser)

        vertx.createHttpServer()
            .requestHandler(router)
            .listen(8080).await()
    }

    private suspend fun getUser(ctx: RoutingContext) {
        val id = ctx.pathParam("id").toLong()
        val row = pgPool.preparedQuery("SELECT * FROM users WHERE id = $1")
            .executeAwait(Tuple.of(id))

        ctx.response().end(row.firstOrNull()?.toJson()?.encode() ?: "{}")
    }
}

// Extension helper (very common)
fun Router.coroutineHandler(fn: suspend (RoutingContext) -> Unit) {
    get("/").handler { ctx ->
        vertx.launch(ctx.vertx().dispatcher()) {
            try {
                fn(ctx)
            } catch (e: Exception) {
                ctx.fail(500, e)
            }
        }
    }
}
```

This keeps code linear, non-blocking, and uses Vert.x dispatcher for context propagation.