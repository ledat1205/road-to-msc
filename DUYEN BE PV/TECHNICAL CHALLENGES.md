### 1. Slow Code Due to Improper Vert.x Usage (Not Offloading Heavy Tasks)

To present this in your interview, frame it as a detective story of performance debugging in an event-driven system. Emphasize the iterative process of hypothesis testing, tool usage, and failed attempts before landing on the solution. This shows your methodical approach to troubleshooting distributed systems.

- **Initial Context and Symptoms:** Start by explaining the setup: "Our application was built on Vert.x for its reactive, non-blocking nature, handling event streams from user interactions. We used an outbox pattern for reliability—storing events in the DB before processing and marking them as 'sent' to avoid duplicates. However, during load tests, the system lagged: API response times spiked from 200ms to 2-5 seconds, and throughput dropped 50% at 1k requests/min. Logs showed sporadic timeouts, but no obvious errors."

- **Pinpointing the Error: Speculations and Testing:**
  - **Speculation 1: Network/DB Latency.** "First, I suspected external factors like DB query slowness. I profiled with Vert.x's built-in metrics (via Micrometer) and tools like pgBadger for Postgres queries. Queries were fast individually (<50ms), but under concurrency, contention arose from frequent idempotency checks (SELECT/UPDATE on unique keys). Tested by simulating load with JMeter—no major network spikes, so ruled out."
  - **Speculation 2: Code Inefficiencies.** "Next, thought it was algorithmic—maybe inefficient data processing loops. I used Java Flight Recorder (JFR) for CPU sampling; it showed hotspots in transformation logic, but optimizing them (e.g., switching to streams) only improved 10%. Still slow."
  - **Speculation 3: Vert.x Misconfiguration.** "Dug deeper into Vert.x docs and realized we might be blocking the event loop. Confirmed with Vert.x's blocked thread checker (via `VertxOptions` with warnings enabled)—logs flooded with 'Thread blocked for X ms' during heavy tasks. Root cause: Synchronous DB ops and computations in the main verticle weren't offloaded, starving the event loop for I/O handling."
  - **Other Tests:** "Ruled out JVM GC pauses by tuning heap sizes and using G1GC; monitored with VisualVM—no correlation with lags. Also checked cluster coordination; no issues there."

- **Trying Different Ways to Solve It:**
  - **Attempt 1: Optimize DB Usage.** "Tried indexing more columns and using read replicas for checks—helped marginally (15% faster), but still DB-bound under scale. Considered sharding the outbox table, but it added complexity without addressing the blocking."
  - **Attempt 2: Vert.x Worker Verticles.** "Offloaded heavy tasks to worker pools via `vertx.executeBlocking()`. This unblocked the loop, improving latency by 20%, but idempotency checks remained slow due to DB round-trips."
  - **Attempt 3: Caching Layer.** "Introduced local Caffeine cache for idempotency—fast for single-node, but failed in distributed setup (inconsistent across instances). Led to duplicates during failovers."
  - **Final Solution: Switch to Redis.** "Integrated Redis for idempotency (using hashes for event keys) and distributed locking (Redlock via Redisson). Tested with chaos engineering (killing nodes mid-process)—no duplicates, and checks dropped to <5ms. Combined with worker offloading, full fix achieved 40-60% latency reduction."
  
- **Outcomes and Reflections:** "We scaled to 2x load without issues. Lesson: In reactive frameworks, always validate non-blocking with tools like JFR; prefer in-memory stores for ephemeral ops to spare DB for durability."

This deep dive (aim for 4-5 minutes) highlights your debugging toolkit and resilience in testing hypotheses.

### 2. Scaling Down Druid and ClickHouse Nodes (Kafka Migration and Optimizations)

Present this as a cost-performance optimization journey, detailing the experimentation in migrations and tuning. Focus on data-driven iterations, showing how you balanced trade-offs in big data pipelines.

- **Initial Context and Symptoms:** "Our setup had Druid and ClickHouse ingesting from Kafka via dedicated microservices for real-time analytics. As data grew (10M events/day), we over-provisioned nodes (5+ each), costing $5k/month extra. Goal: Scale down to 2-3 nodes without SLA breaches (query latency <1s, ingestion lag <5min)."

- **Pinpointing the Error: Speculations and Testing:**
  - **Speculation 1: Over-Ingestion Overhead.** "Suspected intermediary services were inefficient—pulling, transforming, inserting redundantly. Monitored with Kafka's consumer lag metrics and Prometheus; lags hit 10-15min during peaks, with high CPU on services."
  - **Speculation 2: Query Inefficiencies.** "Thought it was bad queries in Druid/ClickHouse. Analyzed with EXPLAIN and Druid's segment metadata—queries were optimized, but ingestion bottlenecks caused stale data."
  - **Speculation 3: Format and Batching Issues.** "After migrating to Kafka engine tables (direct consumption), tests showed CPU spikes. Used ClickHouse's system.metrics to pinpoint JSON parsing as the culprit—deserializing verbose JSON ate 40% CPU. Batching was absent, leading to frequent small inserts."
  - **Other Tests:** "Ruled out network throughput with iperf tests; fine. Simulated failures with Chaos Mesh—no resilience issues, but performance degraded linearly with volume."

