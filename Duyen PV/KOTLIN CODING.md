### 1. Simple HTTP server + coroutine handler + PostgreSQL (PgPool)

Kotlin

```
import io.vertx.core.json.JsonObject
import io.vertx.kotlin.coroutines.CoroutineVerticle
import io.vertx.kotlin.coroutines.await
import io.vertx.kotlin.coroutines.dispatcher
import io.vertx.pgclient.PgBuilder
import io.vertx.pgclient.PgConnectOptions
import io.vertx.pgclient.PgPool
import io.vertx.sqlclient.PoolOptions
import io.vertx.sqlclient.Tuple
import io.vertx.ext.web.Router
import io.vertx.ext.web.RoutingContext
import kotlinx.coroutines.launch

class HttpDbVerticle : CoroutineVerticle() {

  private lateinit var pgPool: PgPool

  override suspend fun start() {
    // 1. Initialize connection pool
    val connectOptions = PgConnectOptions()
      .setPort(5432)
      .setHost("localhost")
      .setDatabase("mydb")
      .setUser("postgres")
      .setPassword("secret")

    val poolOptions = PoolOptions().setMaxSize(20)

    pgPool = PgBuilder
      .pool(poolOptions)
      .using(vertx)
      .connectTo(connectOptions)

    // 2. HTTP server
    val router = Router.router(vertx)

    router.get("/users/:id").coroutineHandler(this::getUser)

    vertx.createHttpServer()
      .requestHandler(router)
      .listen(8080)
      .await()

    println("HTTP server started on port 8080")
  }

  // Coroutine handler (clean & idiomatic)
  private suspend fun getUser(ctx: RoutingContext) {
    val id = ctx.pathParam("id").toIntOrNull() ?: run {
      ctx.response().setStatusCode(400).end("Invalid ID")
      return
    }

    try {
      val row = pgPool
        .preparedQuery("SELECT id, name, email FROM users WHERE id = $1")
        .executeAwait(Tuple.of(id))

      if (row.size() == 0) {
        ctx.response().setStatusCode(404).end("User not found")
        return
      }

      val user = row.iterator().next()
      val json = JsonObject()
        .put("id", user.getInteger("id"))
        .put("name", user.getString("name"))
        .put("email", user.getString("email"))

      ctx.response()
        .putHeader("Content-Type", "application/json")
        .end(json.encode())
    } catch (e: Exception) {
      ctx.response().setStatusCode(500).end("Database error: ${e.message}")
    }
  }

  override suspend fun stop() {
    pgPool.close().await()
    println("PgPool closed")
  }
}

// Extension function – very common idiom
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

**What interviewers look for here**:

- Using CoroutineVerticle
- suspend handlers
- await() on async operations
- Proper resource cleanup in stop()
- Error handling
- Clean separation of concerns

### 2. Event Bus publish / subscribe with typed messages (medium difficulty)

Kotlin

```
import io.vertx.core.json.JsonObject
import io.vertx.kotlin.coroutines.CoroutineVerticle
import io.vertx.kotlin.coroutines.await
import kotlinx.coroutines.launch
import kotlinx.serialization.Serializable
import kotlinx.serialization.json.Json
import kotlinx.serialization.encodeToString

@Serializable
data class OrderEvent(
  val orderId: String,
  val userId: Long,
  val amount: Double,
  val createdAt: String
)

class OrderProcessorVerticle : CoroutineVerticle() {

  override suspend fun start() {
    // Consumer – processes incoming events
    vertx.eventBus().consumer<String>("order.created") { message ->
      launch {
        try {
          val json = message.body()
          val event = Json.decodeFromString<OrderEvent>(json)

          println("Processing order ${event.orderId} for user ${event.userId} – amount: ${event.amount}")

          // Simulate business logic (e.g. check fraud, send email, update inventory)
          delay(800) // pretend work

          // Reply (request-reply pattern)
          message.reply(JsonObject().put("status", "processed").put("orderId", event.orderId))
        } catch (e: Exception) {
          message.fail(500, e.message)
        }
      }
    }

    println("OrderProcessorVerticle started – listening on order.created")
  }
}

