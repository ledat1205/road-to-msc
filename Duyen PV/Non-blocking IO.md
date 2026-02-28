
### Core Idea – No Thread per Request

Traditional servers (Tomcat, old Java servlets):

- 1 incoming connection → 1 dedicated thread
- Thread blocks during I/O → need thousands of threads for thousands of connections

Vert.x (event-driven + non-blocking):

- A small number of event-loop threads **multiplex** thousands of connections
- Threads **never block** on I/O — they use **epoll/kqueue** to wait for many sockets at once
- When data arrives on any socket, the event loop calls your handler for **that one connection** → then immediately moves to the next ready one

### How 10,000 Concurrent Connections Are Handled by ~16 Event-Loop Threads

Assume:

- 8-core machine → Vert.x creates **16 event-loop threads** by default (2 × cores)
- All connections are keep-alive (HTTP/1.1 or HTTP/2) — typical in modern APIs

Step-by-step life of 10,000 concurrent connections:

1. **Connections arrive**
    - Netty boss thread accepts TCP connections (very fast)
    - Each new socket is registered with one of the 16 event-loop threads (round-robin)
2. **Idle connections wait**
    - All 10,000 sockets are registered with epoll/kqueue
    - Each event-loop thread calls epoll_wait() → sleeps until **any** of its ~625 sockets (10,000 ÷ 16) has data
    - **Zero CPU usage** while waiting — kernel handles waiting
3. **Request arrives on one connection**
    - Network card receives packet → interrupt → kernel wakes the correct event-loop thread
    - epoll_wait() returns → “socket #7842 is readable”
    - Event-loop thread reads bytes → parses HTTP headers/body (non-blocking)
    - When full request is ready → calls **your handler** on **that same thread**
    - Handler runs quickly (non-blocking DB, cache lookup, etc.) → sends response (queues write)
    - Thread immediately goes back to epoll_wait() — no waiting
4. **Response is sent**
    - When socket becomes writable → epoll notifies again
    - Event-loop thread writes response chunks (non-blocking)
    - If write buffer full → thread queues rest and continues polling other sockets
5. **Many requests at the same time**
    - If 100 requests arrive at once → epoll returns a list of 100 ready sockets
    - One event-loop thread processes all 100 in a tight loop (microseconds each)
    - No thread switching — just sequential calls to handlers
    - Other 15 event loops do the same for their sockets

Result: **16 event-loop threads** can handle **10,000+ concurrent connections** because:

- Most connections are idle most of the time (keep-alive)
- Idle waiting is done by the kernel (epoll_wait blocks with 0% CPU)
- When work arrives, it’s **very short bursts** of CPU time per request
- One thread can process **hundreds of requests per second** if handlers are fast

### When the 20 Worker Threads Come In

Workers are only used when **you** explicitly offload blocking work.

Example scenarios:

- Sync JDBC query
- Large JSON parsing
- File read/write
- CPU-heavy computation

Flow:

- Your handler needs to do slow work:
    
    Kotlin
    
    ```
    router.get("/report").handler { ctx ->
        vertx.executeBlocking { promise ->
            // Blocking JDBC or file read here
            val report = generateSlowReport()
            promise.complete(report)
        } { ar ->
            if (ar.succeeded()) ctx.response().end(ar.result())
            else ctx.fail(500)
        }
    }
    ```
    
- Vert.x picks one of the **20 worker threads** → runs the blocking code
- Event-loop thread is free immediately → continues handling other requests
- When worker finishes → result is sent back to event loop → handler continues

So the **20 workers** handle occasional blocking tasks while the **16 event loops** keep the event-driven world spinning fast.
- “What does event-driven mean to you?”
    
- “How is Vert.x different from a traditional thread-pool server like Tomcat?”
    
- “Explain how you would build a price-drop notification system in Vert.x.”
    
- “What is the role of the Event Bus in Vert.x?”
    
- “Why is non-blocking important in high-throughput systems?”

### 1. “What does event-driven mean to you?”

**Answer**:

To me, event-driven means the system is **built around reacting to things that happen**, rather than following a fixed sequence of steps or constantly checking if something is ready.

Instead of code saying: "Do this → wait → do that → wait → do the next thing…"

The code says: "When X happens (an event), run this handler. When Y happens, run that handler."

Events can be anything:

- A new HTTP request arrives
- A timer fires
- A message arrives from Kafka
- A database row changes (via CDC)
- A price drops below a threshold (like in a price tracker)