- **Trying Different Ways to Solve It:**
  - **Attempt 1: Tune Existing Services.** "Optimized services with better Kafka consumer configs (e.g., fetch.max.bytes up, session.timeout.ms down)—reduced lag by 20%, but still needed 4+ nodes. Too incremental."
  - **Attempt 2: Partial Migration to Kafka Engines.** "Switched ClickHouse to Kafka tables for direct pulls. Initial perf tests (using Kafka producers to simulate 10k eps) showed 30% faster ingestion, but lags persisted under burst loads. Druid similar."
  - **Attempt 3: Compression and Alternatives.** "Tried Avro format first—compact, but schema evolution issues in tests caused deserialization errors. Switched to Protobuf: Schema-strict but efficient. Without batching, CPU still high."
  - **Final Solution: Protobuf + Batching.** "Implemented batching (group 500 msgs via Kafka consumer batching) with Protobuf. Benchmarked with Locust: CPU down 50%, lag <2min. Monitored post-deploy with Grafana dashboards—stable at reduced nodes."
  
- **Outcomes and Reflections:** "Cut costs 40%, maintained performance. Key: Iterative testing with realistic loads; binary formats like Protobuf shine for scale, but require upfront schema work."

Aim for 5 minutes; use metrics to make it tangible.

### 3. PostgreSQL Batch Snapshots to BigQuery (Replication, Stability, and Degradation)

Frame this as a reliability engineering tale, diving into database internals and long-term maintenance. Highlight error reproduction, tool usage, and evolutionary fixes.

- **Initial Context and Symptoms:** "We replicated Postgres data (e-commerce transactions, 50M+ rows) to BigQuery for analytics via batch exports (pg_dump + GCS uploads) or CDC tools. Issues arose: Jobs failed with 'canceling statement due to conflict with recovery' errors (like StackOverflow discussions on WAL conflicts), plus connection exhaustion and slowing queries as data grew."

- **Pinpointing the Error: Speculations and Testing:**
  - **Speculation 1: WAL/Recovery Conflicts.** "Suspected hot standby replicas interfering with long snapshots. Reproduced by running pg_dump during simulated failovers—error triggered when recovery kicked in. Logs showed query cancellations to avoid stale reads."
  - **Speculation 2: Connection Overload.** "Thought pool starvation: Monitored with pg_stat_activity; during snapshots, active connections maxed out (default 100), causing waits and timeouts."
  - **Speculation 3: Table Bloat and Degradation.** "Over time, queries slowed (from 1min to 10min). Used pgstattuple extension—revealed high bloat from unvacuumed deletes/updates. Index fragmentation confirmed with pg_index stats."
  - **Other Tests:** "Ruled out BigQuery side with isolated uploads—fast. Tested query plans with EXPLAIN ANALYZE; sequential scans on large tables were the bottleneck."

- **Trying Different Ways to Solve It:**
  - **Attempt 1: Basic Retries and Tuning.** "Added retry logic in scripts (exponential backoff)—fixed transient errors but not root conflicts. Increased max_connections to 200—helped short-term, but risked OOM under load."
  - **Attempt 2: Switch to Logical Replication.** "Used pg_logical_slot for streaming changes (via Debezium to Kafka, then BigQuery). Reduced full dumps, but slot overflows occurred during high writes; tuned wal_keep_segments—better, but still occasional lags."
  - **Attempt 3: Pooling and Partitioning.** "Integrated pgBouncer for connection pooling—cut active connections 50%, stabilizing during snapshots. Tried range partitioning tables—faster queries, but migration downtime was risky; tested on staging first."
  - **Final Solution: Comprehensive Techniques.** "Combined HikariCP for app-side pooling, auto-vacuum tuning (scale_factor=0.05), and regular CLUSTER/REINDEX cron jobs. For degradation, added archiving (pg_partman for old partitions to S3). Monitored with Check_Postgres.pl alerts."
  
- **Outcomes and Reflections:** "99.9% success rate, halved times. Insight: DBs degrade subtly; use extensions like pgstattuple early; balance immediate fixes (retries) with structural ones (partitioning)."

This section (4 minutes) underscores proactive monitoring and layered solutions. 

**General Interview Tips:** Weave in tools (e.g., JFR, Prometheus) to show technical depth. Practice storytelling to keep it engaging—use "We tried X, but Y happened, so we pivoted to Z." Be prepared for questions like "What metrics defined success?" or "Any trade-offs?"