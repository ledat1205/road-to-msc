
#### 1. **Latency and Throughput Basics**

- **Memory Hierarchy Latencies** (critical for estimating bottlenecks):
    - L1 cache reference: ~1 ns
    - RAM access: ~100 ns
    - SSD read: ~100 μs (1,000x slower than RAM)
    - HDD seek: ~10 ms (100x slower than SSD)
    - Network round-trip (same data center): ~500 μs
    - Cross-data center (e.g., US-East to US-West): ~50–100 ms
    - Cross-continent (e.g., US to Europe): ~150 ms
- **Network Speeds**:
    - 1 Gbps = 125 MB/s theoretical max (real-world ~100 MB/s)
    - 10 Gbps = 1.25 GB/s
    - Typical cloud bandwidth: 1–5 Gbps per VM instance
- **Common Throughput**:
    - HTTP request: ~1–10 ms processing time on a single server
    - API p99 latency goal: <100 ms for user-facing, <1 s for batch
    - QPS per server: 100–1k for complex apps (e.g., web server with DB calls); 10k+ for simple (e.g., static cache hits)

#### 2. **Scalability and Sizing Estimates**

- **User Scale Calculations** (back-of-the-envelope):
    - Daily Active Users (DAU) to QPS: QPS = (DAU × queries per user per day) / (86,400 seconds). E.g., 1M DAU × 10 queries/day = ~115 QPS peak (assume 2x for peak).
    - Data growth: 1M users generating 1 KB/day/user = ~1 TB/year (before compression/replication).
    - Storage: 1 TB = 10^12 bytes; plan for 3x replication factor in distributed systems.
- **Storage Sizes**:
    - Text: 1 page ~4 KB; tweet ~1 KB; email ~10 KB; HD image ~5 MB; 1 min video ~100 MB.
    - Logs: 1 server generates ~1–10 GB/day; robotics telemetry (sensors): 1–100 GB/hour per device.
- **Sharding/Replication**:
    - Shards: Aim for 1–10 GB per shard in DBs like Cassandra/Mongo for manageability.
    - Replication factor: 3 (common for high availability); quorum reads/writes (e.g., write to 2/3, read from 2/3).
    - Horizontal scaling limit: ~1k nodes/cluster before coordination overhead kills performance.

#### 3. **Database and Storage Choices (Building on Your Table)**

- **When to Choose What** (based on data type/requirements):
    - **Relational (SQL)**: Structured data, transactions, joins. Use for user accounts, inventories. Avoid for >10k QPS without sharding. ACID compliant.
    - **Key-Value (NoSQL)**: Unstructured/simple lookups (e.g., sensor IDs to values in robotics). Redis for caching (sub-ms gets), DynamoDB for managed.
    - **Document (NoSQL)**: Semi-structured (JSON-like), flexible schemas. Mongo for configs/logs; good for robotics metadata.
    - **Wide-Column**: Time-series/high writes (e.g., Cassandra for IoT/sensor streams in robotics—handles 100k+ writes/sec).
    - **Graph**: Relationships (e.g., Neo4j for robotics path-planning/social graphs). Queries: 1–10 ms for small graphs.
    - **Time-Series**: InfluxDB/Prometheus for metrics (e.g., robot vitals). Ingest: 100k–1M points/sec.
    - **OLAP**: Analytics on big data (e.g., ClickHouse for robot fleet logs). Avoid for OLTP.
- **Key Trade-offs**:
    - SQL: Strong consistency, but scales vertically (add CPU/RAM).
    - NoSQL: Eventual consistency, scales horizontally.
    - CAP: In partitions, SQL picks CP (consistent but unavailable); NoSQL often AP (available but inconsistent).
- **Benchmarks to Recall**:
    - MySQL: Max connections ~10k; innodb buffer pool ~75% of RAM.
    - Redis: 100k+ ops/sec single-threaded; use pipelines for batch.
    - Kafka (queues): 1M+ msgs/sec throughput; partitions for parallelism.

#### 4. **Caching and Optimization**

- **Cache Hit Rates**: Aim for 90–99%; eviction policies: LRU (least recently used) common.
- **Redis/Memcached**: ~1 ms latency; 10–100 GB typical size; use for hot data (e.g., robot state in real-time systems).
- **CDN**: Reduces latency by 50–90% for static assets; e.g., CloudFront hits <50 ms globally.
- **Indexing**: Speeds reads 10–100x but slows writes 2–5x; B-tree default for SQL.

#### 5. **Messaging and Queues**

- **When to Use**: Decouple services, handle bursts (e.g., robot commands queue).
    - RabbitMQ: Reliable delivery, ~10k msgs/sec.
    - Kafka: High-throughput pub-sub, 100k–1M msgs/sec; partitions = parallelism (e.g., 1k partitions for scale).
    - SQS: Managed, ~ unlimited throughput but ~10 ms latency.
- **Patterns**: At-least-once vs. exactly-once delivery; idempotency for retries.

#### 6. **Availability, Reliability, and Monitoring**

- **Uptime SLAs**:
    - 99% = 7.3 hours downtime/month
    - 99.9% = 43 min/month
    - 99.99% = 4.3 min/month (goal for critical systems like robotics control)
- **Failure Rates**: Assume 1% of requests fail; use circuit breakers (Hystrix-like) to prevent cascades.
- **Load Balancing**: Round-robin default; health checks every 10–30s.
- **Monitoring**: Prometheus for metrics; p50/p99 latencies; error rates <0.1%.

#### 7. **Security and Edge Cases**

- **Auth**: JWT tokens (stateless); OAuth for third-party.
- **Rate Limiting**: 100–1k reqs/user/min; token bucket algorithm.
- **Data Privacy**: GDPR/CCPA—anon data where possible.
- **Robotics-Specific (Since Mentioned)**: Real-time constraints (e.g., ROS for pub-sub, <10 ms latency); edge computing for low-latency (e.g., process sensor data on-device to avoid cloud round-trips).