The big advantages are:

- **High concurrency** — one thread can wait for thousands of events without wasting CPU
- **Loose coupling** — different parts of the system don't call each other directly; they just emit and react to events
- **Resilience** — if one handler is slow, it doesn't block everything else

In practice, I see this every day in Vert.x: the event loop waits for network I/O or timers using epoll under the hood, then calls my handler only when something is actually ready. At MoMo, our high-traffic mini-apps were event-driven — when a user completed a game round, we published an event, and separate services reacted to update leaderboards or send rewards without blocking the main flow.

### 2. “How is Vert.x different from a traditional thread-pool server like Tomcat?”

**Answer**:

The biggest difference is the **threading model and how they handle requests**.

In a traditional thread-pool server like Tomcat (classic servlet container):

- Each incoming HTTP request gets its own thread from a pool (e.g., 200 threads max by default)
- The thread blocks while doing work: reading the request, calling your servlet, querying the DB, writing the response
- If 200 requests come in at once and each takes 1 second (e.g., slow DB call), the server can only handle 200 concurrent requests — more requests queue up or get rejected
- High thread count → high memory (each thread has ~1 MB stack), lots of context switching, higher latency under load

Vert.x is **event-driven and non-blocking**:

- It uses a small number of event-loop threads (default 2 × CPU cores)
- One event-loop thread can handle **thousands** of requests because it **never blocks** — it registers interest in sockets/timers with epoll, then sleeps until the OS wakes it with ready events
- When a request arrives or data is ready, the event loop calls your handler quickly, then moves on to the next ready event
- Blocking work (e.g., sync DB call) is offloaded to a worker pool or executeBlocking — event loops stay fast
- Result: one server can handle 10k–100k+ concurrent connections with very low CPU/memory

In short: Tomcat scales by adding threads (thread-per-request). Vert.x scales by **avoiding blocking** and reusing few threads efficiently.

At Tiki, we switched some high-throughput endpoints to Vert.x — same hardware handled 5–10× more RPS because we eliminated blocking DB calls on the main loop.

### 3. “Explain how you would build a price-drop notification system in Vert.x.”

**Answer**:

I’d build it as an **event-driven microservice** using Vert.x + Kotlin + coroutines for clean async code.

High-level design:

1. **Input**: Users add products they want to track (keyword or URL) via HTTP API
2. **Periodic checking**: A scheduler periodically searches Google Shopping (or Shopee/Lazada APIs) for the lowest price
3. **Price drop detection**: Compare new price with previous lowest → if drop > threshold (e.g., 5%), publish event
4. **Notification**: Other services react to the event (email, push, in-app alert)

Implementation in Vert.x:

- **Verticle 1: HTTP API + Storage** (normal verticle)
    - POST /track → store keyword + user ID + threshold in PostgreSQL (using vertx-pg-client)
    - GET /tracked → list user’s items
- **Verticle 2: Price Checker** (CoroutineVerticle)
    - setPeriodic(30 minutes) → query DB for all tracked keywords
    - For each keyword: use WebClient (Vert.x HTTP client) to search Google Shopping
    - Parse lowest price (Jsoup or regex on HTML)
    - If new_low < previous_low:
        
        Kotlin
        
        ```
        vertx.eventBus().publish("price.dropped", JsonObject()
            .put("keyword", keyword)
            .put("newPrice", newPrice)
            .put("oldPrice", oldPrice)
            .put("userIds", userIds)
        )
        ```
        
- **Verticle 3: Notification Sender** (worker verticle if email is blocking)
    - Subscribe to Event Bus address "price.dropped"
    - Send email/push via async client (e.g., Vert.x MailClient or Firebase SDK)
    - Update DB with new lowest price

Why Vert.x fits perfectly:

- Event Bus decouples checker from notifier — easy to scale or add more notifiers
- Non-blocking WebClient for Google searches → no event-loop stalls
- Coroutines make async parsing/HTTP/DB calls look synchronous

This way, even if email sending is slow, it doesn't block price checking or API.

### 4. “What is the role of the Event Bus in Vert.x?”

**Answer**:

The **Event Bus** is the **central nervous system** of Vert.x — it's a lightweight, asynchronous message bus that lets verticles (or different parts of your app) communicate without knowing about each other.

Main roles:

