- “What does event-driven mean to you?”
    
- “How is Vert.x different from a traditional thread-pool server like Tomcat?”
    
- “Explain how you would build a price-drop notification system in Vert.x.”
    
- “What is the role of the Event Bus in Vert.x?”
    
- “Why is non-blocking important in high-throughput systems?”


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