- “What does event-driven mean to you?”
    
- “How is Vert.x different from a traditional thread-pool server like Tomcat?”
    
- “Explain how you would build a price-drop notification system in Vert.x.”
    
- “What is the role of the Event Bus in Vert.x?”
    
- “Why is non-blocking important in high-throughput systems?”


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