1. **Decoupling**: Verticles don’t call each other directly — one publishes a message, others subscribe and react. Example: "OrderPlaced" → inventory verticle decreases stock, notification verticle sends email.
2. **Pub/Sub & Request-Reply**:
    - publish() → fire-and-forget to all subscribers
    - send() → point-to-point (round-robin if multiple consumers)
    - request() → request-reply pattern (like RPC over async)
3. **Clustering**: If you enable clustering (Hazelcast or Ignite), Event Bus messages automatically go across JVMs/servers — great for distributed systems.
4. **Thread safety**: All delivery happens on event-loop threads — no locks needed if you follow Vert.x rules.

Real example from MoMo: When a user completed a minigame round, we published "GameRoundCompleted" on Event Bus → leaderboard verticle updated scores, reward verticle credited coins, analytics verticle logged event — all decoupled, easy to add new features.

In short: Event Bus turns your app into a reactive, loosely-coupled system — core to Vert.x’s power.

### 5. “Why is non-blocking important in high-throughput systems?”

**Answer**:

Non-blocking is crucial in high-throughput systems because it lets you **handle many more concurrent requests with fewer resources**.

In blocking systems (like classic Tomcat/JDBC):

- Each request ties up one thread while waiting (DB query, network call, file read)
- If each request waits 100 ms for DB, one thread can handle only ~10 requests per second
- 1,000 concurrent requests → need 1,000 threads → high memory (~1 MB stack per thread), lots of context switching, expensive scaling

In non-blocking systems (Vert.x, Netty, reactive drivers):

- Threads **never wait** — they register interest ("tell me when DB data is ready") and move on
- One event-loop thread can handle **thousands** of requests because it only works when real data arrives
- Waiting is done by the OS kernel (epoll_wait) → zero CPU while idle
- Result: same server handles 10–100× more throughput with 10–20× less memory/CPU
### 1. The Event Loop — Core Idea (Simplified View)

At its simplest:

text

```
while (true) {
    // Step 1: Ask the OS: "Which sockets/files/timers are ready right now?"
    ready_events = poll_for_ready_io_and_timers();   // this call BLOCKS until something is ready

    // Step 2: For each ready event, call the registered callback/handler
    for (event in ready_events) {
        handler = lookup_handler_for(event);
        handler(event);   // invoke user code
    }
}
```

This loop **never** spins (busy-waits) — it **blocks** in the OS kernel until at least one thing is ready (I/O readable, timer expired, signal received, etc.). When something becomes ready, the kernel wakes up the thread → the loop processes all ready events → goes back to sleep.

→ One thread can wait for **thousands** of things at once → massive concurrency with low CPU usage.

### 2. The Three Main Polling Mechanisms (Under the Hood)

The OS provides different system calls for this "wait for many things efficiently". Vert.x (via Netty) chooses the best one depending on the OS.

| Mechanism  | OS Support              | How it works                                                                                                   | Performance / Scalability                    | Used by Vert.x / Netty when?        |
| ---------- | ----------------------- | -------------------------------------------------------------------------------------------------------------- | -------------------------------------------- | ----------------------------------- |
| **select** | Almost all Unix/Windows | Passes array of file descriptors (sockets) to kernel. Kernel returns which are ready.                          | O(n) — slow when you have >1000 descriptors  | Fallback on very old systems        |
| **poll**   | Most Unix               | Like select, but better API (no 1024 limit). Still O(n) scan.                                                  | Better than select, still slow at high scale | Rarely used today                   |
| **epoll**  | Linux (since 2.5.44)    | Kernel maintains a ready list. You register interest once → kernel notifies when ready. O(1) for ready events. | Excellent — scales to 100k+ connections      | Default on Linux (best performance) |
| **kqueue** | BSD/macOS               | Similar to epoll: kernel-managed queue, register once, get notified.                                           | Excellent — same class as epoll              | Default on macOS/FreeBSD            |
| **IOCP**   | Windows                 | Completion ports — kernel queues completed I/O operations. Thread pool pulls from queue.                       | Very good — designed for high concurrency    | Default on Windows                  |

**Most important thing**: All of them allow **one thread to wait on thousands of file descriptors** (sockets, pipes, timers, etc.) **without polling every single one every time**. epoll/kqueue are the kings — they only return the **ready** subset, not the whole list.

### 3. Step-by-Step: What Happens When You Register a Handler

Let's take the **HTTP request** scenario as example (but same logic applies to timers, Kafka polling, etc.).

1. You write:
    
    Kotlin
    
    ```
    server.requestHandler { req -> println("Request from ${req.remoteAddress()}") }
    ```
    
