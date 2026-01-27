## To learn by heart
### 1. Time & Scale Basics

- Seconds in a day: **86,400** (≈ 10^5) → use for QPS = (DAU × actions/day) / 86,400
- Seconds in a year: **31.5 million** (≈ 3×10^7)
- Peak factor: **2–5× average QPS** (common assumption: 3× for headroom)
- Queries per user per day (typical apps): **10–100** (Twitter ≈ 200, simple app ≈ 10)
- Peak QPS rule: **Average × 3–5** (Twitter 500M tweets/day → ~6k avg → 20–50k peak)

### 2. Latency Numbers (Memorize Order of Magnitude)

- L1 cache: **0.5–1 ns**
- RAM: **100 ns** (≈ 10^2 ns)
- SSD read: **10–100 μs** (≈ 10^5 ns = 0.1 ms)
- Network (same DC): **0.5–1 ms** (≈ 500 μs)
- Network (cross-region US): **50–100 ms**
- Network (cross-continent): **150–250 ms**
- Disk seek (HDD): **4–10 ms**
- HTTP request (simple): **1–10 ms** processing
- Goal p99 user-facing: **<100–200 ms**
- Real-time robotics/control loop: **<10 ms**

### 3. Throughput & Bandwidth

- 1 Gbps = **125 MB/s** theoretical (real ≈ **100 MB/s**)
- 10 Gbps = **1.25 GB/s** (real ≈ **1 GB/s**)
- Typical cloud VM bandwidth: **1–5 Gbps**
- Redis single-thread: **100k–200k ops/sec**
- Kafka partition: **10k–100k msgs/sec** per partition
- Cassandra write: **100k+ writes/sec** per node (wide-column)
- QPS per server (complex app with DB): **100–1,000**
- QPS per server (cache-hit/simple): **5k–20k+**

### 4. Data Size & Growth

- 1 KB ≈ tweet / short message
- 4 KB ≈ web page / article
- 10 KB ≈ email
- 1–5 MB ≈ HD photo / compressed image
- 50–200 MB ≈ 1 min 1080p video
- 1 GB = **10^9 bytes**
- 1 TB = **10^12 bytes**
- Replication factor: **3×** (standard for durability/HA)
- Data growth example: 1M users × 1 KB/day = **1 GB/day** → **~1 TB/year** (before replication/compression)

### 5. Storage & Sharding Rules

- Shard size target: **1–10 GB** per shard (Cassandra/Mongo)
- Max nodes per cluster (practical): **~1,000** (coordination overhead)
- Cache hit rate goal: **90–99%**
- Redis cluster size: **10–100 GB** typical

### 6. Uptime & Reliability

- 99% = **3.65 days/year** downtime (≈ 7.3 h/month)
- 99.9% = **8.76 hours/year** (≈ 43 min/month)
- 99.99% = **52.6 min/year** (≈ 4.3 min/month) — critical systems target
- 99.999% (“five 9s”) = **5.26 min/year**

### 7. Quick Calculation Formulas

- QPS from DAU: **QPS ≈ (DAU × actions/day) / 86,400**
    - Example: 10M DAU × 50 actions/day = 500M / 86,400 ≈ **5,800 avg QPS** → peak **15–30k**
- Storage/year: **users × bytes/user/day × 365 × replication**
    - Example: 1M users × 10 KB/day × 365 ≈ **3.65 TB/year** raw → **~11 TB** with 3× replication
- Bandwidth for reads: **QPS × response size**
    - Example: 10k QPS × 10 KB response = **100 MB/s** → needs ~1 Gbps link

### 8. Fast Rules of Thumb

- “10× rule”: Every level slower than previous (L1 → RAM → SSD → network → disk)
- Cache reduces load **10–100×** if hit rate high
- Sharding: divide by **user_id hash** or **time** (time-series)
- Replication: **write to 2/3**, **read from 2/3** for quorum
- Assume **1–2% request failure** → need circuit breakers/retries


## Guides
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
