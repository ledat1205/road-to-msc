
### 0. Approach to System Design Interviews (Start Here Every Time)

1. **Clarify requirements** — Ask about functional (e.g., post tweet, shorten URL) and non-functional (scale: DAU/QPS, latency, availability, consistency, cost) needs. Probe edge cases (e.g., peak traffic, offline support).
2. **Estimate scale** — Use back-of-the-envelope calculations (your section 2) to size QPS, storage, bandwidth.
3. **High-level design** — Draw major components (client → load balancer → app servers → cache → DB → queue → storage).
4. **Deep dive** — Discuss bottlenecks, trade-offs, APIs, data models, scaling strategies.
5. **Address non-functionals** — Availability, reliability, security, monitoring.
6. **Iterate** — Handle follow-ups (e.g., "What if traffic 10x?").

**Tip**: Use RESHADED framework (Requirements → Estimation → Storage → High-level → API → Detailed → Evaluate → Distinctive). Think aloud, balance trade-offs, and draw diagrams.

### 1. Latency and Throughput Basics

Your numbers are spot-on (aligned with updated Jeff Dean-style lists).

**Elaboration**:

- Every hop adds latency—aim for <200–300 ms end-to-end for user-facing (feels instant); <100 ms p99 for APIs.
- Network is often the biggest bottleneck for global systems.
- Throughput limited by slowest component (e.g., DB write vs. network).

**Examples**:

- **URL Shortener**: Redirect latency goal <100 ms → use CDN + in-memory cache (Redis) for 99% hit rate.
- **Twitter/X**: Timeline read <200 ms → fanout-on-write for celebrities (pre-compute), cache timelines in Redis.
- **Robotics**: Control loop <10 ms → edge computing (process sensor data on-device) vs. cloud (150 ms cross-continent round-trip unacceptable for real-time path planning).
- **Instagram**: Photo feed <300 ms → CDN for media, Memcached for metadata.

**Network Speeds**:

- Real-world: 1 Gbps VM → ~100 MB/s after overhead.
- Example: Netflix streams 4K (~25 Mbps) → needs edge caching to avoid backbone saturation.

### 2. Scalability and Sizing Estimates

**Elaboration**:

- Always start with QPS/storage estimates to justify choices (sharding, replication).
- Assume 3x replication for durability, 2–3x peak for headroom.
- Peak factor: 2–5x average QPS.

**Examples**:

- **Twitter/X** — 500M tweets/day = ~6k avg QPS, peak ~30–50k. With 300M DAU × 200 queries/day → ~700k QPS peak → needs heavy sharding + caching.
- **URL Shortener** — 100M new URLs/month (~40 QPS avg), 1B reads/month (~12k QPS) → 100–200 servers + Redis cache.
- **Robotics fleet** — 10k robots × 100 sensor readings/sec = 1M writes/sec → use time-series DB (Cassandra) + Kafka for buffering.
- Storage: Instagram — 1B photos/year × 5 MB = ~5 PB/year → S3-like blob store + metadata in NoSQL.

**Sharding/Replication**:

- Shard by user_id (consistent hashing) to avoid hotspots.
- Replication factor 3: quorum write (2/3), read (2/3) for availability.

### 3. Database and Storage Choices

**Elaboration**:

- Choose based on read/write ratio, consistency needs, data model.
- SQL for strong consistency/transactions; NoSQL for scale/availability.

**Examples**:

- **User accounts/inventory** — PostgreSQL/MySQL (ACID, joins).
- **Twitter/X timeline** — Hybrid: Cassandra (wide-column for time-series tweets) + Redis (hot timelines).
- **Robotics sensor data** — Cassandra/InfluxDB (high-write time-series, 100k+ writes/sec).
- **Instagram feeds** — Redis (pre-computed), Cassandra (storage), DynamoDB (metadata).
- **URL Shortener** — Key-value (Redis/DynamoDB) for short-to-long mapping + bloom filter for existence checks.
- Trade-off: SQL vertical scale → NoSQL horizontal + eventual consistency.

### 4. Caching and Optimization

**Elaboration**:

- Cache hot data (80/20 rule) → 90–99% hit rate reduces DB load 10–100x.
- Invalidation: write-through (consistent but slow), cache-aside (lazy, stale risk).

**Examples**:

- **Twitter/X** — Cache user timelines, profiles in Redis → sub-ms reads.
- **Netflix** — CDN + edge cache for videos → <50 ms global latency.
- **Robotics** — Redis for real-time robot state (location, battery) → <1 ms access.
- Indexing: B-tree speeds reads 10–100x but slows writes → avoid over-indexing.

### 5. Messaging and Queues

**Elaboration**:

- Decouple for bursts, async processing, durability.
- At-least-once + idempotency for retries.

**Examples**:

- **Uber** — Ride requests → Kafka for high-throughput pub-sub.
- **Amazon order fulfillment** — SQS/RabbitMQ for decoupling payment → shipping.
- **Robotics** — ROS topics (pub-sub) or Kafka for command queues → handle bursts without blocking control loop.
- Kafka: 1M+ msgs/sec with partitions for parallelism.

### 6. Availability, Reliability, and Monitoring

**Elaboration**:

- 99.99% = 4.3 min downtime/month → critical for robotics control.
- Use circuit breakers, retries with backoff, health checks.

**Examples**:

- **Netflix** — Chaos engineering + multi-region failover.
- **Robotics** — Redundant edge nodes, quorum voting for control decisions.
- Monitoring: Prometheus + Grafana → alert on p99 >200 ms or error >0.1%.

### 7. Security and Edge Cases

**Elaboration**:

- Stateless JWT/OAuth, rate limiting (token bucket), encryption.

**Examples**:

- **Instagram** — OAuth for login, rate limit uploads.
- **Robotics** — Secure ROS topics, rate limit commands to prevent overload.

### 8. Common Patterns (Add These to Your Toolbox)

- **Fanout-on-write** — Pre-compute feeds (Twitter celebrities).
- **Fanout-on-read** — Lazy load (inactive users).
- **CQRS** — Separate read/write models for scale.
- **Event sourcing** — Store events, replay state (audit trail).
- **Consistent hashing** — Minimize reshuffling on shard changes.
- **Bloom filter** — Space-efficient existence checks (URL shortener).
- **CDN** — Static/media delivery.

### 9. Common System Design Questions (Quick Key Points)

- **URL Shortener** — Base62 encoding, DB for mapping, cache hot URLs, handle collisions, analytics.
- **Twitter/X** — Fanout, timeline cache, sharding by user, eventual consistency for likes/retweets.
- **Instagram** — Feed generation (hybrid fanout), media in S3 + CDN, ranking ML.
- **Uber** — Geohashing/quadtree for nearby drivers, WebSocket for real-time, Kafka for events.
- **Robotics fleet** — Edge processing, time-series DB, pub-sub for commands, low-latency priority.

### 10. Modern Considerations (2026+)

- Microservices + service mesh (Istio) for observability.
- Serverless (Lambda) for bursty workloads.
- AI integration: agentic systems, real-time ML inference.
- Global scale: multi-region, geo-partitioning.

**Pitfalls to Avoid** — Skipping requirements, no trade-offs, overengineering, ignoring monitoring/security.