2. Vert.x / Netty does (simplified):
    - Creates a Netty Channel for the TCP socket.
    - Registers the socket with the event loop's selector (epoll/kqueue).
    - Attaches your handler as a callback in the ChannelPipeline (Netty's handler chain).
3. Event loop thread runs its main loop:
    
    C
    
    ```
    // Pseudo-code of Netty/Vert.x event loop
    while (running) {
        // Ask kernel: give me ready events (epoll_wait / kevent / select)
        int n = epoll_wait(epfd, events, max_events, timeout);
    
        for (i = 0; i < n; i++) {
            if (events[i] is readable) {
                // Read bytes from socket
                read_from_socket(channel);
    
                // If full HTTP request parsed → fire user handler
                channel.pipeline().fireChannelRead(decodedRequest);
            }
            if (events[i] is writable) {
                // Flush queued writes
                flush_pending_writes(channel);
            }
        }
    
        // Also check timers
        process_expired_timers();
    }
    ```
    
4. When fireChannelRead reaches your handler → your code runs **on the same event-loop thread**.
5. If your handler blocks → the **entire event loop** blocks → bad. → Vert.x detects this (>2 seconds) and logs: Blocked thread: vertx-eventloop-thread-3 → You fix by offloading to executeBlocking / worker / virtual thread.

### 4. Why This Model Scales So Well

- **One thread waits for everything** (epoll_wait blocks until ANY socket/timer is ready).
- **Zero polling overhead** — kernel tells you exactly what’s ready (O(1) for ready events).
- **No thread-per-connection** — 1–2 threads per core can handle 10,000–100,000+ connections.
- **Predictable latency** — as long as you don’t block the event loop.

### Comparison with Traditional Blocking Servers

Traditional (Tomcat, old Java servlets):

- One thread per request → accepts connection → spawns thread → thread blocks on read/write/DB.
- 10,000 concurrent requests → need ~10,000 threads → massive context switching, memory (~1 MB per thread stack).

Vert.x / event-driven:

- One event loop thread waits on epoll for all 10,000 sockets.
- When data arrives → handler runs quickly → response queued → loop continues.
- Only a few threads total → very low overhead.

### Quick Summary Table (for Interviews)

| Scenario              | What the OS/kernel does                            | What Vert.x/Netty does                  | Thread that runs your code  |
| --------------------- | -------------------------------------------------- | --------------------------------------- | --------------------------- |
| HTTP request arrives  | Socket becomes readable → epoll notifies           | Reads bytes, parses HTTP, fires handler | Event-loop thread           |
| Timer fires           | Timerfd expires → epoll notifies                   | Checks timer queue, invokes callback    | Event-loop thread           |
| Kafka message arrives | Kafka poll() returns messages (blocking on worker) | Delivers to handler                     | Event-loop thread (handler) |
| DB change (CDC)       | Debezium/Kafka produces message → same as Kafka    | Same as Kafka                           | Event-loop thread (handler) |


#### 1. HTTP Request Finished

**Trigger**: An incoming HTTP request arrives on a socket, or a response is ready to send.

**Technical Flow**:

- **Netty's role**: Vert.x uses Netty's EventLoopGroup (thread pool of event loops). When a client connects (TCP accept event), Netty's boss thread accepts the socket and hands it to a worker event loop.
- **Event loop polling**: The event loop uses OS-level selectors (epoll on Linux) to check for readable sockets. When data is ready (e.g., HTTP request headers/body), it fires a read event.
- **Vert.x handling**: Vert.x wraps Netty channels in HttpServerRequest. The event loop invokes your registered handler (non-blocking). When the request is "finished" (full body read, parsed), Vert.x publishes an internal event or directly calls your handler.
- **Completion**: If you call ctx.response().end(), Netty queues the write on the event loop — it uses write buffers to avoid blocking. Completion is signaled via Future/Handler when write drains.
- **Threads**: Always on the same event-loop thread for the connection (single-threaded guarantee per verticle).

**Code Example** (Kotlin + Vert.x Web):

Kotlin

```
// Setup: event-driven HTTP server
vertx.createHttpServer().requestHandler { req ->
    req.bodyHandler { body ->  // Event: body finished reading
        val json = body.toJsonObject()
        req.response().end(JsonObject().put("processed", json).encode())  // Event: response finished
    }
}.listen(8080) { ar ->
    if (ar.succeeded()) println("Server up")
}
```

Behind the scenes: Netty's ChannelPipeline processes bytes → Vert.x decodes HTTP → fires bodyHandler when full request is ready.

#### 2. Timer Fired

**Trigger**: A scheduled timeout or periodic task expires.

**Technical Flow**:

- **Timer registration**: When you call vertx.setTimer(delay) { handler } or setPeriodic(interval) { handler }, Vert.x adds the timer to a priority queue (min-heap based on expiry time) per event loop.
- **Event loop integration**: Each event loop has a timer queue. During its main loop (while true), after polling for I/O (epoll_wait), it checks the timer queue for expired timers (current time > expiry).
- **Firing**: If expired, the event loop removes it from the queue and invokes the handler directly on the same thread. For periodic, it reschedules.
- **Precision**: Not millisecond-accurate (depends on loop load) — for high-precision, use external schedulers like Quartz.
- **Threads**: Always on the event-loop thread that registered the timer. If handler blocks, it stalls the loop — use executeBlocking inside.

**Code Example** (Periodic timer):

Kotlin

```
vertx.setPeriodic(5000) { id ->  // Event: timer fired every 5s
    launch {  // Coroutine for async work
        val data = fetchDataSuspend()  // Non-blocking
        println("Processed data: $data")
    }
}
```

Behind the scenes: Vert.x's TimerImpl is queued in EventLoopContext.timerQueue (a PriorityQueue). The event loop calls checkTimers() after I/O poll, firing if System.currentTimeMillis() >= timer.expiry.

#### 3. Message Arrived from Kafka

**Trigger**: A new message is available in a Kafka topic/partition.

**Technical Flow**:

- **Vert.x Kafka Client**: Uses Kafka's consumer API under the hood, but wrapped in async handlers. You configure a KafkaConsumer with vertx-kafka-client.
- **Polling loop**: Vert.x runs a background poller (on worker or event loop, configurable) that calls Kafka's poll(timeout) periodically. When messages arrive, it fires the registered handler.
- **Event handling**: The poller delivers messages to your handler on the event-loop thread (non-blocking). Handler processes and commits offsets async (commitAwait() in coroutines).
- **Backpressure**: Vert.x pauses/resumes polling if handler is slow (flow control).
- **Threads**: Polling can be on worker (blocking Kafka poll), but handlers on event loop. Use pause()/resume() to avoid overload.

**Code Example** (Kotlin + Vert.x Kafka):

Kotlin

```
val consumer = KafkaConsumer.create<String, JsonObject>(vertx, config)

consumer.handler { record ->  // Event: message arrived
    launch {  // Coroutine on event loop
        val msg = record.value()
        processMessage(msg)
        consumer.commitAwait()  // Async commit
    }
}

consumer.subscribe("my-topic") { ar ->
    if (ar.succeeded()) println("Subscribed")
}
```

Behind the scenes: Vert.x's KafkaConsumerImpl uses a VertxTimer to periodically call Kafka's poll() (blocking call offloaded to worker). When messages are polled, they are queued and delivered to handler via event loop.

#### 4. DB Change via CDC (Change Data Capture)

**Trigger**: A row insert/update/delete in a DB (e.g., PostgreSQL, MySQL) generates a change event.

**Technical Flow**:

- **CDC Tool**: Use Debezium (embedded or standalone) to capture changes from DB logs (e.g., WAL in PostgreSQL). Debezium publishes changes as Kafka messages (JSON/Avro).
- **Integration with Vert.x**: Vert.x consumes from the Kafka topic created by Debezium (same as Kafka message scenario above). When a message arrives, it's an event: "row changed".
- **Event handling**: Your handler processes the change (e.g., update cache, trigger recompute). Commits offset to Kafka.
- **End-to-end**: DB → Debezium polls logs → Kafka message → Vert.x consumer handler fires.
- **Threads**: Kafka poller on worker (blocking), handler on event loop.

**Code Example** (Vert.x Kafka consumer for CDC events):

Kotlin

```
consumer.handler { record ->  // Event: DB change message arrived via Kafka
    launch {
        val change = record.value().getJsonObject("after")  // Debezium format
        val id = change.getLong("id")
        val newStatus = change.getString("status")
        updateCache(id, newStatus)  // React to change
    }
}
```

Behind the scenes: Debezium uses Kafka Connect to stream binlog/WAL changes as events (key: row ID, value: before/after snapshot). Vert.x polls Kafka (as above) — the "change" is just a Kafka message event.