class OrderPublisherVerticle : CoroutineVerticle() {

  override suspend fun start() {
    // Simulate incoming HTTP request that creates an order
    vertx.setPeriodic(7000) {
      launch {
        val event = OrderEvent(
          orderId = "ORD-${System.currentTimeMillis()}",
          userId = 12345L,
          amount = 149.99,
          createdAt = Instant.now().toString()
        )

        val json = Json.encodeToString(event)

        try {
          val reply = vertx.eventBus()
            .requestAwait<String>("order.created", json)

          println("Order processed successfully: ${reply.body()}")
        } catch (e: Exception) {
          println("Order processing failed: ${e.message}")
        }
      }
    }

    println("OrderPublisherVerticle started – publishing every ~7s")
  }
}
```

**Interview points**:

- Typed messages (Kotlin data class + kotlinx.serialization)
- Request-reply pattern with requestAwait
- Error propagation via fail
- Decoupling via event bus

### 3. Rate limiting with Redis (more advanced)

Kotlin

```
import io.vertx.kotlin.coroutines.await
import io.vertx.redis.client.Redis
import io.vertx.redis.client.RedisAPI
import io.vertx.redis.client.RedisOptions
import io.vertx.ext.web.RoutingContext
import java.time.Instant

class RateLimitHandler(private val redisAPI: RedisAPI) {

  suspend fun handle(ctx: RoutingContext) {
    val userId = ctx.user()?.principal()?.getString("sub") ?: "anonymous"
    val key = "rate:ip:${ctx.request().remoteAddress().host()}:user:$userId"

    try {
      // Lua script would be better, but simple increment for demo
      val current = redisAPI.incr(key).await().toLong()
      if (current == 1L) {
        redisAPI.expire(key, 60).await() // 60 seconds window
      }

      if (current > 30) { // 30 req / minute
        ctx.response()
          .setStatusCode(429)
          .putHeader("Retry-After", "60")
          .end("Rate limit exceeded")
        return
      }

      // Add remaining requests header (nice touch)
      val ttl = redisAPI.ttl(key).await().toLong()
      ctx.response().putHeader("X-RateLimit-Remaining", (30 - current).toString())
      ctx.response().putHeader("X-RateLimit-Reset", (Instant.now().epochSecond + ttl).toString())

      ctx.next()
    } catch (e: Exception) {
      ctx.fail(500)
    }
  }
}

// In start():
val redis = Redis.createClient(vertx, RedisOptions().setConnectionString("redis://localhost:6379"))
val redisAPI = RedisAPI.api(redis)

// Then in router:
router.route().handler { ctx ->
  launch {
    rateLimitHandler.handle(ctx)
  }
}
```

**Interview favorites**:

- Using Redis for distributed rate limiting
- Headers like X-RateLimit-* (industry standard)
- Graceful fallback when Redis is down
- Security (using user principal)

### 4. Quick checklist interviewers often probe

|Topic|One-liner answer / thing to mention|
|---|---|
|Blocking code in event loop|Never. Use executeBlocking, withContext(Dispatchers.IO) or offload to worker verticle|
|Verticle instances|DeploymentOptions().setInstances(n) – match CPU cores for CPU-bound, more for I/O|
|Worker verticle|For blocking operations (e.g. image processing, legacy JDBC)|
|Shared data|SharedData.getLocalMap or distributed map with clustering|
|Circuit breaker|CircuitBreaker.create(vertx) – very common in fintech|
|Metrics / observability|Micrometer + Prometheus endpoint /metrics|
|Graceful shutdown|Close pools, unsubscribe, reply to in-flight requests|
|Testing coroutines|runTest { }, TestCoroutineDispatcher, delay mocking|